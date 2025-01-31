# Маршрутизация между VRF через внешнее устройство с использованием маршрутов EVPN route-type 5
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить маршрутизацию между VRF через внешнее устройство
4. Выводы show commands
5. Тестирование связности
6. Конфигурация АСО
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab8/screenshots/Topology.PNG)
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
В качестве внешнего устройства я использовал маршрутизатор Cisco IOSv (R1). R1 будет выполнять Route Leaking между VRF-ами: SERVICE-1 и SERVICE-2. 
Для этого необходимо настроить BGP соседство в каждом VRF между Leaf3/Leaf4 и R1.

Настраиваем VRF
```
Leaf3/Leaf4
!
vrf instance SERVICE-1
vrf instance SERVICE-2
!
ip routing vrf SERVICE-1
ip routing vrf SERVICE-2
!
R1
!
ip vrf SERVICE-1
 rd 10.4.255.255:1
 route-target export 1:1000
 route-target import 1:1000
 route-target import 2:2000
ip vrf SERVICE-2
 rd 10.5.255.255:1
 route-target export 2:2000
 route-target import 2:2000
 route-target import 1:1000
!
```

Настраиваем SVI и сабинтерфейсы на Leaf3/Leaf4 и R1 соответственно
```
Leaf3
!
vlan 1000,10001,2000,2001
!
interface Vlan1000
   vrf SERVICE-1
   ip address 10.4.255.1/30
interface Vlan2000
   vrf SERVICE-2
   ip address 10.5.255.1/30
!
Leaf4
!
vlan 1000,10001,2000,2001
!
interface Vlan1001
   vrf SERVICE-1
   ip address 10.4.255.5/30
interface Vlan2001
   vrf SERVICE-2
   ip address 10.5.255.5/30
!
R1
!
interface GigabitEthernet0/0
 no shutdown         
interface GigabitEthernet0/0.1000
 encapsulation dot1Q 1000
 ip vrf forwarding SERVICE-1
 ip address 10.4.255.2 255.255.255.252
interface GigabitEthernet0/0.2000
 encapsulation dot1Q 2000
 ip vrf forwarding SERVICE-2
 ip address 10.5.255.2 255.255.255.252
!
interface GigabitEthernet0/1
 no shutdown 
interface GigabitEthernet0/1.1001
 encapsulation dot1Q 1001
 ip vrf forwarding SERVICE-1
 ip address 10.4.255.6 255.255.255.252
interface GigabitEthernet0/1.2001
 encapsulation dot1Q 2001
 ip vrf forwarding SERVICE-2
 ip address 10.5.255.6 255.255.255.252
!
```

На BorderLeaf (Leaf3, Leaf4) создаем все l2vni и l3vni в нашем POD
```
!
interface Vxlan1
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf SERVICE-1 vni 1000
   vxlan vrf SERVICE-2 vni 2000
!
router bgp 64086.60003
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
   vrf SERVICE-1
      rd 10.0.0.3:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
   vrf SERVICE-2
      rd 10.0.0.3:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
!
```

Настраиваем eBGP в каждом VRF на Leaf3 и Leaf4.  Для уменьшения маршрутной информации настроим route-map, которая будет убирать из аннонсов префиксы /32
```
Leaf3
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60003
   vrf SERVICE-1
      neighbor 10.4.255.2 remote-as 64086.59999
      neighbor 10.4.255.2 route-map EX_MACIP_RM out
      redistribute connected
   vrf SERVICE-2
      neighbor 10.5.255.2 remote-as 64086.59999
      neighbor 10.5.255.2 route-map EX_MACIP_RM out
      redistribute connected
!
Leaf4
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60003
   vrf SERVICE-1
      neighbor 10.4.255.6 remote-as 64086.59999
      neighbor 10.4.255.6 route-map EX_MACIP_RM out
      redistribute connected
   vrf SERVICE-2
      neighbor 10.5.255.6 remote-as 64086.59999
      neighbor 10.5.255.6 route-map EX_MACIP_RM out
      redistribute connected
!
```

Настроим eBGP на R1
```
!
router bgp 64086.59999
 bgp router-id 172.16.255.255
 bgp asnotation dot
 bgp log-neighbor-changes
 !
 address-family ipv4 vrf SERVICE-1
  neighbor BLEAF peer-group
  neighbor BLEAF remote-as 64086.60003
  neighbor BLEAF as-override
  neighbor 10.4.255.1 peer-group BLEAF
  neighbor 10.4.255.1 activate
  neighbor 10.4.255.5 peer-group BLEAF
  neighbor 10.4.255.5 activate
  maximum-paths 2
 exit-address-family
 !
 address-family ipv4 vrf SERVICE-2
  neighbor BLEAF peer-group
  neighbor BLEAF remote-as 64086.60003
  neighbor BLEAF as-override
  neighbor 10.5.255.1 peer-group BLEAF
  neighbor 10.5.255.1 activate
  neighbor 10.5.255.5 peer-group BLEAF
  neighbor 10.5.255.5 activate
  maximum-paths 2
 exit-address-family
!
```

## Выводы show commands
**R1**
<details>
<summary>show ip route vrf SERVICE-1</summary>

```
R1#show ip route vrf SERVICE-1
Gateway of last resort is not set
      10.0.0.0/8 is variably subnetted, 9 subnets, 3 masks
B        10.4.0.0/24 [20/0] via 10.4.255.5, 00:45:53
                     [20/0] via 10.4.255.1, 00:45:53
C        10.4.255.0/30 is directly connected, GigabitEthernet0/0.1000
L        10.4.255.2/32 is directly connected, GigabitEthernet0/0.1000
C        10.4.255.4/30 is directly connected, GigabitEthernet0/1.1001
L        10.4.255.6/32 is directly connected, GigabitEthernet0/1.1001
B        10.5.0.0/24 [20/0] via 10.5.255.1 (SERVICE-2), 00:42:54
B        10.5.1.0/24 [20/0] via 10.5.255.1 (SERVICE-2), 00:42:54
B        10.5.255.0/30 [20/0] via 10.5.255.1 (SERVICE-2), 00:42:54
B        10.5.255.4/30 [20/0] via 10.5.255.5 (SERVICE-2), 00:42:43
```
</details>

<details>
<summary>show ip route vrf SERVICE-2</summary>
 
```
R1#show ip route vrf SERVICE-2
Gateway of last resort is not set
      10.0.0.0/8 is variably subnetted, 9 subnets, 3 masks
B        10.4.0.0/24 [20/0] via 10.4.255.5 (SERVICE-1), 00:51:15
B        10.4.255.0/30 [20/0] via 10.4.255.1 (SERVICE-1), 00:51:15
B        10.4.255.4/30 [20/0] via 10.4.255.5 (SERVICE-1), 00:51:15
B        10.5.0.0/24 [20/0] via 10.5.255.5, 00:42:48
                     [20/0] via 10.5.255.1, 00:42:48
B        10.5.1.0/24 [20/0] via 10.5.255.5, 00:42:48
                     [20/0] via 10.5.255.1, 00:42:48
C        10.5.255.0/30 is directly connected, GigabitEthernet0/0.2000
L        10.5.255.2/32 is directly connected, GigabitEthernet0/0.2000
C        10.5.255.4/30 is directly connected, GigabitEthernet0/1.2001
L        10.5.255.6/32 is directly connected, GigabitEthernet0/1.2001
```
</details>

**Leaf1**
<details>
<summary>show ip route vrf SERVICE-1</summary>

```
Leaf1#show ip route vrf SERVICE-1
Gateway of last resort is not set
 B E      10.4.0.3/32 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 C        10.4.0.0/24 is directly connected, Vlan10
 B E      10.4.255.0/30 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.255.4/30 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.0.0/24 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.1.0/24 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.255.0/30 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.255.4/30 [200/0] via VTEP 10.1.3.4 VNI 1000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show ip route vrf SERVICE-2</summary>

```
Leaf1#show ip route vrf SERVICE-2
Gateway of last resort is not set
 B E      10.4.0.0/24 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.255.0/30 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.4.255.4/30 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 C        10.5.0.0/24 is directly connected, Vlan11
 B E      10.5.1.1/32 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.1.0/24 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.255.0/30 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
 B E      10.5.255.4/30 [200/0] via VTEP 10.1.3.4 VNI 2000 router-mac 52:00:00:15:f4:e8 local-interface Vxlan1
```
</details>

<details>
<summary>show bgp evpn route-type ip-prefix ipv4</summary>

```
Leaf1#show bgp evpn route-type ip-prefix ipv4 
          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.4.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.4.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.4.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.5.0.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.5.1.0/24
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.5.255.0/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:1000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:1000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.3:2000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.3:2000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:1000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 *  ec    RD: 10.0.0.4:1000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 64086.59999 64086.59999 i
 * >Ec    RD: 10.0.0.4:2000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
 *  ec    RD: 10.0.0.4:2000 ip-prefix 10.5.255.4/30
                                 10.1.3.4              -       100     0       64086.60000 64086.60003 i
```
</details>

<details>
<summary>show bgp evpn route-type ip-prefix 10.4.0.0/24 </summary>

```
Leaf1#show bgp evpn route-type ip-prefix 10.4.0.0/24 
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 4200000097
BGP routing table entry for ip-prefix 10.4.0.0/24, Route Distinguisher: 10.0.0.3:1000
 Paths: 2 available
  64086.60000 64086.60003
    10.1.3.4 from 10.2.2.1 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:1000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 1000
  64086.60000 64086.60003
    10.1.3.4 from 10.2.1.1 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:1000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 1000
BGP routing table entry for ip-prefix 10.4.0.0/24, Route Distinguisher: 10.0.0.3:2000
 Paths: 2 available
  64086.60000 64086.60003 64086.59999 64086.59999
    10.1.3.4 from 10.2.1.1 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:2000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 2000
  64086.60000 64086.60003 64086.59999 64086.59999
    10.1.3.4 from 10.2.2.1 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:2000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 2000
BGP routing table entry for ip-prefix 10.4.0.0/24, Route Distinguisher: 10.0.0.4:1000
 Paths: 2 available
  64086.60000 64086.60003
    10.1.3.4 from 10.2.1.1 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:1:1000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 1000
  64086.60000 64086.60003
    10.1.3.4 from 10.2.2.1 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:1:1000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 1000
BGP routing table entry for ip-prefix 10.4.0.0/24, Route Distinguisher: 10.0.0.4:2000
 Paths: 2 available
  64086.60000 64086.60003 64086.59999 64086.59999
    10.1.3.4 from 10.2.2.1 (10.0.2.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:2:2000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 2000
  64086.60000 64086.60003 64086.59999 64086.59999
    10.1.3.4 from 10.2.1.1 (10.0.1.0)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:2:2000 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:52:00:00:15:f4:e8
      VNI: 2000
```
</details>

## Тестирование связности
Client1 ping Client3
```
Client1#ping 10.4.0.3 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.4.0.3, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 13/14/16 ms
```
Client1 ping Client2
```
Client1#ping 10.5.0.1 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.0.1, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 25/27/31 ms
```
Client1 ping Client4
```
Client1#ping 10.5.1.1 source 10.4.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.1.1, timeout is 2 seconds:
Packet sent with a source address of 10.4.0.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 18/24/46 ms
```
Client1 traceroute Client2
```
Client1#traceroute 10.5.0.1
Type escape sequence to abort.
Tracing the route to 10.5.0.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.4.0.253 4 msec 4 msec 4 msec
  2 10.4.255.1 12 msec 11 msec 11 msec
  3 10.4.255.2 14 msec 14 msec 15 msec
  4 10.5.255.1 15 msec 15 msec 16 msec
  5 10.5.0.252 27 msec 34 msec 27 msec
  6 10.5.0.1 33 msec 27 msec * 
```

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
!
service routing protocols model multi-agent
!
link tracking group MLAG_UPLINK_TRACK
   recovery delay 60
!
hostname Leaf3
!
no spanning-tree vlan-id 4094
!
vlan 10-11,20,1001,2001
!
vlan 1000
   name SERVICE-1
!
vlan 2000
   name SERVICE-2
!
vlan 4094
   name MLAG
   trunk group MLAGPEER
!
vrf instance SERVICE-1
!
vrf instance SERVICE-2
!
interface Port-Channel3
   description Client3
   switchport access vlan 10
   mlag 3
   spanning-tree portfast
   spanning-tree bpdufilter enable
   link tracking group MLAG_UPLINK_TRACK downstream
!
interface Port-Channel4
   description Client4
   switchport access vlan 20
   mlag 4
   spanning-tree portfast
   spanning-tree bpdufilter enable
   link tracking group MLAG_UPLINK_TRACK downstream
!
interface Port-Channel1000
   description MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAGPEER
!
interface Ethernet1
   no switchport
   ip address 10.2.1.4/31
   link tracking group MLAG_UPLINK_TRACK upstream
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
   link tracking group MLAG_UPLINK_TRACK upstream
!
interface Ethernet3
   switchport trunk allowed vlan 1000,2000
   switchport mode trunk
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
   vrf SERVICE-1
   ip address 10.4.0.251/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE-2
   ip address 10.5.0.251/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf SERVICE-2
   ip address 10.5.1.253/24
   ip virtual-router address 10.5.1.254
!
interface Vlan1000
   vrf SERVICE-1
   ip address 10.4.255.1/30
!
interface Vlan2000
   vrf SERVICE-2
   ip address 10.5.255.1/30
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
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
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
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
mlag configuration
   domain-id 1000
   local-interface Vlan4094
   peer-address 192.168.0.2
   peer-link Port-Channel1000
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
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
      network 10.0.0.3/32
      network 10.1.3.4/32
   !
   vrf SERVICE-1
      rd 10.0.0.3:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.255.2 remote-as 64086.59999
      neighbor 10.4.255.2 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf SERVICE-2
      rd 10.0.0.3:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.255.2 remote-as 64086.59999
      neighbor 10.5.255.2 route-map EX_MACIP_RM out
      redistribute connected
!
```
</details>

<details> 
<summary>Leaf4</summary>
   
```
!
service routing protocols model multi-agent
!
link tracking group MLAG_UPLINK_TRACK
   recovery delay 60
!
hostname Leaf4
!
no spanning-tree vlan-id 4094
!
vlan 10-11,20,1000,2000
!
vlan 1001
   name SERVICE-1
!
vlan 2001
   name SERVICE-2
!
vlan 4094
   name MLAG
   trunk group MLAGPEER
!
vrf instance SERVICE-1
!
vrf instance SERVICE-2
!
interface Port-Channel3
   description Client3
   switchport access vlan 10
   mlag 3
   spanning-tree portfast
   spanning-tree bpdufilter enable
   link tracking group MLAG_UPLINK_TRACK downstream
!
interface Port-Channel4
   description Client4
   switchport access vlan 20
   mlag 4
   spanning-tree portfast
   spanning-tree bpdufilter enable
   link tracking group MLAG_UPLINK_TRACK downstream
!
interface Port-Channel1000
   description MLAG Peer-Link
   switchport mode trunk
   switchport trunk group MLAGPEER
!
interface Ethernet1
   shutdown
   no switchport
   ip address 10.2.1.6/31
   link tracking group MLAG_UPLINK_TRACK upstream
!
interface Ethernet2
   shutdown
   no switchport
   ip address 10.2.2.6/31
   link tracking group MLAG_UPLINK_TRACK upstream
!
interface Ethernet3
   switchport trunk allowed vlan 1001,2001
   switchport mode trunk
   link tracking group MLAG_UPLINK_TRACK downstream
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
   vrf SERVICE-1
   ip address 10.4.0.250/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf SERVICE-2
   ip address 10.5.0.250/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf SERVICE-2
   ip address 10.5.1.252/24
   ip virtual-router address 10.5.1.254
!
interface Vlan1001
   vrf SERVICE-1
   ip address 10.4.255.5/30
!
interface Vlan2001
   vrf SERVICE-2
   ip address 10.5.255.5/30
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
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
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
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
mlag configuration
   domain-id 1000
   local-interface Vlan4094
   peer-address 192.168.0.1
   peer-link Port-Channel1000
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
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
   neighbor 10.2.1.7 peer group SPINE
   neighbor 10.2.2.7 peer group SPINE
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
      network 10.0.0.4/32
      network 10.1.3.4/32
   !
   vrf SERVICE-1
      rd 10.0.0.4:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.255.6 remote-as 64086.59999
      neighbor 10.4.255.6 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf SERVICE-2
      rd 10.0.0.4:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.255.6 remote-as 64086.59999
      neighbor 10.5.255.6 route-map EX_MACIP_RM out
      redistribute connected
!
```
</details>

<details> 
<summary>R1</summary>
   
```
!
hostname R1
!
ip vrf SERVICE-1
 rd 10.4.255.255:1
 route-target export 1:1000
 route-target import 1:1000
 route-target import 2:2000
!
ip vrf SERVICE-2
 rd 10.5.255.255:1
 route-target export 2:2000
 route-target import 2:2000
 route-target import 1:1000
!
interface Loopback0
 ip address 172.16.255.255 255.255.255.255
!        
interface GigabitEthernet0/0.1000
 encapsulation dot1Q 1000
 ip vrf forwarding SERVICE-1
 ip address 10.4.255.2 255.255.255.252
!
interface GigabitEthernet0/0.2000
 encapsulation dot1Q 2000
 ip vrf forwarding SERVICE-2
 ip address 10.5.255.2 255.255.255.252
!
interface GigabitEthernet0/1.1001
 encapsulation dot1Q 1001
 ip vrf forwarding SERVICE-1
 ip address 10.4.255.6 255.255.255.252
!
interface GigabitEthernet0/1.2001
 encapsulation dot1Q 2001
 ip vrf forwarding SERVICE-2
 ip address 10.5.255.6 255.255.255.252
!
router bgp 64086.59999
 bgp router-id 172.16.255.255
 bgp asnotation dot
 bgp log-neighbor-changes
 !
 address-family ipv4 vrf SERVICE-1
  neighbor BLEAF peer-group
  neighbor BLEAF remote-as 64086.60003
  neighbor BLEAF as-override
  neighbor 10.4.255.1 peer-group BLEAF
  neighbor 10.4.255.1 activate
  neighbor 10.4.255.5 peer-group BLEAF
  neighbor 10.4.255.5 activate
  maximum-paths 2
 exit-address-family
 !
 address-family ipv4 vrf SERVICE-2
  neighbor BLEAF peer-group
  neighbor BLEAF remote-as 64086.60003
  neighbor BLEAF as-override
  neighbor 10.5.255.1 peer-group BLEAF
  neighbor 10.5.255.1 activate
  neighbor 10.5.255.5 peer-group BLEAF
  neighbor 10.5.255.5 activate
  maximum-paths 2
 exit-address-family
!
```
</details>
