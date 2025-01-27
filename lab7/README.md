# Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить ESI-LAG
4. Настроить MLAG
5. Конфигурация АСО
6. Тестирование связности и отказоустойчивости
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

## Настройка ESI-LAG
На Leaf1 и Leaf2 настроим ESI-LAG для Client1 и Client2.  
Соотнесем физические интерфесы с port-channel
```
interface Ethernet7
   channel-group 1 mode active
interface Ethernet8
   channel-group 2 mode active
```
Настроим port-channel
```
interface Port-Channel1
   description Client1
   switchport access vlan 10
   evpn ethernet-segment
      identifier 0011:1111:1111:1111:1111
      route-target import 11:11:11:11:11:11
   lacp system-id 1111.1111.1111
interface Port-Channel2
   description Client2
   switchport access vlan 11
   evpn ethernet-segment
      identifier 0022:2222:2222:2222:2222
      route-target import 22:22:22:22:22:22
   lacp system-id 2222.2222.2222
```
После данной настройки Leaf1 и Leaf2 начнут отправлять EVPN сообщения 1 и 4 типов  
EVPN route-type 4
```
Spine2#show bgp evpn route-type ethernet-segment detail
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 4200000096
BGP routing table entry for ethernet-segment 0011:1111:1111:1111:1111 10.0.0.1, Route Distinguisher: 10.0.0.1:1
 Paths: 1 available
  64086.60001
    10.0.0.1 from 10.2.2.0 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:11:11:11:11:11:11
BGP routing table entry for ethernet-segment 0011:1111:1111:1111:1111 10.0.0.2, Route Distinguisher: 10.0.0.2:1
 Paths: 1 available
  64086.60002
    10.0.0.2 from 10.2.2.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:11:11:11:11:11:11
BGP routing table entry for ethernet-segment 0022:2222:2222:2222:2222 10.0.0.1, Route Distinguisher: 10.0.0.1:1
 Paths: 1 available
  64086.60001
    10.0.0.1 from 10.2.2.0 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:22:22:22:22:22:22
BGP routing table entry for ethernet-segment 0022:2222:2222:2222:2222 10.0.0.2, Route Distinguisher: 10.0.0.2:1
 Paths: 1 available
  64086.60002
    10.0.0.2 from 10.2.2.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:22:22:22:22:22:22
```
Данный тип сообщений позволяет выбрать назначенного отравителя для каждого EVI
```

Leaf1#show bgp evpn instance 
EVPN instance: VLAN 10
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10:10010
  Route target export: Route-Target-AS:10:10010
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.0.1
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0011:1111:1111:1111:1111
      Interface: Port-Channel1
      Mode: all-active
      State: up
      ES-Import RT: 11:11:11:11:11:11
      DF election algorithm: modulus
      Designated forwarder: 10.0.0.1
      Non-Designated forwarder: 10.0.0.2
EVPN instance: VLAN 11
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:11:10011
  Route target export: Route-Target-AS:11:10011
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.0.1
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0022:2222:2222:2222:2222
      Interface: Port-Channel2
      Mode: all-active
      State: up
      ES-Import RT: 22:22:22:22:22:22
      DF election algorithm: modulus
      Designated forwarder: 10.0.0.2
      Non-Designated forwarder: 10.0.0.1
```
EVPN route-type 1  
```
Spine2#show bgp evpn route-type auto-discovery detail 
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 4200000096
BGP routing table entry for auto-discovery 0 0011:1111:1111:1111:1111, Route Distinguisher: 10.0.0.1:10
 Paths: 1 available
  64086.60001
    10.0.0.1 from 10.2.2.0 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010
BGP routing table entry for auto-discovery 0 0011:1111:1111:1111:1111, Route Distinguisher: 10.0.0.2:10
 Paths: 1 available
  64086.60002
    10.0.0.2 from 10.2.2.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010
BGP routing table entry for auto-discovery 0 0022:2222:2222:2222:2222, Route Distinguisher: 10.0.0.1:11
 Paths: 1 available
  64086.60001
    10.0.0.1 from 10.2.2.0 (10.0.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:11:10011 TunnelEncap:tunnelTypeVxlan
      VNI: 10011
BGP routing table entry for auto-discovery 0 0022:2222:2222:2222:2222, Route Distinguisher: 10.0.0.2:11
 Paths: 1 available
  64086.60002
    10.0.0.2 from 10.2.2.2 (10.0.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, best
      Extended Community: Route-Target-AS:11:10011 TunnelEncap:tunnelTypeVxlan
      VNI: 10011
```
Благодаря данным маршрутам можно получить L2 и L3 ECMP  
L2 ECMP
```
Leaf4#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------
VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  5000.0006.8001  EVPN      Vx1  10.0.0.1         1       0:00:02 ago
                                     10.0.0.2
```
L3 ECMP
```
Leaf4#show ip route vrf SERVICE
VRF: SERVICE
 B E      10.4.0.1/32 [200/0] via VTEP 10.0.0.2 VNI 10000 router-mac 50:00:00:03:37:66 local-interface Vxlan1
                              via VTEP 10.0.0.1 VNI 10000 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
```
Для L3 ECMP нужны еще маршруты mac-ip, но Leaf1 делает их анонс при помощи route-type 1 на что указывает **EvpnNdFlags:pflag**
```
BGP routing table entry for mac-ip 5000.0006.8001 10.4.0.1, Route Distinguisher: 10.0.0.1:10
  64086.60000 64086.60001
    10.0.0.1 from 10.2.1.7 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:1 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:d5:5d:c0 EvpnNdFlags:pflag
      VNI: 10010 L3 VNI: 10000 ESI: 0011:1111:1111:1111:1111
BGP routing table entry for mac-ip 5000.0006.8001 10.4.0.1, Route Distinguisher: 10.0.0.2:10
  64086.60000 64086.60002
    10.0.0.2 from 10.2.2.7 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:1 Route-Target-AS:10:10010 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:00:00:03:37:66
      VNI: 10010 L3 VNI: 10000 ESI: 0011:1111:1111:1111:1111
```
## Настройка MLAG
На Leaf3 и Leaf4 настроим MLAG для клиентов Client3 и Client4  
Создадим VLAN и SVI для пиринга MLAG устройств, а также настроим Peer Link
```
vlan 4094
   name MLAG
   trunk group MLAGPEER
interface Vlan4094
   description MLAG Peer Sync
   no autostate
   ip address <ip для пиринга>
interface Ethernet4
   channel-group 1000 mode active
interface Ethernet5
   channel-group 1000 mode active
interface Port-Channel1000
   description MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAGPEER
```
Создадим MLAG Domain
```
mlag configuration
   domain-id 1000
   local-interface Vlan4094
   peer-address <ip соседа по MLAG>
   peer-link Port-Channel1000
```
Создадим PO для клиентов и добавим их в MLAG
```
interface Ethernet7
   channel-group 3 mode active
interface Ethernet8
   channel-group 4 mode active
interface Port-Channel3
   description Client3
   switchport access vlan 10
   mlag 3
interface Port-Channel4
   description Client4
   switchport access vlan 20
   mlag 4
```
Изменим настройки VXLAN (Настроим Anycast Lo, настроим использование общего MAC-адреса для обновлений EVPN вместо уникальных для каждого VTEP)
```
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address mlag-system-id
```
Настроим iBGP между MLAG устройствами для отработки отказа падения всех Uplink на VTEP (Знаю что это не best practies)
```
vlan 4093
   name iBGP
interface Vlan4093
   description iBGP Peer
   no autostate
   ip address <ip для iBGP>
router bgp 64086.60003
   neighbor iBGP peer group
   neighbor iBGP remote-as 64086.60003
   neighbor iBGP next-hop-peer
   neighbor iBGP password 7 5BBwhtJStX06PzD9rDdhHg==
   neighbor iBGP send-community extended
   neighbor <ip соседа по iBGP> peer group iBGP
   address-family evpn
      neighbor iBGP activate
   address-family ipv4
      neighbor iBGP activate
```
Проверка MLAG (show mlag)
```
Leaf3#show mlag 
MLAG Configuration:              
domain-id                          :                1000
local-interface                    :            Vlan4094
peer-address                       :         192.168.0.2
peer-link                          :    Port-Channel1000
peer-config                        :          consistent                                                     
MLAG Status:                     
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:15:f4:e8
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False                                                     
MLAG Ports:                      
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   2
```
Вывод маршрутной информации EVPN, полученной от MLAG
```
Leaf1#show bgp evpn next-hop 10.1.3.4
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 4200000097
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:10 mac-ip 5000.0008.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:10 mac-ip 5000.0008.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001 10.4.0.3
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 mac-ip 5000.0008.8001 10.4.0.3
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:10 mac-ip 5000.0008.8001 10.4.0.3
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:10 mac-ip 5000.0008.8001 10.4.0.3
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:20 mac-ip 5000.0009.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:20 mac-ip 5000.0009.8001
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001 10.4.2.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 mac-ip 5000.0009.8001 10.4.2.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:20 mac-ip 5000.0009.8001 10.4.2.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:20 mac-ip 5000.0009.8001 10.4.2.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:10 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:10 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:20 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:20 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:10 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:10 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:20 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:20 imet 10.1.3.4
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
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
end
```
**Leaf1**
```
!
service routing protocols model multi-agent
!
hostname Leaf1
!
vlan 10-11
!
vrf instance SERVICE
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
   vrf SERVICE
   ip address 10.4.0.253/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE
   ip address 10.4.1.253/24
   ip virtual-router address 10.4.1.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
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
   vrf SERVICE
      rd 10.0.0.1:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
!
end
```
**Leaf2**
```
!
service routing protocols model multi-agent
!
hostname Leaf2
!
vlan 10-11
!
vrf instance SERVICE
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
   vrf SERVICE
   ip address 10.4.0.252/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE
   ip address 10.4.1.252/24
   ip virtual-router address 10.4.1.254
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
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
   vrf SERVICE
      rd 10.0.0.2:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
!
end
```
**Leaf3**
```
!
service routing protocols model multi-agent
!
hostname Leaf3
!
no spanning-tree vlan-id 4094
!
vlan 10,20
!
vlan 4093
   name iBGP
!
vlan 4094
   name MLAG
   trunk group MLAGPEER
!
vrf instance SERVICE
!
interface Port-Channel3
   description Client3
   switchport access vlan 10
   mlag 3
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel4
   description Client4
   switchport access vlan 20
   mlag 4
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel1000
   description MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAGPEER
!
interface Ethernet1
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet4
   channel-group 1000 mode active
!
interface Ethernet5
   channel-group 1000 mode active
!
interface Ethernet7
   channel-group 3 mode active
!
interface Ethernet8
   channel-group 4 mode active
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Loopback1
   ip address 10.1.3.4/32
!
interface Vlan10
   vrf SERVICE
   ip address 10.4.0.250/24
   ip virtual-router address 10.4.0.254
!
interface Vlan20
   vrf SERVICE
   ip address 10.4.2.253/24
   ip virtual-router address 10.4.2.254
!
interface Vlan4093
   description iBGP Peer
   no autostate
   ip address 192.168.0.5/30
!
interface Vlan4094
   description MLAG Peer Sync
   no autostate
   ip address 192.168.0.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address mlag-system-id
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf SERVICE vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE
!
mlag configuration
   domain-id 1000
   local-interface Vlan4094
   peer-address 192.168.0.2
   peer-link Port-Channel1000
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
   neighbor iBGP peer group
   neighbor iBGP remote-as 64086.60003
   neighbor iBGP next-hop-peer
   neighbor iBGP password 7 5BBwhtJStX06PzD9rDdhHg==
   neighbor iBGP send-community extended
   neighbor 10.2.1.5 peer group SPINE
   neighbor 10.2.2.5 peer group SPINE
   neighbor 192.168.0.6 peer group iBGP
   !
   vlan 10
      rd auto
      route-target both 10:10010
      route-target both 10:10020
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE activate
      neighbor iBGP activate
   !
   address-family ipv4
      neighbor SPINE activate
      neighbor iBGP activate
      network 10.0.0.3/32
      network 10.1.3.4/32
   !
   vrf SERVICE
      rd 10.0.0.3:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
!
end
```
**Leaf4**
```
!
service routing protocols model multi-agent
!
hostname Leaf4
!
no spanning-tree vlan-id 4094
!
vlan 10,20
!
vlan 4093
   name iBGP
!
vlan 4094
   name MLAG
   trunk group MLAGPEER
!
vrf instance SERVICE
!
interface Port-Channel3
   description Client3
   switchport access vlan 10
   mlag 3
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel4
   description Client4
   switchport access vlan 20
   mlag 4
   spanning-tree portfast
   spanning-tree bpdufilter enable
!
interface Port-Channel1000
   description MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAGPEER
!
interface Ethernet1
   no switchport
   ip address 10.2.1.6/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.6/31
!
interface Ethernet4
   channel-group 1000 mode active
!
interface Ethernet5
   channel-group 1000 mode active
!
interface Ethernet7
   channel-group 3 mode active
!
interface Ethernet8
   channel-group 4 mode active
!
interface Loopback0
   ip address 10.0.0.4/32
!
interface Loopback1
   ip address 10.1.3.4/32
!
interface Vlan10
   no autostate
   vrf SERVICE
   ip address 10.4.0.251/24
   ip virtual-router address 10.4.0.254
!
interface Vlan20
   vrf SERVICE
   ip address 10.4.2.252/24
   ip virtual-router address 10.4.2.254
!
interface Vlan4093
   description iBGP Peer
   no autostate
   ip address 192.168.0.6/30
!
interface Vlan4094
   description MLAG Peer Sync
   no autostate
   ip address 192.168.0.2/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address mlag-system-id
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf SERVICE vni 10000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf SERVICE
!
mlag configuration
   domain-id 1000
   local-interface Vlan4094
   peer-address 192.168.0.1
   peer-link Port-Channel1000
!
router bgp 64086.60003
   bgp asn notation asdot
   router-id 10.0.0.4
   maximum-paths 2
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60000
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor iBGP peer group
   neighbor iBGP remote-as 64086.60003
   neighbor iBGP next-hop-peer
   neighbor iBGP password 7 5BBwhtJStX06PzD9rDdhHg==
   neighbor iBGP send-community extended
   neighbor 10.2.1.7 peer group SPINE
   neighbor 10.2.2.7 peer group SPINE
   neighbor 192.168.0.5 peer group iBGP
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
      neighbor iBGP activate
   !
   address-family ipv4
      neighbor SPINE activate
      neighbor iBGP activate
      network 10.0.0.4/32
      network 10.1.3.4/32
   !
   vrf SERVICE
      rd 10.0.0.4:10000
      route-target import evpn 1:1
      route-target export evpn 1:1
!
end
```
## Тестирование связности и отказоустойчивости
Client1 --> Client2
```
Client1#ping 10.4.1.2 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.1.2, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/21/69 ms
```
Client1 --> Client3
```
Client1#ping 10.4.0.3 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.0.3, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/22/59 ms
```
Client1 --> Client4
```
Client1#ping 10.4.2.4 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.2.4, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/15/18 ms
```
Client1 --> Client4 (Отключение Leaf1, через который в данный момент идет трафик)
```
Client1#ping 10.4.2.4 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.2.4, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 12/13/15 ms
```
Client1 --> Client4 (Отключение Leaf4, через который в данный момент идет трафик)
```
Client1#ping 10.4.2.4 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.2.4, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/13/16 ms
```
