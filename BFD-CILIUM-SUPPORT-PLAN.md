# BFD Support in Cilium BGP Control Plane — Implementation Plan

Based on GitHub issue [#22394](https://github.com/cilium/cilium/issues/22394) and upstream GoBGP BFD support ([#3402](https://github.com/osrg/gobgp/pull/3402), released in v4.6.0).

## Background

### BFD Protocol

BFD (Bidirectional Forwarding Detection, [RFC 5880](https://www.rfc-editor.org/rfc/rfc5880)) is a lightweight health-checking protocol that detects failures in the data forwarding plane at sub-second intervals. It runs over UDP ([RFC 5881](https://www.rfc-editor.org/rfc/rfc5881) for single-hop) and is commonly paired with BGP to trigger fast route withdrawal when a peer becomes unreachable — much faster than BGP's built-in hold timers (which are typically 30-90s).

### Cilium BGP Control Plane Architecture

Cilium's BGP CP uses a layered architecture:

```
Kubernetes CRDs
  CiliumBGPClusterConfig  (cluster-wide desired state)
  CiliumBGPPeerConfig     (per-peer configuration template)
  CiliumBGPAdvertisement  (what to advertise)
        │
        ▼ [Cilium Operator renders per-node]
  CiliumBGPNodeConfig     (per-node instance of the cluster config)
        │
        ▼ [Agent Controller — pkg/bgp/agent/controller.go]
  BGPRouterManager.ReconcileInstances()
        │
        ▼ [ConfigReconcilers run in priority order]
  NeighborReconciler      (last, priority 110)
        │
        ├── getPeerConfig()        → CiliumBGPPeerConfigSpec
        ├── getPeerPassword()      → secret
        └── ToNeighborV2()         → types.Neighbor (vendor-agnostic internal type)
              │
              ▼ [GoBGPServer.AddNeighbor() — pkg/bgp/gobgp/peer.go]
        ToGoBGPPeer()              → gobgp.Peer (protobuf)
              │
              ▼ [server.AddPeer() — GoBGP's BgpServer]
        BGP session established
```

### Current status

- Cilium vendors `github.com/osrg/gobgp/v4 v4.6.1-0.20260630022313-d6dee8360046` (post-v4.6.0 commit from June 30, 2026).
- GoBGP `gobgp.Peer` protobuf message has a `Bfd *BfdPeerConfig` field (line 706 of `gobgp.proto`).
- GoBGP's `BgpServer` automatically creates and manages BFD sessions when `peer.Bfd` is populated (calls `bfdServer.AddPeer()` in `addNeighbor()` at `server.go:3480`).
- Cilium has **zero BFD references** in its BGP code (`pkg/bgp/`), CRD types (`pkg/k8s/apis/cilium.io/v2/`), or anywhere else.
- `ToGoBGPPeer()` never sets `newPeer.Bfd`, so GoBGP never starts BFD for any peer.

### Why this is straightforward

GoBGP already implements the hard parts:
- **BFD state machine** (RFC 5880: Down → Init → Up) in `pkg/server/bfd_peer.go`
- **UDP listener** on port 3784 in `bfd_server.go`
- **Timer management** — configurable tx/rx intervals and detection multiplier
- **BGP integration** — on BFD session failure, calls `ResetPeer(soft=false)` with reason "BFD is down"
- **gRPC API** — the `Peer` protobuf carries `BfdPeerConfig` (for config) and `PeerState` carries `BfdPeerState` (for state)

Cilium only needs to **pass the config through** — no protocol logic, no state machine, no timers.

---

## GoBGP BFD API — vendored details

### Protobuf types (in `api/gobgp.pb.go`, from `gobgp.proto`)

#### `BfdPeerConfig` — configuration
```go
type BfdPeerConfig struct {
    Enabled                  bool
    Port                     uint32     // default 3784
    DesiredMinimumTxInterval uint32     // µs, default 1,000,000 (1s)
    RequiredMinimumReceive   uint32     // µs, default 1,000,000 (1s)
    DetectionMultiplier      uint32     // default 3
}
```

Embedded in `Peer` struct (line 6966):
```go
Bfd *BfdPeerConfig `protobuf:"bytes,12,opt,name=bfd,proto3" json:"bfd,omitempty"`
```

#### `BfdPeerState` — operational state (returned in `PeerState`)
```go
type BfdPeerState struct {
    SessionState                  BfdSessionState     // UP/DOWN/INIT/ADMIN_DOWN
    RemoteSessionState            BfdSessionState
    LastFailureTime               uint64
    FailureTransitions            uint64
    LocalDiscriminator            uint32
    RemoteDiscriminator           uint32
    LocalDiagnosticCode           BfdDiagnosticCode
    RemoteDiagnosticCode          BfdDiagnosticCode
    RemoteMinimumReceiveInterval  uint32
    BfdAsync                      *BfdAsyncCounters
}
```

Embedded in `PeerState` struct (line 7996):
```go
BfdState *BfdPeerState `protobuf:"bytes,23,opt,name=bfd_state,json=bfdState,proto3" json:"bfd_state,omitempty"`
```

#### `BfdSessionState` enum
```
BFD_SESSION_STATE_UNSPECIFIED = 0
BFD_SESSION_STATE_UP          = 1
BFD_SESSION_STATE_DOWN        = 2
BFD_SESSION_STATE_ADMIN_DOWN  = 3
BFD_SESSION_STATE_INIT        = 4
```

#### `BfdDiagnosticCode` enum
```
BFD_DIAGNOSTIC_CODE_NO_DIAGNOSTIC                      = 0
BFD_DIAGNOSTIC_CODE_DETECTION_TIMEOUT                  = 1
BFD_DIAGNOSTIC_CODE_ECHO_FAILED                        = 2
BFD_DIAGNOSTIC_CODE_NEIGHBOR_SIGNALED_SESSION_DOWN     = 3
BFD_DIAGNOSTIC_CODE_FORWARDING_PLANE_RESET             = 4
BFD_DIAGNOSTIC_CODE_PATH_DOWN                          = 5
BFD_DIAGNOSTIC_CODE_CONCATENATED_PATH_DOWN             = 6
BFD_DIAGNOSTIC_CODE_ADMINISTRATIVELY_DOWN              = 7
BFD_DIAGNOSTIC_CODE_REVERSE_CONCATENATED_PATH_DOWN     = 8
```

### What happens server-side

When `peer.Bfd` is set on an `AddPeerRequest`:

1. `BgpServer.addNeighbor()` (`server.go:3480`) reads `neighbor.Bfd.Config`
2. Calls `bfdServer.AddPeer(ctx, addr, bfdConfig)` where `bfdServer` is an instance of `bfdServer` (`bfd_server.go`)
3. `bfdServer` creates a `bfdPeer` (`bfd_peer.go`) with:
   - UDP socket using dynamic source port (49152-65535 per RFC 5881)
   - TTL=255 on outgoing packets (per RFC 5881)
   - Timers: `txInterval` = DesiredMinimumTxInterval, `rxInterval` = RequiredMinimumReceive, `expiryInterval` = DetectionMultiplier × rxInterval
   - State machine: DOWN → INIT → UP on three-way handshake
4. On expiry or remote DOWN: calls `ResetPeer(addr, soft=false)` with reason "BFD is down"

On `UpdatePeer` / `DeletePeer`, the BFD session is similarly updated or removed (`updateBfdPeer`, `bfdServer.DeletePeer`).

### No dedicated BFD RPCs

The `Gobgp` gRPC service has **no** `AddBfdPeer`/`DeleteBfdPeer`/`ListBfdPeer` RPCs. BFD is handled implicitly through the existing `Peer`/`PeerGroup` messages. Cilium's existing `AddNeighbor`/`UpdateNeighbor`/`RemoveNeighbor` flow is sufficient.

---

## Implementation — 6 Layers

### Layer 1: CRD types

#### `pkg/k8s/apis/cilium.io/v2/bgp_peer_types.go` — `CiliumBGPPeerConfigSpec`

Today's `CiliumBGPPeerConfigSpec` has no BFD field. Users cannot express "I want BFD for this peer" anywhere in the CRD graph.

Add a new `CiliumBGPBFD` struct and a `Bfd *CiliumBGPBFD` field:

```go
type CiliumBGPBFD struct {
    Enabled                  bool     `json:"enabled"`
    Port                     *uint32  `json:"port,omitempty"`                     // UDP port, default 3784
    DesiredMinimumTxInterval *uint32  `json:"desiredMinimumTxInterval,omitempty"` // µs, default 1,000,000
    RequiredMinimumReceive   *uint32  `json:"requiredMinimumReceive,omitempty"`   // µs, default 1,000,000
    DetectionMultiplier      *uint32  `json:"detectionMultiplier,omitempty"`      // default 3
}

type CiliumBGPPeerConfigSpec struct {
    Transport       *CiliumBGPTransport
    Timers          *CiliumBGPTimers
    AuthSecretRef   *string
    GracefulRestart *CiliumBGPNeighborGracefulRestart
    EBGPMultihop    *int32
    Families        []CiliumBGPFamilyWithAdverts
    BFD             *CiliumBGPBFD `json:"bfd,omitempty"`          // <-- NEW
}
```

Design rationale:
- Pointers for timer fields mean "absent → use GoBGP default". The zero-value problem: if we used `uint32` directly, we couldn't distinguish "user set 0" from "user didn't set it".
- `Enabled` is a plain `bool` — it must be explicitly set to `true` for BFD to activate. This avoids accidentally enabling BFD via a default.
- Maps 1:1 to GoBGP's `BfdPeerConfig` protobuf.

#### `pkg/k8s/apis/cilium.io/v2/bgp_node_types.go` — `CiliumBGPNodePeerStatus`

Today's `CiliumBGPNodePeerStatus` has no BFD state field. Users cannot see BFD status via kubectl.

Add a `CiliumBGPBFDState` struct and a `BfdState` field:

```go
type CiliumBGPBFDState struct {
    SessionState         string `json:"sessionState,omitempty"`         // "up"|"down"|"admin_down"|"init"
    LocalDiscriminator   uint32 `json:"localDiscriminator,omitempty"`
    RemoteDiscriminator  uint32 `json:"remoteDiscriminator,omitempty"`
    Diagnostics          string `json:"diagnostics,omitempty"`          // human-readable diagnostic code
}

type CiliumBGPNodePeerStatus struct {
    Name            string                          `json:"name,omitempty"`
    PeerAddress     string                          `json:"peerAddress,omitempty"`
    PeerASN         int64                           `json:"peerASN,omitempty"`
    SessionState    string                          `json:"sessionState,omitempty"`
    Uptime          *int64                          `json:"uptime,omitempty"`
    Timers          *CiliumBGPTimersState           `json:"timers,omitempty"`
    Families        []CiliumBGPPeerFamilyStatus     `json:"families,omitempty"`
    GracefulRestart *CiliumBGPNeighborGraceRestartState `json:"gracefulRestart,omitempty"`
    RouteCount      *CiliumBGPPeerRouteCountStatus  `json:"routeCount,omitempty"`
    BFDState        *CiliumBGPBFDState              `json:"bfdState,omitempty"`          // <-- NEW
}
```

---

### Layer 2: Internal `Neighbor` type

#### `pkg/bgp/types/bgp.go`

The `types.Router` interface is designed to be backend-agnostic. It uses `types.Neighbor` as its internal representation, not GoBGP-specific types. BFD must live here so other hypothetical backends (FRR, Bird, etc.) could also support it.

Add `NeighborBFD` struct and a `BFD *NeighborBFD` field on `Neighbor`:

```go
type NeighborBFD struct {
    Enabled                  bool
    Port                     uint16     // smaller type than protobuf — enough for UDP port
    DesiredMinimumTxInterval uint32     // µs
    RequiredMinimumReceive   uint32     // µs
    DetectionMultiplier      uint8      // smaller type — RFC max is 255
}

type Neighbor struct {
    Name            string
    Address         netip.Addr
    ASN             uint32
    AuthPassword    string
    EbgpMultihop    *NeighborEbgpMultihop
    Timers          *NeighborTimers
    Transport       *NeighborTransport
    GracefulRestart *NeighborGracefulRestart
    AfiSafis        []*Family
    BFD             *NeighborBFD     // <-- NEW
}
```

Also add `PeerBFDState` for the reverse (state) path:

```go
type PeerBFDState struct {
    SessionState        string   // UP/DOWN/INIT/ADMIN_DOWN
    LocalDiscriminator  uint32
    RemoteDiscriminator uint32
}

type PeerState struct {
    // ... existing fields
    BFDState *PeerBFDState
}
```

---

### Layer 3: CRD-to-Neighbor conversion

#### `pkg/bgp/types/conversions.go`

The function `ToNeighborV2()` is called by the **NeighborReconciler** (`pkg/bgp/manager/reconciler/neighbor.go`) with the actual `CiliumBGPPeerConfigSpec` from the cluster. It converts CRD types → `types.Neighbor`.

Each field maps through a `toNeighbor*V2()` helper. For example, `toNeighborTimersV2` maps `CiliumBGPTimers` → `NeighborTimers`, checking for nil pointers.

Add `toNeighborBFDV2()` and one line in `ToNeighborV2()`:

```go
func toNeighborBFDV2(pcBFD *v2.CiliumBGPBFD) *NeighborBFD {
    if pcBFD == nil || !pcBFD.Enabled {
        return nil
    }
    bfd := &NeighborBFD{Enabled: true}
    if pcBFD.Port != nil {
        bfd.Port = uint16(*pcBFD.Port)
    }
    if pcBFD.DesiredMinimumTxInterval != nil {
        bfd.DesiredMinimumTxInterval = *pcBFD.DesiredMinimumTxInterval
    }
    if pcBFD.RequiredMinimumReceive != nil {
        bfd.RequiredMinimumReceive = *pcBFD.RequiredMinimumReceive
    }
    if pcBFD.DetectionMultiplier != nil {
        bfd.DetectionMultiplier = uint8(*pcBFD.DetectionMultiplier)
    }
    return bfd
}
```

Inserted into `ToNeighborV2()`:
```go
neighbor.BFD = toNeighborBFDV2(pc.BFD)
```

If `pc.BFD` is nil or `Enabled: false`, `toNeighborBFDV2` returns nil, which means "no BFD" — this is safe to pass through the rest of the pipeline.

---

### Layer 4: Neighbor-to-GoBGP conversion

#### `pkg/bgp/gobgp/conversions.go`

The function `ToGoBGPPeer()` is called by `GoBGPServer.AddNeighbor()` and `UpdateNeighbor()` to build the protobuf `gobgp.Peer` that is sent to `server.AddPeer()`.

Today it builds:

```go
newPeer := &gobgp.Peer{}
newPeer.Conf = toGoBGPPeerConf(n, oldPeer)
newPeer.EbgpMultihop = toGoBGPEbgpMultihop(n.EbgpMultihop)
newPeer.Timers = toGoBGPTimers(n.Timers)
newPeer.Transport = toGoBGPTransport(n.Transport, oldPeer, v4)
newPeer.GracefulRestart = toGoBGPGracefulRestart(n.GracefulRestart)
newPeer.AfiSafis = toGoBGPAfiSafi(n.AfiSafis, newPeer.GracefulRestart)
```

**`newPeer.Bfd` is never set** — this is the root cause of "no BFD."

Add `toGoBGPBFD()` and one line in `ToGoBGPPeer()`:

```go
func toGoBGPBFD(n *types.NeighborBFD) *gobgp.BfdPeerConfig {
    if n == nil || !n.Enabled {
        return nil
    }
    return &gobgp.BfdPeerConfig{
        Enabled:                  n.Enabled,
        Port:                     uint32(n.Port),
        DesiredMinimumTxInterval: n.DesiredMinimumTxInterval,
        RequiredMinimumReceive:   n.RequiredMinimumReceive,
        DetectionMultiplier:      uint32(n.DetectionMultiplier),
    }
}
```

Inserted into `ToGoBGPPeer()`:
```go
newPeer.Bfd = toGoBGPBFD(n.BFD)
```

Once `peer.Bfd` is populated, `server.AddPeer(ctx, &gobgp.AddPeerRequest{Peer: peer})` triggers GoBGP's `addNeighbor()` which calls `bfdServer.AddPeer()`. The BFD session starts automatically.

**UpdateNeighbor also works**: GoBGP's `updateBfdPeer()` (called from `UpdatePeer`) deletes the old BFD peer and adds the new one with the updated config. No additional Cilium plumbing needed.

---

### Layer 5: State collection

#### `pkg/bgp/gobgp/state.go`

The function `GetPeerState()` calls `g.server.ListPeer()` and iterates through GoBGP `Peer` objects. For each peer, it builds a `types.PeerState`. Today it reads session state, ASNs, timers, capabilities, families, graceful restart, etc. — but not BFD state.

Each GoBGP `Peer` has `peer.State.BfdState` (type `*gobgp.BfdPeerState`), which is automatically populated by the BFD subsystem. We just need to extract it.

Add extraction in the ListPeer callback:

```go
var bfdState *types.PeerBFDState
if ps := peer.GetState(); ps != nil {
    if bfs := ps.GetBfdState(); bfs != nil {
        bfdState = &types.PeerBFDState{
            SessionState:        bfs.SessionState.String(),
            LocalDiscriminator:  bfs.LocalDiscriminator,
            RemoteDiscriminator: bfs.RemoteDiscriminator,
        }
    }
}
// Add bfdState to the PeerState being constructed
```

#### `pkg/bgp/types/bgp.go` (PeerBFDState — already covered in Layer 2)

---

### Layer 6: CRD status update

#### `pkg/bgp/manager/reconciler/crd_status.go`

The `StatusReconciler` reads `types.PeerState` from `Router.GetPeerStateLegacy()` and patches `CiliumBGPNodeConfig.Status`. Today it maps peer state fields to `CiliumBGPNodePeerStatus` fields.

Add mapping from `PeerState.BFDState` → `CiliumBGPNodePeerStatus.BFDState`:

```go
if peerState.BFDState != nil {
    status.BFDState = &v2.CiliumBGPBFDState{
        SessionState:        peerState.BFDState.SessionState,
        LocalDiscriminator:  peerState.BFDState.LocalDiscriminator,
        RemoteDiscriminator: peerState.BFDState.RemoteDiscriminator,
    }
}
```

---

## End-to-end data flow

```
User writes CiliumBGPPeerConfig with bfd.enabled=true
  │
  ▼
CiliumBGPNodeConfig (rendered by operator with the peer config ref)
  │
  ▼
Agent Controller.Reconcile() → BGPRouterManager.ReconcileInstances()
  │
  ▼
NeighborReconciler.Reconcile()  [priority 110, runs last]
  │
  ├── getPeerConfig(name) → CiliumBGPPeerConfigSpec
  │     with BFD: &CiliumBGPBFD{Enabled: true, ...}
  │
  ├── getPeerPassword(name) → secret string or ""
  │
  ├── ToNeighborV2(np, pc, password) → *types.Neighbor
  │     .BFD = toNeighborBFDV2(pc.BFD)
  │     // produces NeighborBFD{Enabled: true, Port: 3784, ...}
  │
  └── Router.AddNeighbor(ctx, neighbor)        [types.Router interface]
        │
        ▼
      GoBGPServer.AddNeighbor(ctx, n)           [pkg/bgp/gobgp/peer.go]
        │
        ├── ToGoBGPPeer(n, nil, true) → *gobgp.Peer
        │     .Bfd = toGoBGPBFD(n.BFD)
        │     // produces gobgp.BfdPeerConfig{Enabled: true, Port: 3784, ...}
        │
        └── g.server.AddPeer(ctx, &gobgp.AddPeerRequest{Peer: peer})
              │
              ▼
            GoBGP's addNeighbor()                [server.go:3480]
              │
              ├── Parses peer.Bfd → oc.BfdConfig
              │     via newBfdConfigFromAPIStruct()
              │
              └── bfdServer.AddPeer(ctx, addr, bfdConfig)
                    │
                    ▼
                  bfdPeer.start()                [bfd_peer.go]
                    │
                    ├── Opens UDP socket (dynamic source port 49152-65535)
                    ├── Sets TTL=255
                    ├── Starts tx timer (sends BFD control packets)
                    ├── Starts rx timer (expects packets from peer)
                    └── Runs BFD state machine: DOWN→INIT→UP
                          │
                          ▼
                        On failure: ResetPeer(addr, soft=false)
                          "BFD is down"
                          │
                          ▼
                        BGP session drops, routes withdrawn
```

## Reverse path (state → user)

```
GoBGP peer.State.BfdState populated by bfdServer
  │
  ▼
GoBGPServer.GetPeerState() — ListPeer callback
  │
  ├── peer.GetState().GetBfdState() → *gobgp.BfdPeerState
  └── → *types.PeerBFDState
        │
        ▼
      StatusReconciler
        │
        ├── → CiliumBGPBFDState
        └── Patches CiliumBGPNodeConfig.Status.BFDState
              │
              ▼
            kubectl get ciliumbgpnodeconfig -o json
              → bfdState.sessionState: "up"
```

---

## Files Changed (summary)

| File | Change |
|------|--------|
| `pkg/k8s/apis/cilium.io/v2/bgp_peer_types.go` | Add `CiliumBGPBFD` struct + `BFD *CiliumBGPBFD` field on `CiliumBGPPeerConfigSpec` |
| `pkg/k8s/apis/cilium.io/v2/bgp_node_types.go` | Add `CiliumBGPBFDState` struct + `BFDState *CiliumBGPBFDState` field on `CiliumBGPNodePeerStatus` |
| `pkg/bgp/types/bgp.go` | Add `NeighborBFD` struct, `PeerBFDState` struct + fields on `Neighbor` and `PeerState` |
| `pkg/bgp/types/conversions.go` | Add `toNeighborBFDV2()` + call it from `ToNeighborV2()` |
| `pkg/bgp/gobgp/conversions.go` | Add `toGoBGPBFD()` + call it from `ToGoBGPPeer()` |
| `pkg/bgp/gobgp/state.go` | Extract BFD state from `peer.State.BfdState` in `GetPeerState()` |
| `pkg/bgp/manager/reconciler/crd_status.go` | Map `types.PeerBFDState` → `CiliumBGPBFDState` in status reconciler |

## Files NOT changing

These are generic and require no BFD-specific logic:
- `pkg/bgp/cell.go` — Hive wiring, backend-agnostic
- `pkg/bgp/agent/controller.go` — event loop, backend-agnostic
- `pkg/bgp/manager/manager.go` — instance lifecycle, backend-agnostic
- `pkg/bgp/manager/reconciler/reconcilers.go` — reconciler framework, backend-agnostic
- `pkg/bgp/manager/reconciler/neighbor.go` — calls `Router.AddNeighbor()` with `*types.Neighbor`, generic
- `pkg/bgp/gobgp/server.go` — GoBGP server setup, unchanged (BFD is per-peer, not global)
- `pkg/bgp/gobgp/peer.go` — already wraps `ToGoBGPPeer()`, just passes through the conversion

## GoBGP vendored files that make this work (no changes needed)

| Vendored file | Role |
|---------------|------|
| `vendor/.../gobgp/v4/api/gobgp.pb.go` | `BfdPeerConfig`, `BfdPeerState` protobuf types |
| `vendor/.../gobgp/v4/pkg/server/bfd_server.go` | BFD UDP listener and peer management |
| `vendor/.../gobgp/v4/pkg/server/bfd_peer.go` | Per-peer BFD state machine |
| `vendor/.../gobgp/v4/pkg/server/server.go` | `addNeighbor()` wires BFD, `getPeerState()` returns BFD state |
| `vendor/.../gobgp/v4/pkg/server/grpc_server.go` | `newBfdConfigFromAPIStruct()` converts protobuf → internal config |
| `vendor/.../gobgp/v4/pkg/config/oc/bgp_configs.go` | `BfdConfig`, `BfdState` OC model structs |
| `vendor/.../gobgp/v4/pkg/packet/bfd/bfd.go` | Wire protocol: `BFDHeader` marshal/unmarshal |
