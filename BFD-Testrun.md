# BFD Support for Cilium BGP Control Plane — Test Run

**Date:** 2026-07-08
**Branch:** `bfd-bgp-control-plane-support`
**Commit:** `26eeb36a88`
**Infrastructure:** Kind + ContainerLab + FRR 8.4 + Cilium (local build)

## 1. Topology

```
┌──────────────────────────────────────────────────────────┐
│                     Kind Cluster                         │
│                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐       │
│  │ control-plane       │  │ worker               │       │
│  │ (server0 in clab)   │  │ (server1 in clab)    │       │
│  │ fd00:10:0:1::2      │  │ fd00:10:0:2::2       │       │
│  │ BGP AS 65001        │  │ BGP AS 65001         │       │
│  │ GoBGP + BFD         │  │ GoBGP + BFD          │       │
│  └────────┬────────────┘  └────────┬────────────┘       │
│           │ net0                    │ net1                │
│           │ 10.0.1.0/24            │ 10.0.2.0/24         │
│           │ fd00:10:0:1::0/64      │ fd00:10:0:2::0/64   │
└───────────┼────────────────────────┼─────────────────────┘
            │                        │
       ┌────┴────────────────────────┴────┐
       │           router0 (FRR)          │
       │  net0: fd00:10:0:1::1/64         │
       │  net1: fd00:10:0:2::1/64         │
       │  loopback: fd00:10:0:0::1/128    │
       │  BGP AS 65000                    │
       │  BFD peer (active mode)          │
       └──────────────────────────────────┘
```

- **eBGP peering**: AS 65001 (Cilium) ↔ AS 65000 (FRR)
- **BGP peer address**: `fd00:10::1` (FRR loopback)
- **BFD timers**: TxInterval=300ms, RxInterval=300ms, DetectMultiplier=3 (900ms detection)

## 2. CRD Configuration Applied

### CiliumBGPPeerConfig with BFD

```yaml
apiVersion: cilium.io/v2
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  authSecretRef: bgp-auth-secret
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  bfd:
    enabled: true
    desiredMinTxInterval: 300000
    requiredMinRxInterval: 300000
    detectionMultiplier: 3
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
    - afi: ipv6
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
```

### CiliumBGPClusterConfig

```yaml
apiVersion: cilium.io/v2
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      bgp: "65001"
  bgpInstances:
  - name: "65001"
    localASN: 65001
    peers:
    - name: "65000"
      peerASN: 65000
      peerAddress: fd00:10::1
      peerConfigRef:
        name: "cilium-peer"
```

### FRR BFD Configuration

```
frr version 8.4_git
hostname router0

router bgp 65000
 bgp router-id 10.0.0.1
 no bgp ebgp-requires-policy
 bgp default ipv6-unicast
 bgp bestpath as-path multipath-relax
 neighbor CILIUM peer-group
 neighbor CILIUM remote-as external
 neighbor CILIUM password cilium123
 neighbor fd00:10:0:1::2 peer-group CILIUM
 neighbor fd00:10:0:1::2 bfd
 neighbor fd00:10:0:2::2 peer-group CILIUM
 neighbor fd00:10:0:2::2 bfd

bfd
 peer fd00:10:0:1::2 local-address fd00:10::1
 exit
 !
 peer fd00:10:0:2::2 local-address fd00:10::1
 exit
 !
```

> **Key:** `local-address fd00:10::1` forces FRR to source BFD packets from its
> loopback address, matching the address GoBGP uses to register BFD peers.

## 3. Command Outputs

### 3.1 Cilium BGP Peers

```
$ cilium bgp peers

Node                                   Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
bgp-cplane-dev-service-control-plane   65001      65000     fd00:10::1     established     2m14s    ipv4/unicast   2          1
                                                                                                    ipv6/unicast   2          1
bgp-cplane-dev-service-worker          65001      65000     fd00:10::1     established     2m14s    ipv4/unicast   2          1
                                                                                                    ipv6/unicast   2          1
```

### 3.2 Cilium BGP Routes

```
$ cilium bgp routes available ipv4 unicast

Node                                   VRouter   Prefix        NextHop   Age     Attrs
bgp-cplane-dev-service-control-plane   65001     10.1.0.0/24   0.0.0.0   4m23s   [{Origin: i} {Nexthop: 0.0.0.0}]
bgp-cplane-dev-service-worker          65001     10.1.1.0/24   0.0.0.0   4m23s   [{Origin: i} {Nexthop: 0.0.0.0}]

$ cilium bgp routes available ipv6 unicast

Node                                   VRouter   Prefix             NextHop   Age     Attrs
bgp-cplane-dev-service-control-plane   65001     fd00:10:1::/64     ::        4m23s   [{Origin: i} {MpReach(ipv6-unicast): {Nexthop: ::, NLRIs: [fd00:10:1::/64:0]}}]
bgp-cplane-dev-service-worker          65001     fd00:10:1:1::/64   ::        4m23s   [{Origin: i} {MpReach(ipv6-unicast): {Nexthop: ::, NLRIs: [fd00:10:1:1::/64:0]}}]
```

### 3.3 FRR BFD Peers

```
$ docker exec clab-bgp-cplane-dev-service-router0 vtysh -c "show bfd peer"

BFD Peers:
	peer fd00:10:0:2::2 local-address fd00:10::1 vrf default
		ID: 2535773534
		Remote ID: 4214225209
		Active mode
		Status: up
		Uptime: 1 minute(s), 7 second(s)
		Diagnostics: ok
		Remote diagnostics: ok
		Peer Type: configured
		RTT min/avg/max: 0/0/0 usec
		Local timers:
			Detect-multiplier: 3
			Receive interval: 300ms
			Transmission interval: 300ms
			Echo receive interval: 50ms
			Echo transmission interval: disabled
		Remote timers:
			Detect-multiplier: 3
			Receive interval: 300ms
			Transmission interval: 300ms
			Echo receive interval: disabled

	peer fd00:10:0:1::2 local-address fd00:10::1 vrf default
		ID: 2955421877
		Remote ID: 3185939673
		Active mode
		Status: up
		Uptime: 1 minute(s), 7 second(s)
		Diagnostics: ok
		Remote diagnostics: ok
		Peer Type: configured
		RTT min/avg/max: 0/0/0 usec
		Local timers:
			Detect-multiplier: 3
			Receive interval: 300ms
			Transmission interval: 300ms
			Echo receive interval: 50ms
			Echo transmission interval: disabled
		Remote timers:
			Detect-multiplier: 3
			Receive interval: 300ms
			Transmission interval: 300ms
			Echo receive interval: disabled
```

### 3.4 FRR BGP Summary

```
$ docker exec clab-bgp-cplane-dev-service-router0 vtysh -c "show bgp summary"

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0
BGP table version 4
RIB entries 3, using 576 bytes of memory
Peers 2, using 1434 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
fd00:10:0:1::2  4      65001        18        13        0    0    0 00:02:08            1        2 N/A
fd00:10:0:2::2  4      65001        18        13        0    0    0 00:02:08            1        2 N/A

Total number of neighbors 2

IPv6 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0
BGP table version 4
RIB entries 3, using 576 bytes of memory
Peers 2, using 1434 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
fd00:10:0:1::2  4      65001        18        13        0    0    0 00:02:08            1        2 N/A
fd00:10:0:2::2  4      65001        18        13        0    0    0 00:02:08            1        2 N/A

Total number of neighbors 2
```

### 3.5 CiliumBGPNodeConfig Status (CRD)

```
$ kubectl get ciliumbgpnodeconfig -o yaml

- apiVersion: cilium.io/v2
  kind: CiliumBGPNodeConfig
  metadata:
    name: bgp-cplane-dev-service-control-plane
  spec:
    bgpInstances:
    - localASN: 65001
      name: "65001"
      peers:
      - name: "65000"
        peerASN: 65000
        peerAddress: fd00:10::1
        peerConfigRef:
          name: cilium-peer
  status:
    bgpInstances:
    - localASN: 65001
      name: "65001"
      peers:
      - bfdState:
          failureTransitions: 0
          localDiagnosticCode: no_diagnostic
          localDiscriminator: 0
          remoteDiagnosticCode: no_diagnostic
          remoteDiscriminator: 0
          remoteSessionState: unspecified
          sessionState: up
        establishedTime: "2026-07-08T00:29:54Z"
        name: "65000"
        peerASN: 65000
        peerAddress: fd00:10::1
        peeringState: established
        routeCount:
        - advertised: 1
          afi: ipv4
          received: 0
          safi: unicast
        - advertised: 1
          afi: ipv6
          received: 0
          safi: unicast
        timers:
          appliedHoldTimeSeconds: 90
          appliedKeepaliveSeconds: 30

- apiVersion: cilium.io/v2
  kind: CiliumBGPNodeConfig
  metadata:
    name: bgp-cplane-dev-service-worker
  status:
    bgpInstances:
    - localASN: 65001
      name: "65001"
      peers:
      - bfdState:
          failureTransitions: 0
          localDiagnosticCode: no_diagnostic
          localDiscriminator: 0
          remoteDiagnosticCode: no_diagnostic
          remoteDiscriminator: 0
          remoteSessionState: unspecified
          sessionState: up
        establishedTime: "2026-07-08T00:29:54Z"
        name: "65000"
        peerASN: 65000
        peerAddress: fd00:10::1
        peeringState: established
        routeCount:
        - advertised: 1
          afi: ipv4
          received: 0
          safi: unicast
        - advertised: 1
          afi: ipv6
          received: 0
          safi: unicast
        timers:
          appliedHoldTimeSeconds: 90
          appliedKeepaliveSeconds: 30
```

### 3.6 CiliumBGPPeerConfig Status

```
$ kubectl get ciliumbgppeerconfig -o yaml

- apiVersion: cilium.io/v2
  kind: CiliumBGPPeerConfig
  metadata:
    name: cilium-peer
  spec:
    authSecretRef: bgp-auth-secret
    bfd:
      desiredMinTxInterval: 300000
      detectionMultiplier: 3
      enabled: true
      requiredMinRxInterval: 300000
    ebgpMultihop: 1
    families:
    - advertisements:
        matchLabels:
          advertise: bgp
      afi: ipv4
      safi: unicast
    - advertisements:
        matchLabels:
          advertise: bgp
      afi: ipv6
      safi: unicast
    gracefulRestart:
      enabled: true
      restartTimeSeconds: 15
  status:
    conditions:
    - lastTransitionTime: "2026-07-08T00:27:45Z"
      message: ""
      observedGeneration: 1
      reason: AuthSecretValidated
      status: "False"
      type: cilium.io/MissingAuthSecret
```

### 3.7 GoBGP BFD Lifecycle Logs (control-plane node)

```
$ kubectl -n kube-system logs <cilium-pod> -c cilium-agent | grep -i "bfd" | grep -v "Can't send"

# 1. BFD peer registered in GoBGP server
time=2026-07-08T00:27:45.063395102Z level=info  msg="Insert BFD peer"
  source=bfd_server.go:302  Peer=fd00:10::1

# 2. BFD server starts listening on UDP 3784
time=2026-07-08T00:27:46.064120959Z level=info  msg="BFD server is started"
  source=bfd_server.go:261   Address=:3784

# 3. BFD client starts sending to FRR loopback
time=2026-07-08T00:27:46.064156735Z level=debug msg="BFD client is started"
  source=bfd_peer.go:248      Peer=fd00:10::1
  LocalAddress=:58489  RemoteAddress=[fd00:10::1]:3784

# 4. BFD session transitions to UP (after ~2 min of "Can't send UDP packet" errors
#    while BFD server socket was still initializing)
time=2026-07-08T00:29:36.696777042Z level=debug msg="Set state to UP"
  source=bfd_peer.go:435      Peer=fd00:10::1

# 5. Brief flap: FRR BFD restarted (due to config changes), remote signals Down
time=2026-07-08T00:29:47.603699727Z level=warn  msg="Remote peer signaled BFD down"
  source=bfd_peer.go:330      Peer=fd00:10::1

# 6. BFD state goes DOWN, BGP session torn down with "BFD is down" notification
time=2026-07-08T00:29:47.603762835Z level=debug msg="Set state to DOWN"
  source=bfd_peer.go:413      Peer=fd00:10::1

time=2026-07-08T00:29:47.603794334Z level=info  msg="sent notification"
  source=fsm.go:784           Key=fd00:10::1  State=BGP_FSM_ESTABLISHED
  Code=6 Subcode=4  Communicated-Reason="BFD is down"

# 7. BFD recovers to UP within 200ms
time=2026-07-08T00:29:47.814788783Z level=debug msg="Set state to UP"
  source=bfd_peer.go:435      Peer=fd00:10::1
```

### 3.8 FRR BFD Logs

```
$ docker exec clab-bgp-cplane-dev-service-router0 cat /var/log/frr.log | grep -i bfd | tail -10

2026/07/08 00:28:50 bfdd[5198]: [H6RJW-WMSRG] bfd_rx_process_packet peer fd00:10:0:1::2
  local-address fd00:10::1: switch state from init to up
2026/07/08 00:28:50 bfdd[5198]: [H6RJW-WMSRG] bfd_rx_process_packet peer fd00:10:0:2::2
  local-address fd00:10::1: switch state from init to up
```

## 4. Key Findings

### 4.1 Source Address Mismatch — The Critical Fix

**Problem:** GoBGP registers BFD peers by BGP peer address (`fd00:10::1`). When it receives BFD control packets, it looks up the peer by source IP. FRR by default sources BFD packets from the interface address (`fd00:10:0:1::1`), not the loopback (`fd00:10::1`). This causes "Unknown BFD peer" errors on the GoBGP side.

**Solution:** Configure FRR BFD peers with `local-address fd00:10::1` (the loopback):

```
bfd
 peer fd00:10:0:1::2 local-address fd00:10::1
 peer fd00:10:0:2::2 local-address fd00:10::1
```

This forces FRR to bind its BFD UDP socket to the loopback, so source IP in outgoing BFD packets matches what GoBGP expects.

### 4.2 BFD Session Lifecycle

1. **Insert BFD peer** — GoBGP registers peer when CiliumBGPPeerConfig with `bfd.enabled: true` is applied
2. **BFD server started** — Listens on UDP 3784 (single-hop, per RFC 5881)
3. **BFD client started** — Sends BFD control packets to peer
4. **State: UP** — Both sides exchange BFD control packets successfully
5. **On failure**: BFD DOWN → BGP NOTIFICATION (Cease/Administrative Reset, "BFD is down") → TCP teardown → route withdrawal → BGP reconverges

### 4.3 CRD Status Field Observations

| Field | Value | Notes |
|-------|-------|-------|
| `sessionState` | `up` | Correct — BFD session is active |
| `localDiagnosticCode` | `no_diagnostic` | Correct — no local issues |
| `remoteDiagnosticCode` | `no_diagnostic` | Correct — remote peer is healthy |
| `localDiscriminator` | `0` | GoBGP API limitation — always returns 0 via gRPC |
| `remoteDiscriminator` | `0` | GoBGP API limitation — always returns 0 via gRPC |
| `remoteSessionState` | `unspecified` | GoBGP API limitation — not populated in GetPeerState() |
| `failureTransitions` | `0` | GoBGP limitation — field exists but is never incremented (dead code in bfd_peer.go) |

### 4.4 Files Modified for BFD Support

| File | Change |
|------|--------|
| `pkg/k8s/apis/cilium.io/v2/bgp_peer_types.go` | Added `CiliumBGPBFD` struct, `BFD` field on `CiliumBGPPeerConfigSpec` |
| `pkg/k8s/apis/cilium.io/v2/bgp_node_types.go` | Added `CiliumBGPBFDState` struct, `BFDState` field on `CiliumBGPNodePeerStatus` |
| `pkg/bgp/types/bgp.go` | Added `NeighborBFD`, `PeerBFDState` structs |
| `pkg/bgp/types/conversions.go` | Added `toNeighborBFDV2()` conversion function |
| `pkg/bgp/gobgp/conversions.go` | Added `toGoBGPBFD()` conversion function |
| `pkg/bgp/gobgp/state.go` | Added BFD state extraction with enum mappers |
| `pkg/bgp/manager/reconciler/crd_status.go` | Added BFD state mapping to CRD status |
| `pkg/k8s/apis/cilium.io/v2/zz_generated.deepcopy.go` | Regenerated with BFD types |
| `pkg/k8s/apis/cilium.io/v2/zz_generated.deepequal.go` | Regenerated with BFD types |
| `pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgppeerconfigs.yaml` | Regenerated with BFD schema |
| `pkg/k8s/apis/cilium.io/client/crds/v2/ciliumbgpnodeconfigs.yaml` | Regenerated with BFD state schema |

### 4.5 Test Files

| File | Coverage |
|------|----------|
| `pkg/bgp/types/conversions_test.go` | 4 BFD cases: enabled+defaults, enabled+custom, disabled, nil |
| `pkg/bgp/gobgp/conversions_test.go` | 3 BFD cases: enabled, nil, disabled |
| `pkg/bgp/gobgp/state_test.go` | TestToAgentBfdSessionState (6 cases), TestToAgentBfdDiagnosticCode (11 cases), TestGetPeerStateWithBFD (integration) |

## 5. Failure Detection Demo (Verified)

### 5.1 Test Setup

With BGP+BFD fully operational, we added FRR-originated routes (`10.99.0.0/24` via `redistribute connected`) to verify route propagation and withdrawal. Then killed the BFD daemon on FRR:

```bash
# Add FRR-originated routes to test withdrawal
docker exec clab-bgp-cplane-dev-service-router0 vtysh -c "
conf t
interface lo
 ip address 10.99.0.1/24
exit
router bgp 65000
 redistribute connected
exit
"

# Kill BFD daemon (BGP stays alive)
docker exec clab-bgp-cplane-dev-service-router0 bash -c "kill \$(cat /var/run/frr/bfdd.pid)"
```

### 5.2 Verified Timeline

```
00:44:37.815  BFD: "Remote peer signaled BFD down" → "Set state to DOWN"
              ← FRR's bfdd dies, sends BFD AdminDown to GoBGP

00:44:37.815  BGP: sent NOTIFICATION "BFD is down" (Code=6, Subcode=4, Administrative Reset)
              ← GoBGP immediately tears down BGP session

00:44:37.816  BGP: ESTABLISHED → IDLE
              ← Routes WITHDRAWN from adj-RIB-in at this moment
              ← CRD status: bfdState.sessionState: down

00:44:38-42   BFD: "Can't send UDP packet" (bfdd is dead, socket still open)

00:44:42.817  BGP: IdleHoldTimer expired (5s) → IDLE → ACTIVE
              ← GoBGP auto-reconnects! (BFD still DOWN)

00:44:44.351  BGP: TCP connected → OPENCONFIRM → ESTABLISHED
              ← BGP session re-established WITHOUT BFD

00:44:44.352  Routes re-advertised: 10.1.0.0/24, fd00:10:1::/64
              ← Withdrawal reversed after ~7 seconds
```

### 5.3 What Happened in the Code

```
BFD detects remote down (bfd_peer.go:330)
  → resetPeer(): calls BgpServer.ResetPeer(Soft=false, Communication="BFD is down")
  → BgpServer sends BGP NOTIFICATION: Code=CEASE(6), Subcode=ADMINISTRATIVE_RESET(4)
  → FSM: sends NOTIFICATION on TCP wire, closes TCP connection
  → ESTABLISHED → IDLE
  → handleFSMMessage(): withdraws ALL routes from peer's adj-RIB-in
  → propagates withdrawals to global RIB → best-path recalculation
  → GoBGP watcher notifies Cilium → CRD status: bfdState.sessionState: down

  ... 5 seconds later ...

  BGP IdleHoldTimer fires → FSM: IDLE → ACTIVE → connects to FRR
  → BGP re-establishes WITHOUT checking BFD state
  → Routes re-advertised (withdrawal reversed)
```

### 5.4 Key Finding: Route Withdrawal Is Temporary

**BFD-triggered route withdrawal happens but is reversed within ~7 seconds** because:

1. GoBGP's BGP FSM has an `IdleHoldTimer` (5s default) that auto-reconnects after a hard reset
2. The reconnect logic does **not** check BFD session state before reconnecting
3. Since FRR's `bgpd` is still alive and accepting TCP connections, BGP re-establishes
4. Routes are re-advertised, undoing the withdrawal

This is a **GoBGP limitation**, not a Cilium limitation. In production BGP implementations (Cisco IOS, Junos, FRR):
- BGP checks BFD state before attempting reconnection
- If BFD is DOWN, BGP stays in IDLE until BFD comes back UP
- This prevents route flapping during persistent failures

### 5.5 Implications

| Scenario | Behavior | Impact |
|----------|----------|--------|
| **Transient BFD flap** (<5s) | Routes withdrawn, then re-advertised | Brief route flap (~7s) |
| **Persistent failure** (bfdd killed) | Routes withdrawn, BGP reconnects, routes re-advertised | **Route flap every ~7s** — BGP keeps reconnecting while BFD stays down |
| **Link down** | TCP connection fails, BGP backs off | Routes stay withdrawn (TCP timeout) |

For persistent BFD failures (like bfdd crash), this creates a **route flapping** scenario where BGP keeps re-establishing and routes keep being withdrawn/re-advertised. This is undesirable behavior that should be addressed by either:
- Upstream GoBGP fix: Check BFD state before BGP reconnection
- Cilium-level fix: Disable BGP auto-reconnect when BFD is configured and DOWN

## 6. BFD vs. BGP Without BFD — Blackhole Avoidance

### 6.1 The Problem: Silent Blackholes

Without BFD, BGP relies on its **hold timer** (default 90 seconds) to detect a dead peer. During that window:

```
WITHOUT BFD:
  Link/FRR dies
  → BGP hold timer starts counting (90s default)
  → For ~90s, Cilium nodes STILL have routes pointing to dead FRR
  → Traffic sent toward FRR = BLACKHOLED silently
  → After 90s, hold timer expires → BGP NOTIFICATION → routes withdrawn
```

The router continues forwarding packets into a dead link. Applications experience hangs or silent data loss.

### 6.2 How BFD Solves This

BFD uses lightweight UDP probes (every 300ms in our config) with sub-second detection:

```
WITH BFD:
  Link/FRR dies
  → BFD detects in ~900ms (3 × 300ms)
  → GoBGP tears down BGP session immediately
  → Routes withdrawn from RIB within ~1 second
  → Traffic that would have been blackholed is instead:
     - Dropped locally with ICMP unreachable (if no alternative path)
     - Rerouted via backup path (if ECMP or redundant peers exist)
```

### 6.3 What BFD Actually Provides

| Aspect | BGP alone | BGP + BFD |
|--------|-----------|-----------|
| **Detection time** | ~90s (hold timer) | **<1s** (900ms = 3×300ms) |
| **Detection mechanism** | TCP keepalive timeout | Dedicated UDP probes (UDP 3784) |
| **Detection scope** | Dead peer OR network partition | Dead peer OR network partition OR blackhole |
| **Notification** | "Hold timer expired" | **"BFD is down"** (Cease/Administrative Reset) |
| **Route withdrawal** | After ~90s delay | **Within 1 second** |
| **Application impact** | 90s of hangs/timeouts | <1s of interruption |

### 6.4 Topology Matters

- **Single path (our lab)**: BFD detects fast → routes withdrawn fast → applications fail fast (ICMP unreachable). No silent blackhole, but no automatic reroute.
- **Redundant paths (production)**: BFD detects fast → routes withdrawn fast → traffic fails over to backup path within seconds. This is where BFD truly shines.

## 7. Reproduction Steps

```bash
# 1. Build local images
make dev-docker-image DOCKER_IMAGE_TAG=local
make dev-docker-operator-generic-image DOCKER_IMAGE_TAG=local

# 2. Tag and load into kind
docker tag quay.io/cilium/cilium-dev:local localhost:5000/cilium/cilium-dev:local
docker tag quay.io/cilium/operator-generic:local localhost:5000/cilium/operator-generic:local
kind load docker-image localhost:5000/cilium/cilium-dev:local \
    localhost:5000/cilium/operator-generic:local -n bgp-cplane-dev-service

# 3. Create kind cluster + containerlab
cd contrib/containerlab/service
kind create cluster --config cluster.yaml
sudo containerlab -t topo.yaml deploy  # requires password

# 4. Setup
kubectl taint nodes bgp-cplane-dev-service-control-plane \
    node-role.kubernetes.io/control-plane:NoSchedule-
kubectl -n kube-system create secret generic --type=string bgp-auth-secret \
    --from-literal=password=cilium123

# 5. Install Cilium with local images
helm install cilium -n kube-system install/kubernetes/cilium/ \
    -f contrib/containerlab/service/values.yaml \
    --set image.override="localhost:5000/cilium/cilium-dev:local" \
    --set image.pullPolicy=Never \
    --set operator.image.override="localhost:5000/cilium/operator-generic:local" \
    --set operator.image.pullPolicy=Never
cilium status --wait --namespace kube-system

# 6. Apply BGP + BFD config
kubectl apply -f contrib/containerlab/service/bgp.yaml

# 7. Configure FRR BFD (with local-address fix — see Section 4.1)
docker exec clab-bgp-cplane-dev-service-router0 vtysh -c "
conf t
bfd
 peer fd00:10:0:1::2 local-address fd00:10::1
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
 peer fd00:10:0:2::2 local-address fd00:10::1
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
exit
router bgp 65000
 neighbor fd00:10:0:1::2 bfd
 neighbor fd00:10:0:2::2 bfd
exit
"

# 8. Verify
cilium bgp peers                           # BGP established
docker exec ... vtysh -c "show bfd peer"   # BFD up on both peers
kubectl get ciliumbgpnodeconfig -o yaml    # bfdState.sessionState: up
```

## 8. Bidirectional Test — FRR Detects Cilium Node Death

### 8.1 Goal

Verify that when a Cilium node dies, FRR's BFD detects the failure and withdraws the node's routes from the FRR routing table.

### 8.2 Simulating Node Death

ContainerLab keeps network namespaces alive even after `docker stop`/`docker kill`. To truly simulate a node death, we must bring down the FRR-side interface:

```bash
# This does NOT work — BFD stays UP because namespace survives:
docker stop clab-bgp-cplane-dev-service-server1
docker kill clab-bgp-cplane-dev-service-server1
nsenter --net=/var/run/docker/netns/clab-bgp-cplane-dev-service-server1 -- kill -9 -1
ip netns delete clab-bgp-cplane-dev-service-server1

# This DOES work — kills the link at kernel level:
docker exec clab-bgp-cplane-dev-service-router0 ip link set net1 down
```

### 8.3 Verified Timeline (FRR Side)

```
T+0ms     docker exec FRR ip link set net1 down
          ← FRR-side veth goes DOWN, BFD packets stop

T+~900ms  FRR BFD: "control detection time expired"
          ← 3 × 300ms = 900ms detection window

T+~900ms  FRR BFD: Status: down

T+~900ms  FRR BGP: 10.1.1.0/24 loses *> flag (invalid/best)
          ← Routes from dead node pulled from RIB

T+~900ms  FRR BGP: PfxSnt drops from 7 → 5
          ← Dead node's routes stop being advertised

T+~90s    FRR BGP: TCP session finally drops (hold timer expiry)
          ← BGP TCP lingers until hold timer (90s default)
```

### 8.4 Key Findings

**FRR's `neighbor bfd` behavior:**
- BFD detects failure in ~900ms (sub-second)
- Routes from the dead node are **invalidated immediately** (lose best-path status)
- Route **advertisements stop** (PfxSnt drops)
- BGP TCP session **lingers** until hold timer expires — this is harmless since the routing table is already correct

**Infrastructure caveat:**
- `docker stop` ≠ real node death in ContainerLab
- ContainerLab manages network namespaces independently from Docker containers
- To simulate real node failure: must destroy the network link (`ip link set netX down`)

### 8.5 Bidirectional Summary

| Direction | Detection Time | Route Withdrawal | BGP Session |
|-----------|---------------|-----------------|-------------|
| **Cilium → FRR** (kill bfdd) | <1s | Routes withdrawn, re-advertised in ~7s (GoBGP auto-reconnects) | Auto-reconnects without checking BFD |
| **FRR → Cilium** (kill node) | ~900ms | Routes invalidated, advertisements stop | Lingers until hold timer (90s) |

Both directions show sub-second BFD detection. The asymmetry in BGP session behavior is due to:
- **GoBGP**: Auto-reconnects without checking BFD state (limitation)
- **FRR**: Holds BGP session until hold timer expires (correct behavior — routing table is already updated)
