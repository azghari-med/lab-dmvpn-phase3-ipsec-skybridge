# Troubleshooting Checklist — SkyBridge DMVPN Phase 3 + IPsec Lab

## ⚡ Quick Diagnostic Order

Always follow this order — do not skip steps.
Each layer depends on the one below it.

```
Layer 1: ISP reachability (SW1)        → ping 10.10.0.x
Layer 2: IKEv2 / IPsec session         → show crypto session
Layer 3: Tunnel interface state         → show interface tunnel0
Layer 4: NHRP registration             → show ip nhrp
Layer 5: OSPF neighbors                → show ip ospf neighbor
Layer 6: Routing table                  → show ip route ospf
Layer 7: Phase 3 shortcuts             → show ip route (look for /32)
Layer 8: End-to-end spoke-to-spoke     → traceroute
```

\---

## Layer 1 — ISP Simulation (SW1)

```
FRA-HQ# ping 10.10.0.21   ← BCN-R1
FRA-HQ# ping 10.10.0.22   ← DXB-R2
FRA-HQ# ping 10.10.0.23   ← NYC-R3
FRA-HQ# ping 10.10.0.24   ← SGP-R4
```

ALL must succeed before continuing.
If any fails: check cable to SW1, check Eth0/0 IP, check no shutdown.

\---

## Layer 2 — IKEv2 / IPsec

```
show crypto session
show crypto ikev2 sa
show crypto ipsec sa
```

Expected: Session status: UP-ACTIVE on all peers

If DOWN:
\[ ] IKEv2 proposal configured: aes-cbc-256 / sha256 / group 14
\[ ] IKEv2 keyring peer ANY with pre-shared-key sky@ipsec2024
\[ ] IKEv2 profile: match identity remote address 0.0.0.0
\[ ] IPsec transform-set: esp-aes 256 esp-sha256-hmac mode transport
\[ ] IPsec profile: set transform-set + set ikev2-profile
\[ ] tunnel protection ipsec profile SKYBRIDGE-IPSEC applied on Tunnel0
Use: debug crypto ikev2 → look for INVALID\_SYNTAX or AUTH\_FAILED

\---

## Layer 3 — Tunnel Interface

```
show interface tunnel0
show ip interface tunnel0
```

Must be: up / up
If down:
\[ ] tunnel source Ethernet0/0 (not loopback or wrong interface)
\[ ] tunnel mode gre multipoint
\[ ] ip address assigned
\[ ] Eth0/0 is up/up

\---

## Layer 4 — NHRP Registration

```
show ip nhrp
show ip nhrp nhs detail
show ip nhrp summary
```

On FRA-HQ: must see all 4 spokes registered
On spokes: must see FRA-HQ as NHS with status UP

If spokes not registering:
\[ ] NHRP password: sky@bridge55 (same on all)
\[ ] network-id: 300 (same on all)
\[ ] ip nhrp nhs 10.77.0.1 (hub TUNNEL IP)
\[ ] ip nhrp map 10.77.0.1 10.10.0.1 (tunnel then WAN IP)
\[ ] ip nhrp map multicast 10.10.0.1
Use: debug nhrp authentication → finds password mismatch

\---

## Layer 5 — OSPF Neighbors

```
show ip ospf neighbor
show ip ospf interface tunnel0
```

On FRA-HQ: must see all 4 spokes in FULL state
On spokes: must see FRA-HQ in FULL state

If neighbors not forming or stuck in INIT/2WAY:
\[ ] ip ospf network point-to-multipoint on ALL tunnel interfaces
\[ ] OSPF process 1 area 0 on all tunnel interfaces
\[ ] Hello timer 10 / Dead timer 40 must match on all routers
\[ ] OSPF authentication: same key-id (1) and password (ospf@sky99)
\[ ] router-id configured (prevents ID conflict)
Use: debug ip ospf adj → shows exactly why adjacency fails

\---

## Layer 6 — Routing Table

```
show ip route ospf
show ip route 172.30.0.0
show ip route 172.34.0.0
```

Every router must see all LAN subnets (or summary if Task 8 done)

If routes missing:
\[ ] network statement covers correct subnet in router ospf 1
\[ ] OSPF neighbors are FULL (Layer 5 must pass first)
\[ ] no ip split-horizon ospf 1 on FRA-HQ Tunnel0
\[ ] Check passive-interface not set on Tunnel0

\---

## Layer 7 — Phase 3 Shortcuts

```
BCN-R1# show ip route
BCN-R1# show ip nhrp
```

After spoke-to-spoke traffic:
Routing table must have /32 host routes:
10.77.0.22/32 via 10.77.0.22  ← DXB-R2 shortcut
10.77.0.23/32 via 10.77.0.23  ← NYC-R3 shortcut

If /32 routes not appearing:
\[ ] ip nhrp redirect on FRA-HQ Tunnel0
\[ ] ip nhrp shortcut on ALL spoke Tunnel0 interfaces
\[ ] Both commands required — missing either one = Phase 3 broken
\[ ] Generate traffic first: ping or traceroute to trigger NHRP

\---

## Layer 8 — End-to-End Spoke-to-Spoke

```
BCN-R1# traceroute 172.34.0.1 source ethernet0/1
```

Expected: 1 hop directly to 10.77.0.24 (SGP-R4 tunnel IP)
If 2 hops: Phase 3 shortcuts not formed (go back to Layer 7)
If no path: routing table issue (go back to Layer 6)

\---

## 🔑 Most Common Mistakes in This Lab

|Mistake|Symptom|Fix|
|-|-|-|
|IKEv2 key mismatch|Crypto session DOWN|Verify sky@ipsec2024 in keyring on all routers|
|OSPF network type broadcast|No OSPF neighbors|ip ospf network point-to-multipoint on all tunnels|
|Missing ip nhrp redirect on hub|Phase 3 never activates|Add to FRA-HQ Tunnel0|
|Missing ip nhrp shortcut on spoke|Phase 3 never activates|Add to all spoke Tunnel0|
|OSPF auth key mismatch|Neighbors drop after auth|Verify key ID 1 + ospf@sky99 on all tunnels|
|IPsec before tunnel mode|Crypto session DOWN|Set tunnel mode gre multipoint BEFORE tunnel protection|
|Wrong OSPF timer|Neighbors flapping|hello 10 / dead 40 must match on all tunnel interfaces|
||||



## ⚡ Debug Commands (always run undebug all after)

```
debug crypto ikev2          ← IKEv2 negotiation issues
debug crypto ipsec          ← IPsec SA issues
debug nhrp                  ← NHRP registration
debug nhrp authentication   ← NHRP password problems
debug dmvpn all             ← full DMVPN events
debug ip ospf adj           ← OSPF neighbor formation
debug ip ospf hello         ← hello packet issues
debug tunnel                ← tunnel key mismatch
undebug all                 ← ALWAYS run this when done
```

