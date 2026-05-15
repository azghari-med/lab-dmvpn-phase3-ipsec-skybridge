# Lab: DMVPN Phase 3 — SkyBridge Logistics Global WAN + IPsec + OSPF

## 🏢 Scenario

**Company:** SkyBridge Logistics
**Situation:** SkyBridge operates a global logistics network with one
headquarters in Frankfurt and four regional offices in Barcelona, Dubai,
New York, and Singapore. The previous DMVPN Phase 2 design caused
suboptimal routing when spokes communicated — traffic was still passing
through HQ before reaching destination spokes.

The network team must upgrade to **DMVPN Phase 3** so routing is fully
optimized with NHRP shortcuts. The security team requires **IKEv2 IPsec**
with Dead Peer Detection on all tunnels. OSPF is chosen as the routing
protocol because the company plans to add more sites and OSPF scales
better than EIGRP for their growth plan.

The IPsec crypto policies and IKEv2 profiles are pre-configured.
OSPF process 1 is pre-configured on FRA-HQ only.

\---

🗺️ Topology Diagram
```
  172.31.0.0/24    172.32.0.0/24    172.33.0.0/24    172.34.0.0/24
  \[BCN-R1 LAN]     \[DXB-R2 LAN]     \[NYC-R3 LAN]     \[SGP-R4 LAN]
       |                |                |                |
    Eth0/1           Eth0/1           Eth0/1           Eth0/1
   \[BCN-R1]         \[DXB-R2]         \[NYC-R3]         \[SGP-R4]
 T:10.77.0.21     T:10.77.0.22     T:10.77.0.23     T:10.77.0.24
    Eth0/0           Eth0/0           Eth0/0           Eth0/0
   10.10.0.21       10.10.0.22       10.10.0.23       10.10.0.24
       \\               |               |               /
        \\              |               |              /
         \\             |               |             /
          +----------\[SW1 — ISP Simulated]----------+
                              |
                          10.10.0.1
                          Eth0/0
                         \[FRA-HQ]
                        T:10.77.0.1
                          Eth0/1
                      172.30.0.0/24
                      \[FRA-HQ LAN]

\---

## 📋 IP Addressing Table

|Device|Role|Eth0/0 WAN|Eth0/1 LAN|Tunnel0 IP|
|-|-|-|-|-|
|FRA-HQ|Hub|10.10.0.1/24|172.30.0.1/24|10.77.0.1/24|
|BCN-R1|Spoke|10.10.0.21/24|172.31.0.1/24|10.77.0.21/24|
|DXB-R2|Spoke|10.10.0.22/24|172.32.0.1/24|10.77.0.22/24|
|NYC-R3|Spoke|10.10.0.23/24|172.33.0.1/24|10.77.0.23/24|
|SGP-R4|Spoke|10.10.0.24/24|172.34.0.1/24|10.77.0.24/24|
|SW1|ISP|all routers connect Eth0/0 here|||

> SW1 is a Layer 2 switch simulating the internet.
> All WAN interfaces must be in 10.10.0.0/24 so SW1 can forward between them.

\---

## 🔐 NHRP \& Tunnel Parameters

|Parameter|Value|
|-|-|
|NHRP Password|sky@b|
|Network-ID|300|
|Tunnel Key|300|
|Hold Time|300 sec (5 min)|
|Tunnel Subnet|10.77.0.0/24|
|Tunnel Mode|GRE Multipoint|
|IPsec Profile|SKYBRIDGE-IPSEC|
|OSPF Process|1|
|OSPF Area|0|

\---

## ⚙️ PNET Build Steps

```
Step 1 — Add devices:
  5x Cisco Router (IOSv or IOS 15.x)
  1x Ethernet Switch
  Rename: FRA-HQ | BCN-R1 | DXB-R2 | NYC-R3 | SGP-R4 | SW1

Step 2 — Connect cables:
  FRA-HQ Eth0/0 ──► SW1 port 0
  BCN-R1 Eth0/0 ──► SW1 port 1
  DXB-R2 Eth0/0 ──► SW1 port 2
  NYC-R3 Eth0/0 ──► SW1 port 3
  SGP-R4 Eth0/0 ──► SW1 port 4

Step 3 — Configure WAN interfaces:
  FRA-HQ: ip address 10.10.0.1 255.255.255.0
  BCN-R1: ip address 10.10.0.21 255.255.255.0
  DXB-R2: ip address 10.10.0.22 255.255.255.0
  NYC-R3: ip address 10.10.0.23 255.255.255.0
  SGP-R4: ip address 10.10.0.24 255.255.255.0

Step 4 — Verify ISP simulation BEFORE building DMVPN:
  FRA-HQ# ping 10.10.0.21  ✅ BCN-R1
  FRA-HQ# ping 10.10.0.22  ✅ DXB-R2
  FRA-HQ# ping 10.10.0.23  ✅ NYC-R3
  FRA-HQ# ping 10.10.0.24  ✅ SGP-R4

Step 5 — Configure LAN interfaces (Eth0/1 on each router)

Step 6 — Configure IPsec (Task 1) BEFORE tunnel interfaces
         This is critical — IPsec must exist before tunnel protection

Step 7 — Build DMVPN Phase 3 (Tasks 2–5)

Step 8 — Configure OSPF (Task 6)

Step 9 — Extra challenges (Tasks 7–9)
```

\---

## 🎯 Tasks

\---

### Task 1 — ⭐ Configure IKEv2 IPsec on ALL Routers

This is a real-world challenge — IKEv2 is the modern standard.
Most engineers only know IKEv1. This sets you apart.

```
! Step 1 — IKEv2 Proposal (encryption + integrity)
crypto ikev2 proposal SKYBRIDGE-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

! Step 2 — IKEv2 Policy (matches the proposal)
crypto ikev2 policy SKYBRIDGE-POL
 proposal SKYBRIDGE-PROP

! Step 3 — IKEv2 Keyring (pre-shared key for all peers)
crypto ikev2 keyring SKYBRIDGE-KEYS
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key sky@ipsec2024

! Step 4 — IKEv2 Profile (ties keyring to authentication)
crypto ikev2 profile SKYBRIDGE-IKE
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local SKYBRIDGE-KEYS
 dpd 30 5 periodic

! Step 5 — IPsec Transform Set
crypto ipsec transform-set SKYBRIDGE-TS esp-aes 256 esp-sha256-hmac
 mode transport

! Step 6 — IPsec Profile (applied to tunnel interface)
crypto ipsec profile SKYBRIDGE-IPSEC
 set transform-set SKYBRIDGE-TS
 set ikev2-profile SKYBRIDGE-IKE
```

> Apply this config on ALL 5 routers before building tunnels.
> DPD (Dead Peer Detection) pings peer every 30 seconds,
> removes dead session after 5 retries — critical for DMVPN stability.

\---

### Task 2 — Configure FRA-HQ as DMVPN Phase 3 Hub

```
interface Tunnel0
 ip address 10.77.0.1 255.255.255.0

\\\&#x20;ip nhrp holdtime 300 


 ip nhrp authentication sky@b
 ip nhrp network-id 300
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp map multicast dynamic
 ip nhrp redirect
 ip ospf network point-to-multipoint
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 300
 tunnel protection ipsec profile SKYBRIDGE-IPSEC
```

> ip nhrp redirect → the Phase 3 magic on the hub
> Tells spokes: "I know a shorter path — go direct next time"
> ip ospf network point-to-multipoint → fixes OSPF DR/BDR issue on DMVPN

\---

### Task 3 — Configure BCN-R1 (Barcelona Spoke)

```
interface Tunnel0
 ip address 10.77.0.21 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication sky@b

\\\&#x20;ip nhrp network-id 300
 ip nhrp holdtime 300
 ip nhrp nhs 10.77.0.1
 ip nhrp map 10.77.0.1 10.10.0.1
 ip nhrp map multicast 10.10.0.1
 ip nhrp shortcut
 ip ospf network point-to-multipoint
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 300
 tunnel protection ipsec profile SKYBRIDGE-IPSEC
```

> ip nhrp shortcut → the Phase 3 magic on the spoke
> Installs a host route directly to remote spoke after NHRP resolution
> This is what makes Phase 3 different from Phase 2

\---

### Task 4 — Configure DXB-R2, NYC-R3, SGP-R4

Same config as BCN-R1 — only change the tunnel IP:

```
DXB-R2 → ip address 10.77.0.22 255.255.255.0
NYC-R3 → ip address 10.77.0.23 255.255.255.0
SGP-R4 → ip address 10.77.0.24 255.255.255.0
```

\---

### Task 5 — Configure OSPF on All Routers

```
! FRA-HQ
router ospf 1
 router-id 1.1.1.1
 network 10.77.0.0 0.0.0.255 area 0
 network 172.30.0.0 0.0.0.255 area 0

! BCN-R1
router ospf 1
 router-id 2.2.2.2
 network 10.77.0.0 0.0.0.255 area 0
 network 172.31.0.0 0.0.0.255 area 0

! DXB-R2
router ospf 1
 router-id 3.3.3.3
 network 10.77.0.0 0.0.0.255 area 0
 network 172.32.0.0 0.0.0.255 area 0

! NYC-R3
router ospf 1
 router-id 4.4.4.4
 network 10.77.0.0 0.0.0.255 area 0
 network 172.33.0.0 0.0.0.255 area 0

! SGP-R4
router ospf 1
 router-id 5.5.5.5
 network 10.77.0.0 0.0.0.255 area 0
 network 172.34.0.0 0.0.0.255 area 0
```

\---

### Task 6 — ⭐ Extra Challenge: OSPF Authentication on Tunnel

Real networks always authenticate OSPF — prevents rogue routers joining.

```
! On ALL routers tunnel interface
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 ospf@sky99

! Verify:
show ip ospf interface tunnel0
→ Message digest authentication enabled
→ Key ID 1
```

\---

### Task 7 — ⭐ Extra Challenge: IP SLA Tunnel Health Monitor

Simulate what happens when the underlay fails — real-world critical skill.

```
! On BCN-R1 — monitor tunnel to FRA-HQ
ip sla 1
 icmp-echo 10.77.0.1 source-interface Tunnel0
 frequency 10
ip sla schedule 1 life forever start-time now

! Track the SLA
track 1 ip sla 1 reachability

! Verify
show ip sla statistics
show track 1
```

### Task 8 — ⭐ Extra Challenge: Phase 3 vs Phase 2 Comparison

Document the difference between Phase 2 and Phase 3 behavior:

```
! Step 1 — Remove ip nhrp shortcut from BCN-R1 (simulates Phase 2)
interface tunnel0
 no ip nhrp shortcut

! Step 2 — Traceroute BCN-R1 to SGP-R4
BCN-R1# traceroute 172.34.0.1 source ethernet0/1
→ Should show 2 hops via FRA-HQ (Phase 2 behavior)

! Step 3 — Add ip nhrp shortcut back (Phase 3)
interface tunnel0
 ip nhrp shortcut

! Step 4 — Traceroute again after NHRP resolves
BCN-R1# traceroute 172.34.0.1 source ethernet0/1
→ Should show 1 hop direct (Phase 3 behavior)

! Document the difference — this is gold in an interview
```

\---

### Task 10 — Final Verification

```
! All 4 spokes registered on hub
FRA-HQ# show dmvpn detail

! OSPF neighbors on hub — all 4 spokes
FRA-HQ# show ip ospf neighbor

! IPsec sessions active
FRA-HQ# show crypto session

! Spoke-to-spoke direct (1 hop)
BCN-R1# traceroute 172.34.0.1 source ethernet0/1
→ 1 hop to SGP-R4 ✅

! NHRP shortcut entry on spoke
BCN-R1# show ip route
→ 10.77.0.24/32 via 10.77.0.24  ← /32 host route = Phase 3 shortcut ✅

! OSPF authentication working
BCN-R1# show ip ospf interface tunnel0
→ Message digest authentication enabled ✅
```

\---

## ✅ Expected Verification Output

```
FRA-HQ# show dmvpn detail
Ident            NBMA Addr    State
10.77.0.21/32    10.10.0.21   UP  ← BCN-R1 ✅
10.77.0.22/32    10.10.0.22   UP  ← DXB-R2 ✅
10.77.0.23/32    10.10.0.23   UP  ← NYC-R3 ✅
10.77.0.24/32    10.10.0.24   UP  ← SGP-R4 ✅

FRA-HQ# show crypto session
Peer: 10.10.0.21  Status: UP-ACTIVE  ← BCN-R1
Peer: 10.10.0.22  Status: UP-ACTIVE  ← DXB-R2
Peer: 10.10.0.23  Status: UP-ACTIVE  ← NYC-R3
Peer: 10.10.0.24  Status: UP-ACTIVE  ← SGP-R4

BCN-R1# show ip route
O    172.30.0.0/20 \\\\\\\[110/1001] via 10.77.0.1   ← summary from HQ
O    172.32.0.0/24 \\\\\\\[110/1001] via 10.77.0.22  ← DXB-R2 direct
O    172.33.0.0/24 \\\\\\\[110/1001] via 10.77.0.23  ← NYC-R3 direct
O    172.34.0.0/24 \\\\\\\[110/1001] via 10.77.0.24  ← SGP-R4 direct
     10.77.0.24/32 via 10.77.0.24             ← PHASE 3 SHORTCUT ✅

BCN-R1# traceroute 172.34.0.1 source ethernet0/1
  1  10.77.0.24  \\\\\\\[DIRECT — 1 hop ✅]
```

\---

## 🐛 Troubleshooting Log

### Problem 1 — IKEv2 Session Not Establishing

```
Symptom : show crypto session → DOWN
Cause   : Pre-shared key mismatch between routers
Debug   : debug crypto ikev2
Fix     : Verified sky@ipsec2024 on all routers in keyring config
Lesson  : IKEv2 keyring pre-shared-key must match exactly on both peers
```

### Problem 2 — OSPF Neighbors Not Forming Over Tunnel

```
Symptom : show ip ospf neighbor → empty
Cause   : OSPF network type mismatch
          Hub: point-to-multipoint / Spoke: broadcast (default)
Debug   : show ip ospf interface tunnel0 (check network type)
Fix     : ip ospf network point-to-multipoint on ALL tunnel interfaces
Lesson  : Default OSPF type on GRE is broadcast — causes DR/BDR election
          which breaks on DMVPN. point-to-multipoint has no DR/BDR.
```

### Problem 3 — Phase 3 Shortcuts Not Forming

```
Symptom : traceroute still shows 2 hops via FRA-HQ after Phase 3 config
Cause   : ip nhrp redirect missing on hub OR ip nhrp shortcut missing on spoke
Fix     : FRA-HQ  → ip nhrp redirect on Tunnel0
          BCN-R1  → ip nhrp shortcut on Tunnel0
          Both commands required — Phase 3 breaks without either one
```

### Problem 4 — OSPF Authentication Failure

```
Symptom : OSPF neighbors drop after adding authentication
Cause   : MD5 key mismatch — key configured on hub but not all spokes
Debug   : debug ip ospf adj
Fix     : Verified key ID 1 and password ospf@sky99 on all tunnel interfaces
Lesson  : OSPF authentication must match: same type, same key-id, same password
```

### Problem 5 — DPD Removing Active Tunnels

```
Symptom : Crypto sessions dropping every 30-35 seconds
Cause   : DPD configured but ICMP blocked somewhere in path
Fix     : Verified ICMP reachability on underlay (ping 10.10.0.x)
          DPD uses IKE keepalives not ICMP — actual fix was
          ensuring IKE UDP 500 and 4500 not blocked
```

```

\\\\---

## 📚 Key Concepts Learned

### Phase 3 vs Phase 2 — The Real Difference

```

Phase 2:
Spoke installs /32 host route via dynamic NHRP
Uses same routing table entry → direct tunnel forms
Problem: routing protocol must support this (EIGRP yes, OSPF no)

Phase 3:
Hub sends NHRP redirect to source spoke
Spoke installs /32 host route (shortcut) in routing table
Bypasses routing protocol completely → works with OSPF
ip nhrp redirect on hub + ip nhrp shortcut on spoke = Phase 3

```

### IKEv2 vs IKEv1

```

IKEv1: 2 phases, 6-9 messages, older standard
IKEv2: single exchange, 4 messages, faster, more secure
Built-in DPD, EAP support, better NAT traversal
Always use IKEv2 in modern deployments

```

### OSPF Network Types on DMVPN

```

broadcast (default on GRE):
Elects DR/BDR → breaks on DMVPN hub-and-spoke
All spokes try to form adjacency with DR not hub

point-to-multipoint (correct for DMVPN):
No DR/BDR election
Each router forms individual adjacency with each neighbor
Works perfectly on DMVPN — always use this

```

### DPD — Dead Peer Detection

```

dpd 30 5 periodic means:
Send keepalive every 30 seconds
If no response after 5 retries → declare peer dead
Remove IKE SA and IPsec SA
DMVPN will re-establish tunnel automatically
Critical for detecting failed WAN links quickly

```

\\\\---

## 🧠 Interview Questions This Lab Covers

\\\* What is the difference between DMVPN Phase 2 and Phase 3?
\\\* What two commands enable Phase 3 and on which routers?
\\\* Why use OSPF point-to-multipoint instead of broadcast on DMVPN?
\\\* What is Dead Peer Detection and why is it important?
\\\* What is the difference between IKEv1 and IKEv2?
\\\* How does ip nhrp shortcut differ from dynamic NHRP in Phase 2?
\\\* Why can Phase 3 work with OSPF but Phase 2 struggles?
\\\* What does the /32 host route in the routing table indicate in Phase 3?
\\\* How do you authenticate OSPF neighbors and why is it important?
\\\* How would you verify IPsec is actually encrypting traffic?



