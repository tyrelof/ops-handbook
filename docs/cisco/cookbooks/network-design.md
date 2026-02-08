# Cookbook: Network Design & Cisco Networking

> [!NOTE]
> Enterprise network engineering and design patterns.
> Covers: Network architecture, Cisco IOS/NX-OS, routing, switching, security, VPN, and QoS.

## 1. Network Design Fundamentals

### Hierarchical Network Architecture
```
                    ┌─────────────────┐
                    │   Internet/ISP  │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
        ┌───▼──┐         ┌───▼──┐        ┌───▼──┐
        │ Core │         │Core  │        │Core  │
        │ FW   │         │FW    │        │FW    │
        └───┬──┘         └───┬──┘        └───┬──┘
            │                │                │
    ┌───────┴────────────────┼────────────────┴────────┐
    │                        │                         │
 ┌──▼──┐              ┌──────▼─────┐          ┌───────▼─┐
 │ Dist│              │  Dist SW   │          │ Dist SW │
 │ SW  │              │  (Core)    │          │ (Core)  │
 └──┬──┘              └──────┬─────┘          └───────┬─┘
    │                        │                        │
    │         ┌──────────────┼──────────────┐         │
    │         │              │              │         │
 ┌──▼────┐ ┌──▼────┐  ┌──────▼──┐  ┌────────▼──┐ ┌───▼────┐
 │Access │ │Access │  │ Access  │  │ Access    │ │Access  │
 │SW 1   │ │SW 2   │  │ SW 3    │  │ SW 4      │ │SW 5    │
 └───┬───┘ └───┬───┘  └───┬─────┘  └────┬─────┘ └───┬────┘
     │         │          │             │           │
  Hosts    Hosts     Servers       Servers        Hosts
```

### VLAN Strategy
```
VLAN 10:  Management        10.0.10.0/24
VLAN 20:  Users             10.0.20.0/24
VLAN 30:  Servers           10.0.30.0/24
VLAN 40:  VoIP              10.0.40.0/24
VLAN 50:  DMZ               10.0.50.0/24
VLAN 99:  Native            10.0.99.0/24 (unused / management plane)
```

### Redundancy Pattern
```
Primary Path:   Client → Core Switch 1 → Core Router 1 → ISP A
Secondary Path: Client → Core Switch 2 → Core Router 2 → ISP B
Tertiary Path:  Client → Access Switch → Dist Switch → Core
```

## 2. Cisco Switch Configuration (Access Layer)

### Basic Switch Setup
```cisco
configure terminal

! Hostname
hostname sw-access-01

! Management IP
interface vlan 10
  ip address 10.0.10.50 255.255.255.0
  no shutdown
exit

! Gateway
ip default-gateway 10.0.10.1

! DNS
ip domain-name example.com
ip name-server 8.8.8.8

! SNMP (monitoring)
snmp-server community public ro
snmp-server location "Building A"
snmp-server contact "Network Team"

! SSH (instead of Telnet)
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 4
  transport input ssh
### OSPF Design Example

Use OSPF area 0 for backbone and area 1 for distribution. Advertise only summarized networks to reduce RIB size.

```cisco
configure terminal

router ospf 1
 router-id 10.0.99.1
 network 10.0.0.0 0.0.255.255 area 0
 network 10.0.30.0 0.0.0.255 area 1
!
! Interface example
interface GigabitEthernet0/1
 ip address 10.0.254.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
exit

! Summarize at area border router
interface Loopback0
 ip address 10.0.0.254 255.255.255.255
exit

router ospf 1
 area 1 range 10.0.30.0 255.255.255.0
exit

end
write memory
```

### BGP Edge Example

For Internet edge use BGP with your public ASN. Keep internal networks as RFC1918 (10.x) and advertise only required prefixes.

```cisco
configure terminal

router bgp 65000
 bgp router-id 10.0.99.1
 neighbor 203.0.113.1 remote-as 65001
 neighbor 203.0.113.1 description ISP-A
 ! Accept only prefixes you expect
 ip prefix-list EXPORT-TO-ISP seq 5 permit 198.51.100.0/24
 ip prefix-list EXPORT-TO-ISP seq 10 permit 203.0.113.0/24
 route-map TO_ISP permit 10
  match ip address prefix-list EXPORT-TO-ISP
  set local-preference 100
 exit
 neighbor 203.0.113.1 route-map TO_ISP out
!
 address-family ipv4
  network 10.0.0.0 mask 255.255.0.0
 exit-address-family

end
write memory
```

### Cisco ASA Firewall Examples

Place firewalls at the edge and DMZ boundary. Keep a dedicated DMZ subnet (10.0.50.0/24).

Basic interface and NAT example:

```asa
! ASA interface config
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address <ISP-PUBLIC-1> 255.255.255.248
!
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.0.10.1 255.255.255.0
!
interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 10.0.50.1 255.255.255.0
!
! NAT (dynamic PAT) - translate inside hosts to outside IP
object network OBJ_INSIDE_NET
 subnet 10.0.20.0 255.255.255.0
 nat (inside,outside) dynamic interface
!
! Static NAT for DMZ web server
object network OBJ_WEB
 host 10.0.50.10
 nat (dmz,outside) static 198.51.100.10
!
! Access rule - allow HTTP from outside to web server
access-list OUTSIDE_IN extended permit tcp any host 198.51.100.10 eq 80
access-group OUTSIDE_IN in interface outside
```

High availability (active/standby) snippet:

```asa
! On primary ASA
failover
failover lan unit primary
failover lan interface failover GigabitEthernet0/3
failover interface ip failover 10.0.99.11 255.255.255.252 standby 10.0.99.12
!
! On standby ASA
failover lan unit secondary
failover lan interface failover GigabitEthernet0/3
failover interface ip failover 10.0.99.12 255.255.255.252 standby 10.0.99.11
```

### Huawei Quidway Notes

Huawei Quidway (older line) and modern Huawei VRP platforms use a CLI similar to Cisco but with different keywords. Key differences and tips:

- Interface naming: `interface GigabitEthernet0/0/1` vs Cisco `GigabitEthernet0/1`
- VLAN and SVI: `vlan 10` then `interface Vlanif10` with `ip address 10.0.10.1 255.255.255.0`
- OSPF: `ospf 1` then `area 0` configuration similar, but command syntax differs for area/type
- ACLs: use `acl number 2000` and `rule permit ip source ...` then apply with `traffic-filter inbound acl 2000 interface GigabitEthernet0/0/1`
- For migrations, map ACLs and NAT rules carefully, and validate behavior in lab first.

Example Huawei SVI and OSPF snippet:

```text
system-view
vlan 10
interface Vlanif10
 ip address 10.0.10.1 255.255.255.0
quit
ospf 1
 area 0
  network 10.0.0.0 0.255.255.255
quit
```

---

## 9. Troubleshooting & Verification

Key commands (Cisco):
- `show ip route`
- `show ip ospf neighbor`
- `show bgp summary`
- `show access-lists`
- `show interfaces status`
- `show running-config`

Use packet captures at aggregation points, and validate with synthetic traffic (iperf) when testing QoS and throughput.

---

*Enterprise network design additions: L2/L3 production addressing, OSPF/BGP edge, Cisco ASA examples, and Huawei notes.*
enable secret MySecurePassword

! Management interface
interface gigabitethernet 0/0
  ip address 10.0.99.1 255.255.255.0
  no shutdown
exit

! SNMP
snmp-server community public ro
snmp-server location "Data Center"

! SSH
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 4
  transport input ssh
  password VtyPassword
exit

end
write memory
```

### VLAN Routing (Router on a Stick)
```cisco
configure terminal

! Trunk interface to switch
interface gigabitethernet 0/0
  no shutdown
  no ip address
exit

! Create sub-interfaces for each VLAN
interface gigabitethernet 0/0.10
  encapsulation dot1Q 10
  ip address 192.168.10.1 255.255.255.0
exit

interface gigabitethernet 0/0.20
  encapsulation dot1Q 20
  ip address 192.168.20.1 255.255.255.0
exit

interface gigabitethernet 0/0.30
  encapsulation dot1Q 30
  ip address 192.168.30.1 255.255.255.0
exit

end
write memory

! Verify
show ip route
show ip interface brief
```

### Static Routing
```cisco
configure terminal

! Static route to remote network
ip route 10.0.0.0 255.255.255.0 192.168.99.254

! Default route (to ISP)
ip route 0.0.0.0 0.0.0.0 203.0.113.1

! Backup route with higher metric
ip route 10.0.0.0 255.255.255.0 192.168.99.253 10

end
write memory

! Verify
show ip route
show ip route static
```

### OSPF (Dynamic Routing)
```cisco
configure terminal

! Enable OSPF
router ospf 1
  router-id 192.168.99.1
  
  ! Advertise networks
  network 192.168.10.0 0.0.0.255 area 0
  network 192.168.20.0 0.0.0.255 area 0
  network 192.168.30.0 0.0.0.255 area 0
  network 192.168.99.0 0.0.0.255 area 0
  
  ! Passive interfaces (don't send OSPF)
  passive-interface gigabitethernet 0/1
  
  ! Redistribute static routes
  redistribute static subnets

end
write memory

! Verify
show ip ospf neighbor
show ip ospf database
show ip route ospf
```

## 4. Access Control Lists (ACLs)

### Standard ACL
```cisco
configure terminal

! Deny users from downloading files
access-list 100 deny tcp 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255 eq 21
access-list 100 deny tcp 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255 eq 22
access-list 100 permit ip any any

! Apply to interface
interface gigabitethernet 0/0.20
  ip access-group 100 out
exit

end
write memory

! Verify
show access-lists
show ip access-lists
```

### Named Extended ACL
```cisco
configure terminal

! Allow SSH only from admin network
ip access-list extended ADMIN-SSH
  permit tcp 192.168.10.0 0.0.0.255 any eq 22
  deny tcp any any eq 22
  permit ip any any
exit

! Apply
interface gigabitethernet 0/0
  ip access-group ADMIN-SSH in
exit

end
write memory
```

## 5. VPN Configuration

### Site-to-Site VPN (IPsec)
```cisco
! Router A (Site 1)
configure terminal

! IKE Phase 1 (authentication)
crypto isakmp policy 1
  authentication pre-share
  encryption aes 256
  hash sha512
  group 15
  lifetime 28800
exit

! Pre-shared key
crypto isakmp key MyVPNKey123 address 203.0.113.2

! IKE Phase 2 (encryption)
crypto ipsec transform-set TRANSFORMS esp-aes 256 esp-sha512-hmac
  mode tunnel
exit

! Crypto map (tie it together)
crypto map SITE-TO-SITE 1 ipsec-isakmp
  set peer 203.0.113.2
  set transform-set TRANSFORMS
  match address 100
exit

! ACL - traffic to encrypt
access-list 100 permit ip 192.168.0.0 0.0.255.255 10.0.0.0 0.0.255.255

! Apply to WAN interface
interface gigabitethernet 0/0
  crypto map SITE-TO-SITE
exit

end
write memory

! Verify
show crypto isakmp sa
show crypto ipsec sa
show crypto map
```

## 6. Quality of Service (QoS)

### Traffic Shaping
```cisco
configure terminal

! Define traffic classes
class-map VOICE
  match protocol rtp
exit

class-map VIDEO
  match dscp af31 af32 af33
exit

class-map DEFAULT
  match any
exit

! Define policy (bandwidth allocation)
policy-map BRANCH-POLICY
  class VOICE
    priority percent 30  ! 30% guaranteed
  exit
  
  class VIDEO
    bandwidth percent 40  ! 40% guaranteed
  exit
  
  class DEFAULT
    bandwidth percent 30  ! 30% shared
  exit
exit

! Apply to interface
interface serial 0/0/0  ! WAN link
  service-policy output BRANCH-POLICY
exit

end
write memory

! Verify
show policy-map
show policy-map interface serial 0/0/0
```

## 7. Network Monitoring & Troubleshooting

### Verification Commands
```cisco
! Check interface status
show interface
show interface gigabitethernet 0/0
show interface status

! Check IP routing
show ip route
show ip route summary
show ip route connected

! Check neighbor devices
show cdp neighbors
show cdp neighbors detail

! Check VLAN
show vlan
show vlan brief

! Check spanning tree
show spanning-tree
show spanning-tree vlan 20

! Check routing protocols
show ip ospf neighbor
show ip ospf database
show bgp summary

! Check ACL
show access-lists
show ip access-lists

! Check VPN
show crypto session
show crypto ipsec sa
```

### Diagnostic Commands
```cisco
! Ping (Layer 3 connectivity)
ping 192.168.30.1

! Traceroute
traceroute 10.0.0.1

! Debug OSPF
debug ip ospf events
debug ip ospf adjacency
undebug all  ! Turn off

! Check ARP table
show arp
show ip arp

! Check MAC address table
show mac-address-table
show mac-address-table dynamic

! Packet capture
monitor session 1 source interface gigabitethernet 0/0
monitor session 1 destination interface gigabitethernet 0/1
show monitor session 1
```

## 8. Network Device Management

### SNMP Monitoring
```cisco
configure terminal

! SNMP read-only community
snmp-server community public ro

! Trap destination (alerts to monitoring server)
snmp-server enable traps
snmp-server host 192.168.10.100 version 2c public

! Check SNMP
show snmp

! SNMP via command line (from management PC)
! snmpget -v 2c -c public 192.168.99.1 sysDescr.0
! snmpwalk -v 2c -c public 192.168.99.1 .1.3.6.1.2.1.1
```

### Backup & Restore
```cisco
! Backup config to TFTP server
copy running-config tftp://192.168.10.100/router-backup.cfg

! Restore from backup
copy tftp://192.168.10.100/router-backup.cfg running-config

! View running config
show running-config

! Compare running vs startup
show differences

! Save current config
write memory  ! Or "copy running-config startup-config"
```

## 9. High Availability

### HSRP (Hot Standby Routing Protocol)
```cisco
! Primary Router
configure terminal

interface gigabitethernet 0/0.20
  ip address 192.168.20.100 255.255.255.0
  
  ! HSRP configuration
  standby 1 ip 192.168.20.1  ! Virtual IP
  standby 1 priority 150  ! Higher = active
  standby 1 preempt  ! Take over if higher priority
  standby 1 authentication md5 key-string MyKey123
  
exit

end

! Secondary Router
configure terminal

interface gigabitethernet 0/0.20
  ip address 192.168.20.101 255.255.255.0
  
  standby 1 ip 192.168.20.1
  standby 1 priority 100  ! Lower = standby
  standby 1 preempt
  standby 1 authentication md5 key-string MyKey123
  
exit

end

! Verify
show standby
show standby summary
show standby brief
```

## 10. Network Security

### Firewall ACLs (Stateful)
```cisco
configure terminal

! Inbound - block unauthorized traffic
access-list 101 permit tcp 192.168.30.0 0.0.0.255 192.168.50.0 0.0.0.255 eq 443  ! HTTPS
access-list 101 permit tcp 192.168.30.0 0.0.0.255 192.168.50.0 0.0.0.255 eq 80   ! HTTP
access-list 101 deny ip any any
access-list 101 permit ip 192.168.0.0 0.0.255.255 any  ! Internal traffic

interface gigabitethernet 0/0
  ip access-group 101 in
exit

end
write memory
```

### DDoS Mitigation
```cisco
configure terminal

! Rate limiting per source
traffic-shape rate 1000000  ! 1 Mbps limit

! Connection limits
ip tcp synwait-time 5
ip tcp connection-timeout 600

! TCP intercept (SYN flood protection)
ip tcp intercept list 110
ip tcp intercept mode intercept
ip tcp intercept watch-timeout 15
ip tcp intercept connection-timeout 300

access-list 110 permit tcp any any eq 80
access-list 110 permit tcp any any eq 443

end
write memory
```

## 11. Network Design Checklist

| Component | Consideration | Implementation |
|-----------|---------------|-----------------|
| **Redundancy** | No single point of failure | Dual ISP, dual core, HSRP |
| **Bandwidth** | Sufficient for growth | Plan for 3x current usage |
| **Latency** | <50ms for VoIP | Direct paths, avoid congestion |
| **Security** | Perimeter & internal | ACLs, DMZ, VPN |
| **Scalability** | Room to grow | Modular design, OSPF |
| **Monitoring** | Early issue detection | SNMP, syslog, NetFlow |
| **Backup links** | ISP failure recovery | Dual WAN connections |
| **Segmentation** | Contain breaches | VLANs, access lists |

## 12. Common Scenarios

### New Branch Office Connection
```cisco
! 1. Create VLANs for branch
vlan 100
  name Branch-Users
vlan 101
  name Branch-Servers
exit

! 2. Configure VPN tunnel to branch router
! (See VPN section above)

! 3. Add static routes to branch subnets
ip route 10.0.0.0 255.255.0.0 203.0.113.100

! 4. Create ACLs for traffic filtering

! 5. Configure QoS for branch users
```

### Network Upgrade (IOS Update)
```cisco
! 1. Backup current config
copy running-config tftp://192.168.10.100/backup.cfg

! 2. Download new IOS to flash
copy tftp://192.168.10.100/ios-image.bin flash:

! 3. Verify image
show flash:
verify flash:/ios-image.bin

! 4. Update boot statement
configure terminal
boot system flash:/ios-image.bin
end

! 5. Schedule reload
reload at 02:00  ! 2 AM

! Or reload now (carefully!)
reload
```

---

**Essential Commands Summary**:

| Task | Command |
|------|---------|
| Show interfaces | `show interface` |
| Show routes | `show ip route` |
| Show OSPF | `show ip ospf neighbor` |
| Show VLANs | `show vlan` |
| Show ACL | `show access-lists` |
| Ping | `ping <ip>` |
| Traceroute | `traceroute <ip>` |
| Save config | `write memory` |
| Reload device | `reload` |

**This foundation covers enterprise network design and management.**
