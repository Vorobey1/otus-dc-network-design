# Построение Overlay на основе VXLAN EVPN c L3 VNI
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить eBGP для Underlay
4. Настроить VXLAN EVPN
5. Конфигурация АСО
6. Вывод show commands (show bgp evpn route-type ip-prefix ipv4, show vxlan flood vtep, show ip route vrf)
7. Тестирование L3 связности между клиентскими сетями (ping)
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab6/screenshots/Topology.PNG)
## Адресное пространство для сети
Адресацию для АСО я переиспользовал из работы "[Проектирование адресного пространства](https://github.com/Vorobey1/otus-dc-network-design/edit/main/lab1/README.md)".

|Device |Loopback    |
|:-----:|:----------:|
|Spine1 |10.0.1.0/32 |
|Spine2 |10.0.2.0/32 |
|Leaf1  |10.0.0.1/32 |
|Leaf2  |10.0.0.2/32 |
|Leaf3  |10.0.0.3/32 |

|p2p         |Spine1      |Spine2      |
|:----------:|:----------:|:----------:|
|Leaf1       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf2       |10.2.1.2/31 |10.2.2.2/31 |
|Leaf3       |10.2.1.4/31 |10.2.2.4/31 |

|Device  |Ip-address  |
|:------:|:----------:|
|Client1 |10.4.0.1/24 |
|Client2 |10.5.0.1/24 |
|Client3 |10.4.0.2/24 |
|Client4 |10.5.0.2/24 |

## Настройка eBGP для Underlay сети
При настройке eBGP в топологиях CLOS можно руководствоваться следующим рекомендациям:
1. Использовать схему ASN, которая исключает проблему path-hunting
2. Использовать multipath для ECMP
3. Использовать MP-BGP
4. Использовать BFD, изменить стандартные таймера, настроить аутентификацию

При настройке eBGP для Spine я использовал приватную AS - 64086.60000, для Leaf - 64086.60Ln, где Ln - номер Leaf в диапазоне [001...999].  
Для упрощения конфигурации использовал peer groups. На Spine для автообнаружения соседей использовал bgp listen 

## Настройка VXLAN EVPN c L3VNI
Настраиваем VLAN 10,20 на всех Leaf (VTEP)
```
vlan 10
   name SERVICE-1
vlan 20
   name SERVICE-2
```
Создаем NVE (туннельные интерфейс для инкапсуляции/декапсуляции фреймов) на VTEP с использованием ipv6 и свяжем VLAN c VNI
```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
```
В BGP на Leaf и Spine добавляем возможность пересылки extended community и активируем AF EVPN
```
router bgp ASN
neighbor NEIGHBOR send-community extended
   address-family evpn
      neighbor NEIGHBOR activate
```
На VTEP создаем EVPN Instance (EVI): указываем RD, RT, включаем анонс MAC-адресов
```
router bgp ASN
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
```

## Конфигурация АСО
**Spine1**
```
!
service routing protocols model multi-agent
!
hostname Spine1
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:101/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:103/127
!
interface Ethernet3
   no switchport
   ipv6 address fd00::2:105/127
!
interface Loopback0
   ipv6 address fd00::100/128
!
no ip routing
!
ipv6 unicast-routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000099 result accept
!
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range fd00::2:100/120 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv6
      neighbor LEAF activate
      network fd00::100/128
!
```
**Spine2**
```
!
service routing protocols model multi-agent
!
hostname Spine2
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:201/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:203/127
!
interface Ethernet3
   no switchport
   ipv6 address fd00::2:205/127
!
interface Loopback0
   ipv6 address fd00::200/128
!
no ip routing
!
ipv6 unicast-routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000099 result accept
!
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range fd00::2:200/120 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv6
      neighbor LEAF activate
      network fd00::200/128
!
```
**Leaf1**
```
!
service routing protocols model multi-agent
!
hostname Leaf1
!
vlan 10
   name SERVICE-1
!
vlan 20
   name SERVICE-2
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:100/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:200/127
!
interface Ethernet8
   switchport access vlan 10
   spanning-tree portfast
!
interface Loopback0
   ipv6 address fd00::1:1/128
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.0.1
   maximum-paths 2
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60000
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor fd00::2:101 peer group SPINE
   neighbor fd00::2:201 peer group SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv6
      neighbor SPINE activate
      network fd00::1:1/128
!
```
**Leaf2**
```
!
service routing protocols model multi-agent
!
hostname Leaf2
!
vlan 10
   name SERVICE-1
!
vlan 20
   name SERVICE-2
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:102/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:202/127
!
interface Ethernet8
   switchport access vlan 20
   spanning-tree portfast
!
interface Loopback0
   ipv6 address fd00::1:2/128
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60002
   bgp asn notation asdot
   router-id 10.0.0.2
   maximum-paths 2
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60000
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor fd00::2:103 peer group SPINE
   neighbor fd00::2:203 peer group SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv6
      neighbor SPINE activate
      network fd00::1:2/128
!
```
**Leaf3**
```
!
service routing protocols model multi-agent
!
hostname Leaf3
!
vlan 10
   name SERVICE-1
!
vlan 20
   name SERVICE-2
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:104/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:204/127
!
interface Ethernet7
   switchport access vlan 10
   spanning-tree portfast
!
interface Ethernet8
   switchport access vlan 20
   spanning-tree portfast
!
interface Loopback0
   ipv6 address fd00::1:3/128
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60003
   bgp asn notation asdot
   router-id 10.0.0.3
   maximum-paths 2
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60000
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor fd00::2:105 peer group SPINE
   neighbor fd00::2:205 peer group SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv6
      neighbor SPINE activate
      network fd00::1:3/128
!
```
## Вывод show commands
**Leaf1**
```
Leaf1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 4200000097
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.2:20 mac-ip 5000.0007.8001
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:20 mac-ip 5000.0007.8001
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >Ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 * >      RD: 10.0.0.1:10 imet fd00::1:1
                                 -                     -       -       0       i
 * >      RD: 10.0.0.1:20 imet fd00::1:1
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.2:10 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:10 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >Ec    RD: 10.0.0.2:20 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:20 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >Ec    RD: 10.0.0.3:10 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
Leaf1#show bgp evpn summary 
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 64086.60001
Neighbor Status Codes: m - Under maintenance
  Neighbor    V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:101 4 64086.60000      1645      1658    0    0 00:12:24 Estab   6      6
  fd00::2:201 4 64086.60000      1650      1660    0    0 00:12:25 Estab   6      6
!
Leaf1#show vxlan vtep
Remote VTEPS for Vxlan1:

VTEP            Tunnel Type(s)
--------------- --------------
fd00::1:2       flood, unicast         
fd00::1:3       flood, unicast
!
Leaf1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5000.0006.8001    DYNAMIC     Et8        1       0:00:59 ago
  10    5000.0008.8001    DYNAMIC     Vx1        1       0:00:59 ago
  20    5000.0007.8001    DYNAMIC     Vx1        1       0:00:05 ago
  20    5000.0009.8001    DYNAMIC     Vx1        1       0:00:05 ago
Total Mac Addresses for this criterion: 4
!
```
**Leaf2**
```
Leaf2#show bgp evpn 
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 4200000098
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.1:10 mac-ip 5000.0006.8001
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:10 mac-ip 5000.0006.8001
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >      RD: 10.0.0.2:20 mac-ip 5000.0007.8001
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.1:10 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:10 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >Ec    RD: 10.0.0.1:20 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:20 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >      RD: 10.0.0.2:10 imet fd00::1:2
                                 -                     -       -       0       i
 * >      RD: 10.0.0.2:20 imet fd00::1:2
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:10 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 imet fd00::1:3
                                 fd00::1:3             -       100     0       64086.60000 64086.60003 i
Leaf2#show bgp evpn summary 
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 64086.60002
Neighbor Status Codes: m - Under maintenance
  Neighbor    V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:103 4 64086.60000      2119      2126    0    0 01:29:15 Estab   7      7
  fd00::2:203 4 64086.60000      2118      2115    0    0 01:29:11 Estab   7      7
Leaf2#show vxlan vtep 
Remote VTEPS for Vxlan1:

VTEP            Tunnel Type(s)
--------------- --------------
fd00::1:3       flood, unicast
fd00::1:1       flood, unicast

Total number of remote VTEPS:  2
Leaf2#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5000.0006.8001    DYNAMIC     Vx1        1       0:02:45 ago
  10    5000.0008.8001    DYNAMIC     Vx1        1       0:02:45 ago
  20    5000.0007.8001    DYNAMIC     Et8        1       0:01:51 ago
  20    5000.0009.8001    DYNAMIC     Vx1        1       0:01:51 ago
Total Mac Addresses for this criterion: 4
```
**Leaf3**
```
Leaf3#show bgp evpn 
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 4200000099
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.1:10 mac-ip 5000.0006.8001
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:10 mac-ip 5000.0006.8001
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >Ec    RD: 10.0.0.2:20 mac-ip 5000.0007.8001
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:20 mac-ip 5000.0007.8001
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >      RD: 10.0.0.3:10 mac-ip 5000.0008.8001
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.1:10 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:10 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >Ec    RD: 10.0.0.1:20 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 *  ec    RD: 10.0.0.1:20 imet fd00::1:1
                                 fd00::1:1             -       100     0       64086.60000 64086.60001 i
 * >Ec    RD: 10.0.0.2:10 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:10 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >Ec    RD: 10.0.0.2:20 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 *  ec    RD: 10.0.0.2:20 imet fd00::1:2
                                 fd00::1:2             -       100     0       64086.60000 64086.60002 i
 * >      RD: 10.0.0.3:10 imet fd00::1:3
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:20 imet fd00::1:3
                                 -                     -       -       0       i
Leaf3#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 64086.60003
Neighbor Status Codes: m - Under maintenance
  Neighbor    V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:105 4 64086.60000       743       741    0    0 00:30:31 Estab   6      6
  fd00::2:205 4 64086.60000       735       736    0    0 00:30:31 Estab   6      6
Leaf3#show vxlan vtep 
Remote VTEPS for Vxlan1:

VTEP            Tunnel Type(s)
--------------- --------------
fd00::1:2       unicast, flood
fd00::1:1       unicast, flood

Total number of remote VTEPS:  2
Leaf3#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    5000.0006.8001    DYNAMIC     Vx1        1       0:03:38 ago
  10    5000.0008.8001    DYNAMIC     Et7        1       0:03:38 ago
  20    5000.0007.8001    DYNAMIC     Vx1        1       0:02:44 ago
  20    5000.0009.8001    DYNAMIC     Et8        1       0:02:44 ago
Total Mac Addresses for this criterion: 4
```
## Тестирование L2 связности между клиентами
**Client1 --> Client3**
```
Client1#ping FD00::4:3 source FD00::4:1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00::4:3, timeout is 2 seconds:
Packet sent with a source address of FD00::4:1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 11/11/14 ms
```
**Client2 --> Client4**
```
Client2#ping FD00::5:4 source FD00::5:2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00::5:4, timeout is 2 seconds:
Packet sent with a source address of FD00::5:2
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 10/11/15 ms
```
