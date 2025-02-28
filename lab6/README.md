# Построение Overlay на основе VXLAN EVPN c L3 VNI
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить eBGP для Underlay
4. Настроить VXLAN EVPN
5. Конфигурация АСО
6. Вывод show commands (show bgp evpn route-type ip-prefix ipv4, show ip route vrf SERVICE)
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
|Client2 |10.4.1.1/24 |
|Client3 |10.4.2.1/24 |
|Client4 |10.4.3.1/24 |

## Настройка eBGP для Underlay сети
При настройке eBGP в топологиях CLOS можно руководствоваться следующим рекомендациям:
1. Использовать схему ASN, которая исключает проблему path-hunting
2. Использовать multipath для ECMP
3. Использовать MP-BGP
4. Использовать BFD, изменить стандартные таймера, настроить аутентификацию

При настройке eBGP для Spine я использовал приватную AS - 64086.60000, для Leaf - 64086.60Ln, где Ln - номер Leaf в диапазоне [001...999].  
Для упрощения конфигурации использовал peer groups. На Spine для автообнаружения соседей использовал bgp listen 

## Настройка VXLAN EVPN c L3VNI
Настраиваем VRF instance на всех Leaf (VTEP) и соотносим int vlan 10,20,30,40 c vrf SERVICE. Включаем routing для vrf
```
vrf instance SERVICE
interface Vlan[10,20,30,40]
   vrf SERVICE
ip routing vrf SERVICE
```
Настраиваем virtual-mac и virtual-ip для общего использования шлюза на VTEP
```
ip virtual-router mac-address 00:00:00:00:00:01
interface Vlan[10,20,30,40]
   ip virtual-router address 10.4.[0,1,2,3].254
```
В существующем NVE на всех VTEP связываем VRF c VNI
```
interface Vxlan1
   vxlan vrf SERVICE vni 10000
```
На VTEP создаем EVPN Instance (EVI) для VRF: указываем RD, RT, влючем редистрибуцию connected сетей
```
router bgp ASN
   vrf SERVICE
      rd 10.0.0.Ln:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
      redistribute connected
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
   ip address 10.2.1.1/31
!
interface Ethernet2
   no switchport
   ip address 10.2.1.3/31
!
interface Ethernet3
   no switchport
   ip address 10.2.1.5/31
!
interface Loopback0
   ip address 10.0.1.0/32
!
ip routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000099 result accept
!
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range 10.2.1.0/24 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv4
      neighbor LEAF activate
      network 10.0.1.0/32
   !
   address-family ipv6
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
   ip address 10.2.2.1/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.3/31
!
interface Ethernet3
   no switchport
   ip address 10.2.2.5/31
!
interface Loopback0
   ip address 10.0.2.0/32
!
ip routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000099 result accept
!
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range 10.2.2.0/24 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv4
      neighbor LEAF activate
      network 10.0.2.0/32
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
   name Client1
!
vrf instance SERVICE
!
interface Ethernet1
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet8
   switchport access vlan 10
   spanning-tree portfast
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Vlan10
   vrf SERVICE
   ip address 10.4.0.253/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf SERVICE vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE
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
   neighbor 10.2.1.1 peer group SPINE
   neighbor 10.2.2.1 peer group SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.1/32
   !
   vrf SERVICE
      rd 10.0.0.1:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
      redistribute connected
!
```
**Leaf2**
```
!
service routing protocols model multi-agent
!
hostname Leaf2
!
vlan 20
   name Client2
!
vrf instance SERVICE
!
interface Ethernet1
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet8
   switchport access vlan 20
   spanning-tree portfast
!
interface Loopback0
   ip address 10.0.0.2/32
!
interface Vlan20
   vrf SERVICE
   ip address 10.4.1.253/24
   ip virtual-router address 10.4.1.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vrf SERVICE vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE
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
   neighbor 10.2.1.3 peer group SPINE
   neighbor 10.2.2.3 peer group SPINE
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.2/32
   !
   vrf SERVICE
      rd 10.0.0.2:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
      redistribute connected
!
```
**Leaf3**
```
!
service routing protocols model multi-agent
!
hostname Leaf3
!
vlan 30
   name Client3
!
vlan 40
   name Client4
!
vrf instance SERVICE
!
interface Ethernet1
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet7
   switchport access vlan 30
   spanning-tree portfast
!
interface Ethernet8
   switchport access vlan 40
   spanning-tree portfast
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Vlan30
   vrf SERVICE
   ip address 10.4.2.253/24
   ip virtual-router address 10.4.2.254
!
interface Vlan40
   vrf SERVICE
   ip address 10.4.3.253/24
   ip virtual-router address 10.4.3.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 30 vni 10030
   vxlan vlan 40 vni 10040
   vxlan vrf SERVICE vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE
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
   neighbor 10.2.1.5 peer group SPINE
   neighbor 10.2.2.5 peer group SPINE
   !
   vlan 30
      rd auto
      route-target both 30:10030
      redistribute learned
   !
   vlan 40
      rd auto
      route-target both 40:10040
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.3/32
   !
   vrf SERVICE
      rd 10.0.0.3:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
      redistribute connected
!
```
## Вывод show commands
**Leaf1**
```
Leaf1#show bgp evpn route-type ip-prefix ipv4 
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 4200000097
          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:10000 ip-prefix 10.4.0.0/24
                                 -                     -       -       0       i
 * >      RD: 10.0.0.2:10000 ip-prefix 10.4.1.0/24
                                 10.0.0.2              -       100     0       64086.60000 64086.60002 i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.2.0/24
                                 10.0.0.3              -       100     0       64086.60000 64086.60003 i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.3.0/24
                                 10.0.0.3              -       100     0       64086.60000 64086.60003 i

Leaf1#show ip route vrf SERVICE
VRF: SERVICE
 C        10.4.0.0/24 is directly connected, Vlan10
 B E      10.4.1.0/24 [200/0] via VTEP 10.0.0.2 VNI 10000 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 B E      10.4.2.0/24 [200/0] via VTEP 10.0.0.3 VNI 10000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.3.0/24 [200/0] via VTEP 10.0.0.3 VNI 10000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
**Leaf2**
```
Leaf2#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 4200000098
          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:10000 ip-prefix 10.4.0.0/24
                                 10.0.0.1              -       100     0       64086.60000 64086.60001 i
 * >      RD: 10.0.0.2:10000 ip-prefix 10.4.1.0/24
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.2.0/24
                                 10.0.0.3              -       100     0       64086.60000 64086.60003 i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.3.0/24
                                 10.0.0.3              -       100     0       64086.60000 64086.60003 i

Leaf2#show ip route vrf SERVICE
VRF: SERVICE
 B E      10.4.0.0/24 [200/0] via VTEP 10.0.0.1 VNI 10000 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 C        10.4.1.0/24 is directly connected, Vlan20
 B E      10.4.2.0/24 [200/0] via VTEP 10.0.0.3 VNI 10000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.3.0/24 [200/0] via VTEP 10.0.0.3 VNI 10000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
**Leaf3**
```
Leaf3#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 4200000099
          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:10000 ip-prefix 10.4.0.0/24
                                 10.0.0.1              -       100     0       64086.60000 64086.60001 i
 * >      RD: 10.0.0.2:10000 ip-prefix 10.4.1.0/24
                                 10.0.0.2              -       100     0       64086.60000 64086.60002 i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.2.0/24
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:10000 ip-prefix 10.4.3.0/24
                                 -                     -       -       0       i

Leaf3#show ip route vrf SERVICE
VRF: SERVICE
 B E      10.4.0.0/24 [200/0] via VTEP 10.0.0.1 VNI 10000 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
 B E      10.4.1.0/24 [200/0] via VTEP 10.0.0.2 VNI 10000 router-mac 50:00:00:03:37:66 local-interface Vxlan1
 C        10.4.2.0/24 is directly connected, Vlan30
 C        10.4.3.0/24 is directly connected, Vlan40
```
## Тестирование L3 связности между клиентскими сетями
**Client1 --> Client2**
```
Client1#ping 10.4.1.1 so 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/22/44 ms
```
**Client1 --> Client3**
```
Client1#ping 10.4.2.1 so 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.2.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/16/29 ms
```
**Client1 --> Client4**
```
Client1#ping 10.4.3.1 so 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.3.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/16/26 ms
```
