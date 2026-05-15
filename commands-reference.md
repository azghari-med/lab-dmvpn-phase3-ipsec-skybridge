# Commands Reference — SkyBridge DMVPN Phase 3 + IPsec Lab

## 📋 IP Addressing Table

|Device|Role|Eth0/0 WAN|Eth0/1 LAN|Tunnel0 IP|OSPF Router-ID|
|-|-|-|-|-|-|
|FRA-HQ|Hub|10.10.0.1/24|172.30.0.1/24|10.77.0.1/24|1.1.1.1|
|BCN-R1|Spoke|10.10.0.21/24|172.31.0.1/24|10.77.0.21/24|2.2.2.2|
|DXB-R2|Spoke|10.10.0.22/24|172.32.0.1/24|10.77.0.22/24|3.3.3.3|
|NYC-R3|Spoke|10.10.0.23/24|172.33.0.1/24|10.77.0.23/24|4.4.4.4|
|SGP-R4|Spoke|10.10.0.24/24|172.34.0.1/24|10.77.0.24/24|5.5.5.5|

\---

## 🔑 Phase 3 Key Commands — Know These Cold

```
Phase 3 on HUB:
  ip nhrp redirect          ← tells spokes: use a shortcut next time

Phase 3 on SPOKE:
  ip nhrp shortcut          ← installs /32 host route when hub redirects

BOTH are required. Missing one = Phase 3 broken silently.

Proof of Phase 3 working:
  show ip route → look for /32 host routes (10.77.0.xx/32)
  These /32 entries ARE the Phase 3 shortcuts
```

\---

## 🔒 IKEv2 Show Commands

```
show crypto ikev2 sa
show crypto ikev2 sa detail
show crypto ikev2 stats
show crypto session
show crypto session detail
show crypto ipsec sa
show crypto ipsec sa detail
show crypto ipsec transform-set
show crypto ikev2 proposal
show crypto ikev2 policy
show crypto ikev2 profile
show crypto ikev2 keyring
```

\---

## 🌐 DMVPN / NHRP Show Commands

```
show dmvpn
show dmvpn detail
show ip nhrp
show ip nhrp nhs
show ip nhrp nhs detail
show ip nhrp summary
show ip nhrp shortcut
show interface tunnel0
```

\---

## 🔀 OSPF Show Commands

```
show ip ospf neighbor
show ip ospf neighbor detail
show ip ospf interface tunnel0
show ip ospf database
show ip ospf database summary
show ip ospf
```

\---

## 🗺️ Routing Show Commands

```
show ip route
show ip route ospf
show ip route 10.77.0.0
show ip route 172.30.0.0
show ip protocols
```

\---

## 🎯 Verification One-Liners

|What to verify|Command|
|-|-|
|All spokes registered|FRA-HQ# show dmvpn|
|IPsec encrypting|show crypto ipsec sa (encaps/decaps > 0)|
|IKEv2 session active|show crypto ikev2 sa|
|OSPF neighbors FULL|show ip ospf neighbor|
|Phase 3 shortcuts installed|show ip route (look for /32 host routes)|
|OSPF auth working|show ip ospf interface tunnel0|
|DPD configured|show crypto ikev2 profile|
|Spoke-to-spoke direct|traceroute \[spoke-LAN] source eth0/1|

\---

## 📝 Spoke Config Template

```
! Change TUNNEL-IP per spoke:
! BCN-R1 = 10.77.0.21 | DXB-R2 = 10.77.0.22
! NYC-R3 = 10.77.0.23 | SGP-R4 = 10.77.0.24

! IKEv2 (same on all routers)
crypto ikev2 proposal SKYBRIDGE-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy SKYBRIDGE-POL
 proposal SKYBRIDGE-PROP

crypto ikev2 keyring SKYBRIDGE-KEYS
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key sky@ipsec2024

crypto ikev2 profile SKYBRIDGE-IKE
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local SKYBRIDGE-KEYS
 dpd 30 5 periodic

crypto ipsec transform-set SKYBRIDGE-TS esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile SKYBRIDGE-IPSEC
 set transform-set SKYBRIDGE-TS
 set ikev2-profile SKYBRIDGE-IKE

! Interfaces
interface Ethernet0/0
 ip address \[WAN-IP] 255.255.255.0
 no shutdown

interface Ethernet0/1
 ip address \[LAN-IP] 255.255.255.0
 no shutdown

interface Tunnel0
 ip address \[TUNNEL-IP] 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication sky@b

&#x20;ip nhrp network-id 300
 ip nhrp holdtime 300
 ip nhrp nhs 10.77.0.1
 ip nhrp map 10.77.0.1 10.10.0.1
 ip nhrp map multicast 10.10.0.1
 ip nhrp shortcut
 ip ospf network point-to-multipoint
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 ospf@sky99
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 300
 tunnel protection ipsec profile SKYBRIDGE-IPSEC

! OSPF
router ospf 1
 router-id \[X.X.X.X]
 network 10.77.0.0 0.0.0.255 area 0
 network \[LAN-NETWORK] 0.0.0.255 area 0
```

\---

## 📊 Phase Comparison — Quick Reference

|Feature|Phase 1|Phase 2|Phase 3|
|-|-|-|-|
|Spoke-to-Spoke|❌ via hub|✅ direct|✅ direct|
|Hub command|—|—|ip nhrp redirect|
|Spoke command|—|—|ip nhrp shortcut|
|Routing proof|normal routes|/32 via peer|/32 host route|
|Works with OSPF|✅|⚠️ limited|✅|
|Works with EIGRP|✅|✅|✅|
|Scales better|❌|medium|✅ best|



