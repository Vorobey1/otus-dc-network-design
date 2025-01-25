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

|p2p         |Spine1      |Spine2      |
|:----------:|:----------:|:----------:|
|Leaf1       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf2       |10.2.1.2/31 |10.2.2.2/31 |
|Leaf3       |10.2.1.4/31 |10.2.2.4/31 |
|Leaf4       |10.2.1.6/31 |10.2.2.6/31 |

|Device  |Ip-address  |
|:------:|:----------:|
|Client1 |10.4.0.1/24 |
|Client2 |10.4.1.1/24 |
|Client3 |10.4.2.1/24 |
|Client4 |10.4.3.1/24 |

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

## Конфигурация АСО

## Конфигурация

