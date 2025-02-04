# Проектирование сегментированной сети в датацентре на базе EVPN\VXLAN c использованием Multi-Fabric
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить маршрутизацию между VRF через FW
4. Выводы show commands после настройки маршрутизаци между VRF
5. Тестирование связности в POD1
6. Настроить Multi-Fabric
7. Выводы show commands после настройки Multi-Fabric
8. Тестирование связности между POD1 и POD2
9. Конфигурация АСО

## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS, FW - Cisco ASAv, L3VPN - Cisco vIOS Router.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/diplom/screenshots/Topology.PNG)  
## Адресное пространство для сети
Адресацию для АСО я переиспользовал из работы "[Проектирование адресного пространства](https://github.com/Vorobey1/otus-dc-network-design/edit/main/lab1/README.md)".

<details>
<summary>Адресное пространство POD1</summary>

|Device   |Loopback    |
|:-------:|:----------:|
|Spine11  |10.0.1.0/32 |
|Spine12  |10.0.2.0/32 |
|Leaf11   |10.0.0.1/32 |
|Leaf12   |10.0.0.2/32 |
|BLeaf13  |10.0.0.3/32 |
|BLeaf14  |10.0.0.4/32 |

|p2p          |Spine11     |Spine12     |
|:-----------:|:----------:|:----------:|
|Leaf11       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf12       |10.2.1.2/31 |10.2.2.2/31 |
|BLeaf13      |10.2.1.4/31 |10.2.2.4/31 |
|BLeaf14      |10.2.1.6/31 |10.2.2.6/31 |

|Device  |Ip-address  |
|:------:|:----------:|
|Client1 |10.4.0.1/24 |
|Client2 |10.5.0.1/24 |
|Client3 |10.6.0.1/24 |

|VRF         |IP Pool     |
|:----------:|:----------:|
|DEV         |10.4.0.0/16 |
|STAGE       |10.5.0.0/16 |
|PROD        |10.6.0.0/16 |
</details>

<details>
<summary>Адресное пространство POD2</summary>

|Device   |Loopback    |
|:-------:|:----------:|
|Spine21  |10.8.1.0/32 |
|Spine22  |10.8.2.0/32 |
|Leaf21   |10.8.0.1/32 |
|Leaf22   |10.8.0.2/32 |
|BLeaf23  |10.8.0.3/32 |
|BLeaf24  |10.8.0.4/32 |

|p2p          |Spine21      |Spine22      |
|:-----------:|:-----------:|:-----------:|
|Leaf21       |10.10.1.0/31 |10.10.2.0/31 |
|Leaf22       |10.10.1.2/31 |10.10.2.2/31 |
|BLeaf23      |10.10.1.4/31 |10.10.2.4/31 |
|BLeaf24      |10.10.1.6/31 |10.10.2.6/31 |

|Device  |Ip-address  |
|:------:|:----------:|
|Client1 |10.12.0.1/24 |
|Client2 |10.13.0.1/24 |
|Client3 |10.14.0.1/24 |

|VRF         |IP Pool     |
|:----------:|:----------:|
|DEV         |10.12.0.0/16 |
|STAGE       |10.13.0.0/16 |
|PROD        |10.14.0.0/16 |
</details>

## Настройка маршрутизации между VRF чере Firewall
В качестве внешнего устройства я использовал Firewall Cisco ASA (FW1). FW1 будет выполнять Route Leaking между VRF-ами: DEV, STAGE, PROD. 
Для этого необходимо настроить BGP соседство в каждом VRF между BLeaf13/BLeaf14 и FW1. FW1 будет всю маршрутную информацию собирать в GRT.

Настраиваем VRF
```
BLeaf13/BLeaf14
!
vrf instance DEV
vrf instance STAGE
vrf instance PROD
!
ip routing vrf DEV
ip routing vrf STAGE
ip routing vrf PROD
!
Leaf23/Leaf24
!
vrf instance DEV
vrf instance STAGE
vrf instance PROD
!
ip routing vrf DEV
ip routing vrf STAGE
ip routing vrf PROD
!
```
Настраиваем SVI и сабинтерфейсы на BLeaf13/BLeaf14 и FW1 соответственно
```
BLeaf13
!
vlan 3000,3001,3002
!
interface Vlan3000
   vrf DEV
   ip address 10.4.254.2/30
interface Vlan3001
   vrf STAGE
   ip address 10.5.254.2/30
interface Vlan3002
   vrf PROD
   ip address 10.6.254.2/30
!
BLeaf14
!
vlan 4000,4001,4002
!
interface Vlan4000
   vrf DEV
   ip address 10.4.254.6/30
interface Vlan4001
   vrf STAGE
   ip address 10.5.254.6/30
interface Vlan4002
   vrf PROD
   ip address 10.6.254.6/30
FW1
!
zone DEV
zone STAGE
zone PROD
!
interface GigabitEthernet0/0.3000
 vlan 3000
 nameif DEV1
 security-level 30
 zone-member DEV
 ip address 10.4.254.1 255.255.255.252 
interface GigabitEthernet0/0.3001
 vlan 3001
 nameif STAGE1
 security-level 60
 zone-member STAGE
 ip address 10.5.254.1 255.255.255.252 
interface GigabitEthernet0/0.3002
 vlan 3002
 nameif PROD1
 security-level 90
 zone-member PROD
 ip address 10.6.254.1 255.255.255.252 
interface GigabitEthernet0/1.4000
 vlan 4000
 nameif DEV2
 security-level 30
 zone-member DEV
 ip address 10.4.254.5 255.255.255.252 
interface GigabitEthernet0/1.4001
 vlan 4001
 nameif STAGE2
 security-level 60
 zone-member STAGE
 ip address 10.5.254.5 255.255.255.252 
interface GigabitEthernet0/1.4002
 vlan 4002
 nameif PROD2
 security-level 90
 zone-member PROD
 ip address 10.6.254.5 255.255.255.252 
!
```
На BLeaf13, BLeaf14 создаем все l2vni и l3vni в нашем POD
```
!
interface Vxlan1
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf DEV vni 1000
   vxlan vrf PROD vni 3000
   vxlan vrf STAGE vni 2000
!
router bgp 64086.60001
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   vlan 11
      rd auto
      route-target both 11:10011
      redistribute learned
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   vrf DEV
      rd 10.0.0.3:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
   vrf PROD
      rd 10.0.0.3:3000
      route-target import evpn 3:3000
      route-target export evpn 3:3000
   vrf STAGE
      rd 10.0.0.3:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
!
```
Настраиваем eBGP в каждом VRF на BLeaf13 и BLeaf14.  Для уменьшения маршрутной информации настроим route-map, которая будет убирать из аннонсов префиксы /32
```
Bleaf13
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60001
   vrf DEV
      neighbor 10.4.254.1 remote-as 64086.59998
      neighbor 10.4.254.1 route-map EX_MACIP_RM out
      aggregate-address 10.4.0.0/16 summary-only
      redistribute connected
   vrf PROD
      neighbor 10.6.254.1 remote-as 64086.59998
      neighbor 10.6.254.1 route-map EX_MACIP_RM out
      aggregate-address 10.6.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.5.254.1 remote-as 64086.59998
      neighbor 10.5.254.1 route-map EX_MACIP_RM out
      aggregate-address 10.5.0.0/16 summary-only
      redistribute connected
!
BLeaf14
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60001
   vrf DEV
      neighbor 10.4.254.5 remote-as 64086.59998
      neighbor 10.4.254.5 route-map EX_MACIP_RM out
      aggregate-address 10.4.0.0/16 summary-only
      redistribute connected
   vrf PROD
      neighbor 10.6.254.5 remote-as 64086.59998
      neighbor 10.6.254.5 route-map EX_MACIP_RM out
      aggregate-address 10.6.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.5.254.5 remote-as 64086.59998
      neighbor 10.5.254.5 route-map EX_MACIP_RM out
      aggregate-address 10.5.0.0/16 summary-only
      redistribute connected
!
```
Настроим eBGP на FW1
```
!
router bgp 64086.59998
 bgp log-neighbor-changes
 bgp asnotation dot
 address-family ipv4 unicast
  neighbor 10.4.254.2 remote-as 64086.60001
  neighbor 10.4.254.2 activate
  neighbor 10.4.254.2 default-originate
  neighbor 10.5.254.2 remote-as 64086.60001
  neighbor 10.5.254.2 activate
  neighbor 10.5.254.2 default-originate
  neighbor 10.6.254.2 remote-as 64086.60001
  neighbor 10.6.254.2 activate
  neighbor 10.6.254.2 default-originate
  neighbor 10.4.254.6 remote-as 64086.60001
  neighbor 10.4.254.6 activate
  neighbor 10.4.254.6 default-originate
  neighbor 10.5.254.6 remote-as 64086.60001
  neighbor 10.5.254.6 activate
  neighbor 10.5.254.6 default-originate
  neighbor 10.6.254.6 remote-as 64086.60001
  neighbor 10.6.254.6 activate
  neighbor 10.6.254.6 default-originate
  maximum-paths 2
  no auto-summary
  no synchronization
 exit-address-family
!
```
## Выводы show commands после настройки маршрутизаци между VRF
**FW1**
<details>
<summary>show route</summary>

```
R1#show ip route vrf SERVICE-1
Gateway of last resort is not set
B        10.4.0.0 255.255.0.0 [20/0] via 10.4.254.6, 01:51:16
                              [20/0] via 10.4.254.2, 01:51:16
C        10.4.254.0 255.255.255.252 is directly connected, DEV1
L        10.4.254.1 255.255.255.255 is directly connected, DEV1
C        10.4.254.4 255.255.255.252 is directly connected, DEV2
L        10.4.254.5 255.255.255.255 is directly connected, DEV2
B        10.5.0.0 255.255.0.0 [20/0] via 10.5.254.6, 01:51:15
                              [20/0] via 10.5.254.2, 01:51:15
C        10.5.254.0 255.255.255.252 is directly connected, STAGE1
L        10.5.254.1 255.255.255.255 is directly connected, STAGE1
C        10.5.254.4 255.255.255.252 is directly connected, STAGE2
L        10.5.254.5 255.255.255.255 is directly connected, STAGE2
B        10.6.0.0 255.255.0.0 [20/0] via 10.6.254.6, 01:51:16
                              [20/0] via 10.6.254.2, 01:51:16
C        10.6.254.0 255.255.255.252 is directly connected, PROD1
L        10.6.254.1 255.255.255.255 is directly connected, PROD1
C        10.6.254.4 255.255.255.252 is directly connected, PROD2
L        10.6.254.5 255.255.255.255 is directly connected, PROD2
```
</details>

**Leaf11**
<details>
<summary>show ip route vrf DEV</summary>

```
Leaf11#show ip route vrf DEV
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 1000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 1000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.0.0/24 is directly connected, Vlan10
 B I      10.4.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 1000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 1000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf STAGE</summary>

```
Leaf1#show ip route vrf STAGE
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 2000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 2000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.5.0.0/24 is directly connected, Vlan11
 B I      10.5.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 2000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 2000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf PROD</summary>

```
Leaf11#show ip route vrf PROD
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 3000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 3000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.6.0.0/24 is directly connected, Vlan20
 B I      10.6.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 3000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 3000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show bgp evpn route-type ip-prefix ipv4</summary>

```
Leaf11#show bgp evpn route-type ip-prefix ipv4 
          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.3:1000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.3:2000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.3:3000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.3:3000 ip-prefix 0.0.0.0/0
                                 10.0.0.3              -       100     0       64086.59998 i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.4:1000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.4:2000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.4:3000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.4:3000 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       64086.59998 i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.4.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.4.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.4.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.4.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.5.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.5.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.5.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.5.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.3:3000 ip-prefix 10.6.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.3:3000 ip-prefix 10.6.0.0/16
                                 10.0.0.3              -       100     0       i Or-ID: 10.0.0.3 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.0.4:3000 ip-prefix 10.6.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.0.4:3000 ip-prefix 10.6.0.0/16
                                 10.0.0.4              -       100     0       i Or-ID: 10.0.0.4 C-LST: 10.0.2.0 
```
</details>

## Тестирование связности в POD1

Client3 ping Client1
```
Client3#ping 10.4.0.1 so 10.6.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 29/31/33 ms
```
Client3 ping Client2
```
Client3#ping 10.5.0.1 so 10.6.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 25/31/48 ms
```
Client1 ping Client3 (Правильно что нет пинга: меньше security-level)
```
Client1#ping 10.6.0.1 so 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.6.0.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```
## Настрйока Multi-Fabric между POD1 и POD2 
Multi-Fabric будет представлять из себя L3 Hand-off (Настройка eBGP в каждом VRF: DEV, STAGE, PROD). Предварительно был настроен POD2 по аналогии с POD1, но в своем адресном пространстве.

Настраиваем VRF на L3
```
!
ip vrf DEV
 rd 10.4.255.255:1
 route-target export 1:1000
 route-target export 4:4000
 route-target import 1:1000
 route-target import 4:4000
ip vrf PROD
 rd 10.4.255.255:3
 route-target export 3:3000
 route-target export 6:6000
 route-target import 3:3000
 route-target import 6:6000
ip vrf STAGE
 rd 10.4.255.255:2
 route-target export 2:2000
 route-target export 5:5000
 route-target import 2:2000
 route-target import 5:5000
!
```
Настраиваем SVI и сабинтерфейсы на BLeaf13/BLeaf14/BLeaf23/BLeaf24 и R1 соответственно
```
BLeaf13
!
vlan 2000,2001,2002
!
interface Vlan2000
   vrf DEV
   ip address 10.4.255.2/30
interface Vlan2001
   vrf STAGE
   ip address 10.5.255.2/30
interface Vlan2002
   vrf PROD
   ip address 10.6.255.2/30
!
BLeaf14
!
vlan 2010,2011,2012
!
interface Vlan2010
   vrf DEV
   ip address 10.4.255.6/30
interface Vlan2011
   vrf STAGE
   ip address 10.5.255.6/30
interface Vlan2012
   vrf PROD
   ip address 10.6.255.6/30
!
BLeaf23
!
vlan 2100,2101,2102
!
interface Vlan2100
   vrf DEV
   ip address 10.12.255.2/30
interface Vlan2101
   vrf STAGE
   ip address 10.13.255.2/30
interface Vlan2102
   vrf PROD
   ip address 10.14.255.2/30
!
BLeaf24
!
vlan 2110,2111,2112
!
interface Vlan2110
   vrf DEV
   ip address 10.12.255.6/30
interface Vlan2111
   vrf STAGE
   ip address 10.13.255.6/30
interface Vlan2112
   vrf PROD
   ip address 10.14.255.6/30
!
L3
!
interface GigabitEthernet0/0.2000
 encapsulation dot1Q 2000
 ip vrf forwarding DEV
 ip address 10.4.255.1 255.255.255.252
interface GigabitEthernet0/0.2001
 encapsulation dot1Q 2001
 ip vrf forwarding STAGE
 ip address 10.5.255.1 255.255.255.252
interface GigabitEthernet0/0.2002
 encapsulation dot1Q 2002
 ip vrf forwarding PROD
 ip address 10.6.255.1 255.255.255.252
interface GigabitEthernet0/1.2010
 encapsulation dot1Q 2010
 ip vrf forwarding DEV
 ip address 10.4.255.5 255.255.255.252
interface GigabitEthernet0/1.2011
 encapsulation dot1Q 2011
 ip vrf forwarding STAGE
 ip address 10.5.255.5 255.255.255.252
interface GigabitEthernet0/1.2012
 encapsulation dot1Q 2012
 ip vrf forwarding PROD
 ip address 10.6.255.5 255.255.255.252
interface GigabitEthernet0/2.2100
 encapsulation dot1Q 2100
 ip vrf forwarding DEV
 ip address 10.12.255.1 255.255.255.252
interface GigabitEthernet0/2.2101
 encapsulation dot1Q 2101
 ip vrf forwarding STAGE
 ip address 10.13.255.1 255.255.255.252
interface GigabitEthernet0/2.2102
 encapsulation dot1Q 2102
 ip vrf forwarding PROD
 ip address 10.14.255.1 255.255.255.252
interface GigabitEthernet0/3.2110
 encapsulation dot1Q 2110
 ip vrf forwarding DEV
 ip address 10.12.255.5 255.255.255.252
interface GigabitEthernet0/3.2111
 encapsulation dot1Q 2111
 ip vrf forwarding STAGE
 ip address 10.13.255.5 255.255.255.252
interface GigabitEthernet0/3.2112
 encapsulation dot1Q 2112
 ip vrf forwarding PROD
 ip address 10.14.255.5 255.255.255.252
!
```
Настраиваем eBGP в каждом VRF на BLeaf13, BLeaf14, BLeaf23, BLeaf24
```
Bleaf13
!
router bgp 64086.60001
   vrf DEV
      neighbor 10.4.255.1 remote-as 64086.59999
      aggregate-address 10.4.0.0/16 summary-only
      redistribute connected
   vrf PROD
      neighbor 10.6.255.1 remote-as 64086.59999
      aggregate-address 10.6.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.5.255.1 remote-as 64086.59999
      aggregate-address 10.5.0.0/16 summary-only
      redistribute connected
!
Bleaf14
!
router bgp 64086.60001
 vrf DEV
      neighbor 10.4.255.5 remote-as 64086.59999
      aggregate-address 10.4.0.0/16 summary-only
      redistribute connected
   vrf PROD
      neighbor 10.6.255.5 remote-as 64086.59999
      aggregate-address 10.6.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.5.255.5 remote-as 64086.59999
      aggregate-address 10.5.0.0/16 summary-only
      redistribute connected
!
Bleaf23
!
router bgp 64086.60002
   vrf DEV
      neighbor 10.12.255.1 remote-as 64086.59999
      aggregate-address 10.12.0.0/16 summary-only
      redistribute connected
   vrf PROD
      neighbor 10.14.255.1 remote-as 64086.59999
      aggregate-address 10.14.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.13.255.1 remote-as 64086.59999
      aggregate-address 10.13.0.0/16 summary-only
      redistribute connected
!
Bleaf24
!
router bgp 64086.60002
   vrf DEV
      neighbor 10.4.255.5 remote-as 64086.59999
      aggregate-address 10.4.0.0/16 as-set summary-only
      redistribute connected
   vrf PROD
      neighbor 10.6.255.5 remote-as 64086.59999
      aggregate-address 10.6.0.0/16 summary-only
      redistribute connected
   vrf STAGE
      neighbor 10.5.255.5 remote-as 64086.59999
      aggregate-address 10.5.0.0/16 summary-only
      redistribute connected
!
```
Настроим eBGP на L3
```
!
router bgp 64086.59999
 bgp router-id 172.16.255.255
 bgp asnotation dot
 bgp log-neighbor-changes
 address-family ipv4 vrf DEV
  neighbor BLEAF-POD1 peer-group
  neighbor BLEAF-POD1 remote-as 64086.60001
  neighbor BLEAF-POD2 peer-group
  neighbor BLEAF-POD2 remote-as 64086.60002
  neighbor 10.4.255.2 peer-group BLEAF-POD1
  neighbor 10.4.255.2 activate
  neighbor 10.4.255.6 peer-group BLEAF-POD1
  neighbor 10.4.255.6 activate
  neighbor 10.12.255.2 peer-group BLEAF-POD2
  neighbor 10.12.255.2 activate
  neighbor 10.12.255.6 peer-group BLEAF-POD2
  neighbor 10.12.255.6 activate
  maximum-paths 2
 exit-address-family
 address-family ipv4 vrf PROD
  neighbor POD1-PROD peer-group
  neighbor POD1-PROD remote-as 64086.60001
  neighbor POD2-PROD peer-group
  neighbor POD2-PROD remote-as 64086.60002
  neighbor 10.6.255.2 peer-group POD1-PROD
  neighbor 10.6.255.2 activate
  neighbor 10.6.255.6 peer-group POD1-PROD
  neighbor 10.6.255.6 activate
  neighbor 10.14.255.2 peer-group POD2-PROD
  neighbor 10.14.255.2 activate
  neighbor 10.14.255.6 peer-group POD2-PROD
  neighbor 10.14.255.6 activate
  maximum-paths 2
 exit-address-family
 address-family ipv4 vrf STAGE
  neighbor POD1-STAGE peer-group
  neighbor POD1-STAGE remote-as 64086.60001
  neighbor POD2-STAGE peer-group
  neighbor POD2-STAGE remote-as 64086.60002
  neighbor 10.5.255.2 peer-group POD1-STAGE
  neighbor 10.5.255.2 activate
  neighbor 10.5.255.6 peer-group POD1-STAGE
  neighbor 10.5.255.6 activate
  neighbor 10.13.255.2 peer-group POD2-STAGE
  neighbor 10.13.255.2 activate
  neighbor 10.13.255.6 peer-group POD2-STAGE
  neighbor 10.13.255.6 activate
  maximum-paths 2
 exit-address-family
!
```
## Выводы show commands после настройки Multi-Fabric
**L3**
<details>
<summary>show ip route vrf DEV</summary>

```
L3#show ip route vrf DEV
Gateway of last resort is not set
B*    0.0.0.0/0 [20/0] via 10.4.255.6, 00:29:52
                [20/0] via 10.4.255.2, 00:29:52
      10.0.0.0/8 is variably subnetted, 12 subnets, 4 masks
B        10.4.0.0/16 [20/0] via 10.4.255.6, 00:29:52
                     [20/0] via 10.4.255.2, 00:29:52
C        10.4.255.0/30 is directly connected, GigabitEthernet0/0.2000
L        10.4.255.1/32 is directly connected, GigabitEthernet0/0.2000
C        10.4.255.4/30 is directly connected, GigabitEthernet0/1.2010
L        10.4.255.5/32 is directly connected, GigabitEthernet0/1.2010
B        10.12.0.0/16 [20/0] via 10.12.255.6, 00:19:43
                      [20/0] via 10.12.255.2, 00:19:43
B        10.12.0.0/24 [20/0] via 10.12.255.6, 00:19:43
B        10.12.254.4/30 [20/0] via 10.12.255.6, 00:19:43
C        10.12.255.0/30 is directly connected, GigabitEthernet0/2.2100
L        10.12.255.1/32 is directly connected, GigabitEthernet0/2.2100
C        10.12.255.4/30 is directly connected, GigabitEthernet0/3.2110
L        10.12.255.5/32 is directly connected, GigabitEthernet0/3.2110
```
</details>

<details>
<summary>show ip route vrf STAGE</summary>

```
L3#show ip route vrf STAGE
Gateway of last resort is not set
B*    0.0.0.0/0 [20/0] via 10.5.255.6, 00:31:06
                [20/0] via 10.5.255.2, 00:31:06
      10.0.0.0/8 is variably subnetted, 12 subnets, 4 masks
B        10.5.0.0/16 [20/0] via 10.5.255.6, 00:31:06
                     [20/0] via 10.5.255.2, 00:31:06
C        10.5.255.0/30 is directly connected, GigabitEthernet0/0.2001
L        10.5.255.1/32 is directly connected, GigabitEthernet0/0.2001
C        10.5.255.4/30 is directly connected, GigabitEthernet0/1.2011
L        10.5.255.5/32 is directly connected, GigabitEthernet0/1.2011
B        10.13.0.0/16 [20/0] via 10.13.255.6, 00:06:58
                      [20/0] via 10.13.255.2, 00:06:58
B        10.13.0.0/24 [20/0] via 10.13.255.6, 00:06:58
B        10.13.254.4/30 [20/0] via 10.13.255.6, 00:06:58
C        10.13.255.0/30 is directly connected, GigabitEthernet0/2.2101
L        10.13.255.1/32 is directly connected, GigabitEthernet0/2.2101
C        10.13.255.4/30 is directly connected, GigabitEthernet0/3.2111
L        10.13.255.5/32 is directly connected, GigabitEthernet0/3.2111
```
</details>

<details>
<summary>show ip route vrf PROD</summary>

```
L3#show ip route vrf PROD
Gateway of last resort is not set
B*    0.0.0.0/0 [20/0] via 10.6.255.6, 00:32:05
                [20/0] via 10.6.255.2, 00:32:05
      10.0.0.0/8 is variably subnetted, 12 subnets, 4 masks
B        10.6.0.0/16 [20/0] via 10.6.255.6, 00:32:05
                     [20/0] via 10.6.255.2, 00:32:05
C        10.6.255.0/30 is directly connected, GigabitEthernet0/0.2002
L        10.6.255.1/32 is directly connected, GigabitEthernet0/0.2002
C        10.6.255.4/30 is directly connected, GigabitEthernet0/1.2012
L        10.6.255.5/32 is directly connected, GigabitEthernet0/1.2012
B        10.14.0.0/16 [20/0] via 10.14.255.6, 00:21:56
                      [20/0] via 10.14.255.2, 00:21:56
B        10.14.0.0/24 [20/0] via 10.14.255.6, 00:21:56
B        10.14.254.4/30 [20/0] via 10.14.255.6, 00:21:56
C        10.14.255.0/30 is directly connected, GigabitEthernet0/2.2102
L        10.14.255.1/32 is directly connected, GigabitEthernet0/2.2102
C        10.14.255.4/30 is directly connected, GigabitEthernet0/3.2112
L        10.14.255.5/32 is directly connected, GigabitEthernet0/3.2112
```
</details>

**Leaf11 (POD1)**
<details>
<summary>show ip route vrf DEV</summary>

```
Leaf11#show ip route vrf DEV
Gateway of last resort is not set
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 1000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 1000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.0.0/24 is directly connected, Vlan10
 B I      10.4.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 1000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 1000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B I      10.12.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 1000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                               via VTEP 10.0.0.3 VNI 1000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf STAGE</summary>

```
Leaf11#show ip route vrf STAGE
Gateway of last resort is not set
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 2000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 2000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.5.0.0/24 is directly connected, Vlan11
 B I      10.5.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 2000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 2000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B I      10.13.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 2000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                               via VTEP 10.0.0.3 VNI 2000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf PROD</summary>

```
Leaf11#show ip route vrf PROD
Gateway of last resort is not set
 B I      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 3000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                            via VTEP 10.0.0.3 VNI 3000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        10.6.0.0/24 is directly connected, Vlan20
 B I      10.6.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 3000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                              via VTEP 10.0.0.3 VNI 3000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B I      10.14.0.0/16 [200/0] via VTEP 10.0.0.4 VNI 3000 router-mac 50:00:00:af:d3:f6 local-interface Vxlan1
                               via VTEP 10.0.0.3 VNI 3000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

**Leaf21 (POD2)**
<details>
<summary>show ip route vrf DEV</summary>

```
Leaf21#show ip route vrf DEV
 B I      0.0.0.0/0 [200/0] via VTEP 10.8.0.3 VNI 4000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                            via VTEP 10.8.0.4 VNI 4000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 B I      10.4.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 4000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                              via VTEP 10.8.0.4 VNI 4000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 C        10.12.0.0/24 is directly connected, Vlan110
 B I      10.12.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 4000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                               via VTEP 10.8.0.4 VNI 4000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf STAGE</summary>

```
Leaf21#show ip route vrf STAGE
Gateway of last resort is not set
 B I      0.0.0.0/0 [200/0] via VTEP 10.8.0.3 VNI 5000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                            via VTEP 10.8.0.4 VNI 5000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 B I      10.5.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 5000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                              via VTEP 10.8.0.4 VNI 5000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 C        10.13.0.0/24 is directly connected, Vlan111
 B I      10.13.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 5000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                               via VTEP 10.8.0.4 VNI 5000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf PROD</summary>

```
Leaf21#show ip route vrf PROD
Gateway of last resort is not set
 B I      0.0.0.0/0 [200/0] via VTEP 10.8.0.3 VNI 6000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                            via VTEP 10.8.0.4 VNI 6000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 B I      10.6.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 6000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                              via VTEP 10.8.0.4 VNI 6000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
 C        10.14.0.0/24 is directly connected, Vlan120
 B I      10.14.0.0/16 [200/0] via VTEP 10.8.0.3 VNI 6000 router-mac 50:00:00:d8:ac:19 local-interface Vxlan1
                               via VTEP 10.8.0.4 VNI 6000 router-mac 50:00:00:ba:c6:f8 local-interface Vxlan1
```
</details>

## Тестирование связности между POD1 и POD2

Client6 ping Client1
```
Client6#ping 10.4.0.1 source 10.14.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.14.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 32/33/38 ms
```
Client6 ping Client2
```
Client6#ping 10.5.0.1 source 10.14.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.14.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 32/38/56 ms
```
Client6 ping Client3
```
Client6#ping 10.6.0.1 source 10.14.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.6.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.14.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 28/29/30 ms
```
Client4 ping Client3 (Правильно что нет пинга: меньше security-level)
```
Client4#ping 10.6.0.1 so 10.12.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.6.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.12.0.1 
.....
Success rate is 0 percent (0/5)
```
## Конфигурация АСО
**POD1**
<details>
<summary>Spine11</summary>

```
!
service routing protocols model multi-agent
!
hostname Spine11
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
   10 match as-range 4200000097 result accept
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range 10.2.1.0/24 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF route-reflector-client
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv4
      neighbor LEAF activate
      neighbor LEAF next-hop-self
      network 10.0.1.0/32
!
```
</details>
<details>
<summary>Spine12</summary>

```
!
service routing protocols model multi-agent
!
hostname Spine12
!
spanning-tree mode mstp
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
   10 match as-range 4200000097 result accept
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.2.0
   timers bgp 3 9
   maximum-paths 3
   bgp listen range 10.2.2.0/24 peer-group LEAF peer-filter LEAF_RANGE_ASN
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF route-reflector-client
   neighbor LEAF password 7 SBL80tRxYfD5nL5xXyMQwQ==
   neighbor LEAF send-community extended
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv4
      neighbor LEAF activate
      neighbor LEAF next-hop-self
      network 10.0.2.0/32
!
```
</details>
