# Маршрутизацию между VRF через внешнее устройство с использованием маршрутов EVPN route-type 5
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить маршрутизацию между VRF через внешнее устройство
5. Конфигурация АСО
6. Тестирование связности
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab7/screenshots/Topology.PNG)
## Адресное пространство для сети
Адресацию для АСО я переиспользовал из работы "[Проектирование адресного пространства](https://github.com/Vorobey1/otus-dc-network-design/edit/main/lab1/README.md)".

|Device |Loopback    |
|:-----:|:----------:|
|Spine1 |10.0.1.0/32 |
|Spine2 |10.0.2.0/32 |
|Leaf1  |10.0.0.1/32 |
|Leaf2  |10.0.0.2/32 |
|Leaf3  |10.0.0.3/32 |
|Leaf4  |10.0.0.4/32 |
|Leaf3-4|10.1.3.4/32 |

|p2p         |Spine1      |Spine2      |
|:----------:|:----------:|:----------:|
|Leaf1       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf2       |10.2.1.2/31 |10.2.2.2/31 |
|Leaf3       |10.2.1.4/31 |10.2.2.4/31 |
|Leaf4       |10.2.1.6/31 |10.2.2.6/31 |

|Device  |Ip-address  |
|:------:|:----------:|
|Client1 |10.4.0.1/24 |
|Client2 |10.4.1.2/24 |
|Client3 |10.4.0.3/24 |
|Client4 |10.4.2.4/24 |

|VRF         |IP Pool     |
|:----------:|:----------:|
|SERVICE-1   |10.4.0.0/16 |
|SERVICE-2   |10.5.0.0/16 |

## Настройка маршрутизации между VRF через внешнее устройство
## Конфигурация АСО
<details> 

<summary>Spine1</summary>

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
interface Ethernet4
   no switchport
   ip address 10.2.1.7/31
!
interface Loopback0
   ip address 10.0.1.0/32
!
ip routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000100 result accept
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
end
```
</details>

<details> 
<summary>Spine2</summary>
   
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
interface Ethernet4
   no switchport
   ip address 10.2.2.7/31
!
interface Loopback0
   ip address 10.0.2.0/32
!
ip routing
!
peer-filter LEAF_RANGE_ASN
   10 match as-range 4200000097-4200000100 result accept
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
</details>

<details> 
<summary>Leaf1</summary>
   
```
!
service routing protocols model multi-agent
!
hostname Leaf1
!
vlan 10-11
!
vrf instance SERVICE-1
!
vrf instance SERVICE-2
!
interface Port-Channel1
   description Client1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0011:1111:1111:1111:1111
      route-target import 11:11:11:11:11:11
   lacp system-id 1111.1111.1111
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel2
   description Client2
   switchport access vlan 11
   !
   evpn ethernet-segment
      identifier 0022:2222:2222:2222:2222
      route-target import 22:22:22:22:22:22
   lacp system-id 2222.2222.2222
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Ethernet1
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.0/31
!
interface Ethernet7
   channel-group 1 mode active
!
interface Ethernet8
   channel-group 2 mode active
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Vlan10
   vrf SERVICE-1
   ip address 10.4.0.253/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE-2
   ip address 10.5.0.253/24
   ip virtual-router address 10.5.0.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vrf SERVICE-1 vni 1000
   vxlan vrf SERVICE-2 vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE-1
ip routing vrf SERVICE-2
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
   vlan 11
      rd auto
      route-target both 11:10011
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.1/32
   !
   vrf SERVICE-1
      rd 10.0.0.1:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
   !
   vrf SERVICE-2
      rd 10.0.0.1:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
!
```
</details>

<details> 
<summary>Leaf2</summary>
   
```
!
service routing protocols model multi-agent
!
hostname Leaf2
!
vlan 10-11
!
vrf instance SERVICE-1
!
vrf instance SERVICE-2
!
interface Port-Channel1
   description Client1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0011:1111:1111:1111:1111
      route-target import 11:11:11:11:11:11
   lacp system-id 1111.1111.1111
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel2
   description client2
   switchport access vlan 11
   !
   evpn ethernet-segment
      identifier 0022:2222:2222:2222:2222
      route-target import 22:22:22:22:22:22
   lacp system-id 2222.2222.2222
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Ethernet1
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/31
!
interface Ethernet7
   channel-group 1 mode active
!
interface Ethernet8
   channel-group 2 mode active
!
interface Loopback0
   ip address 10.0.0.2/32
!
interface Vlan10
   vrf SERVICE-1
   ip address 10.4.0.252/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE-2
   ip address 10.5.0.252/24
   ip virtual-router address 10.5.0.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vrf SERVICE-1 vni 1000
   vxlan vrf SERVICE-2 vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE-1
ip routing vrf SERVICE-2
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
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 11
      rd auto
      route-target both 11:10011
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor SPINE activate
      network 10.0.0.2/32
   !
   vrf SERVICE-1
      rd 10.0.0.2:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
   !
   vrf SERVICE-2
      rd 10.0.0.2:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
!
```
</details>

<details> 
<summary>Leaf3</summary>
   
```
```
</details>

<details> 
<summary>Leaf4</summary>
   
```
```
</details>

<details> 
<summary>R1</summary>
   
```
```
</details>
