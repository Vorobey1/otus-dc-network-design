# Построение Overlay на основе VXLAN EVPN для L2 связности
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для Underlay cети
3. Настроить eBGP для Underlay/Overlay сети
5. Конфигурация АСО
6. Вывод show commands (show bgp evpn, show bgp evpn summaru, show vxlan vtep, show mac address-table)
7. Тестирование связности между клиентами (ping)
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab4/screenshots/Topology.PNG)
## Адресное пространство для Underlay сети
Для выполения данной лабораторной работы будем использовать IPv6. Метод распределения IP-адресов в IPv6 будет аналогичен методу для IPv4, который был изучен на курсе.  
Думаю, что для Underlay необходимо использовать ULA (Unique Local Address). Для простоты (откажемся от правильной генерации сети ULA) используeм следующую сеть в одном ЦОД - FD00::/109.  

Dn - Диапазон в зависимости от номера ЦОДа  
**Dn = [8(N-1)..8(N)-1]**, где N - номер ЦОДа, если N = 1 --> Dn = [0..7]  
**Lo Spine = 8(N-1)**, если N = 1 --> Lo = 0  
**Lo Leaf = 8(N-1)+1**, если N = 1 --> Lo = 1  
**p2p Leaf-Spine = 8(N-1)+2**, если N = 1 --> p2p = 2  
**reserved = 8(N-1)+3**,  если N = 1 --> reserved = 3  
**services = 8(N-1)+[4..7]**,  если N = 1 --> services = [4..7]

**IP Spine = FD00::Dn:Sn00/128**, где Sn - номер Spine  
**IP Leaf = FD00::Dn:00Ln/128**, где Ln - номер Leaf  
**IP p2p = FD00::Dn:Sn2(Ln-1)/127**, где Sn - первые 2 значения в "октете", а 2(Ln-1) - следующие 2 значения в "октете"

После расчета Dn, Sn, Ln полученный результат необходимо будет перевести в 16-ую систему счисления.  

Данный метод распределения IP-адресов позволяет быстро понять с каким типом IP-адреса мы имеем дело, а также можем оптимально сделать суммаризацию.  

Адресация для нашей топологии сети представлена в двух таблицах 
|Device |Loopback     |
|:-----:|:-----------:|
|Spine1 |FD00::100/128|
|Spine2 |FD00::200/128|
|Leaf1  |FD00::1:1/128|
|Leaf2  |FD00::1:2/128|
|Leaf3  |FD00::1:3/128|

|p2p         |Spine1        |Spine2        |
|:----------:|:------------:|:------------:|
|Leaf1       |FD00:2:100/127|FD00:2:200/127|
|Leaf2       |FD00:2:102/127|FD00:2:202/127|
|Leaf3       |FD00:2:104/127|FD00:2:202/127|

Адресация для клиентов

## Настройка eBGP для Underlay сети
При настройке eBGP в топологиях CLOS можно руководствоваться следующим рекомендациям:
1. Использовать схему ASN, которая исключает проблему path-hunting
2. Использовать multipath для ECMP
3. Использовать MP-BGP
4. Использовать BFD, изменить стандартные таймера, настроить аутентификацию

При настройке eBGP для Spine я использовал приватную AS - 64086.60000, для Leaf - 64086.60Ln, где Ln - номер Leaf в диапазоне [001...999]

## Конфигурация АСО
**Spine1**
```
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
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.1.0
   timers bgp 3 9
   maximum-paths 3
   neighbor fd00::2:100 remote-as 64086.60001
   neighbor fd00::2:100 bfd
   neighbor fd00::2:100 password 7 TtEEh4cth76q956T3TrsHg==
   neighbor fd00::2:102 remote-as 64086.60002
   neighbor fd00::2:102 bfd
   neighbor fd00::2:102 password 7 Sl1LPqFEWbAjE25t7N2LXQ==
   neighbor fd00::2:104 remote-as 64086.60003
   neighbor fd00::2:104 bfd
   neighbor fd00::2:104 password 7 2ULH8OJ/jDAeDydnYKfIUA==
   !
   address-family ipv6
      neighbor fd00::2:100 activate
      neighbor fd00::2:102 activate
      neighbor fd00::2:104 activate
      network fd00::100/128
!
```
**Spine2**
```
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
router bgp 64086.60000
   bgp asn notation asdot
   router-id 10.0.2.0
   maximum-paths 3
   neighbor fd00::2:200 remote-as 64086.60001
   neighbor fd00::2:200 bfd
   neighbor fd00::2:200 password 7 HfF+8I6sqbvSWub/PH45YQ==
   neighbor fd00::2:202 remote-as 64086.60002
   neighbor fd00::2:202 bfd
   neighbor fd00::2:202 password 7 zt/vmDxoIUI7cl8z0Ul+bQ==
   neighbor fd00::2:204 remote-as 64086.60003
   neighbor fd00::2:204 bfd
   neighbor fd00::2:204 password 7 vDBozeY7rR6HIcnLP9h5Aw==
   !
   address-family ipv6
      neighbor fd00::2:200 activate
      neighbor fd00::2:202 activate
      neighbor fd00::2:204 activate
      network fd00::200/128
!
```
**Leaf1**
```
!
hostname Leaf1
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:100/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:200/127
!
interface Loopback0
   ipv6 address fd00::1:1/128
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.0.1
   maximum-paths 2
   neighbor fd00::2:101 remote-as 64086.60000
   neighbor fd00::2:101 bfd
   neighbor fd00::2:101 password 7 TtEEh4cth76q956T3TrsHg==
   neighbor fd00::2:201 remote-as 64086.60000
   neighbor fd00::2:201 bfd
   neighbor fd00::2:201 password 7 HfF+8I6sqbvSWub/PH45YQ==
   !
   address-family ipv6
      neighbor fd00::2:101 activate
      neighbor fd00::2:201 activate
      network fd00::1:1/128
!
```
**Leaf2**
```
!
hostname Leaf2
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:102/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:202/127
!
interface Loopback0
   ipv6 address fd00::1:2/128
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60002
   bgp asn notation asdot
   router-id 10.0.0.2
   maximum-paths 2
   neighbor fd00::2:103 remote-as 64086.60000
   neighbor fd00::2:103 bfd
   neighbor fd00::2:103 password 7 Sl1LPqFEWbAjE25t7N2LXQ==
   neighbor fd00::2:203 remote-as 64086.60000
   neighbor fd00::2:203 bfd
   neighbor fd00::2:203 password 7 zt/vmDxoIUI7cl8z0Ul+bQ==
   !
   address-family ipv6
      neighbor fd00::2:103 activate
      neighbor fd00::2:203 activate
      network fd00::1:2/128
!
```
**Leaf3**
```
!
hostname Leaf3
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:104/127
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:204/127
!
interface Loopback0
   ipv6 address fd00::1:3/128
!
no ip routing
!
ipv6 unicast-routing
!
router bgp 64086.60003
   bgp asn notation asdot
   router-id 10.0.0.3
   maximum-paths 2
   neighbor fd00::2:105 remote-as 64086.60000
   neighbor fd00::2:105 bfd
   neighbor fd00::2:105 password 7 2ULH8OJ/jDAeDydnYKfIUA==
   neighbor fd00::2:205 remote-as 64086.60000
   neighbor fd00::2:205 bfd
   neighbor fd00::2:205 password 7 vDBozeY7rR6HIcnLP9h5Aw==
   !
   address-family ipv6
      neighbor fd00::2:105 activate
      neighbor fd00::2:205 activate
      network fd00::1:3/128
!
```
## Вывод show commands
**Spine1**
```
Spine1#show ipv6 bgp 
BGP routing table information for VRF default
Router identifier 10.0.1.0, local AS number 64086.60000
         Network                Next Hop            Metric  LocPref Weight  Path
 * >     fd00::100/128          -                     0       0       -       i
 * >     fd00::1:1/128          fd00::2:100           0       100     0       64086.60001 i
 * >     fd00::1:2/128          fd00::2:102           0       100     0       64086.60002 i
 * >     fd00::1:3/128          fd00::2:104           0       100     0       64086.60003 i
Spine1#show ipv6 bgp summary 
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 64086.60000
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:100      4  64086.60001       507       519    0    0 00:09:31 Estab   1      1
  fd00::2:102      4  64086.60002       280       282    0    0 00:06:05 Estab   1      1
  fd00::2:104      4  64086.60003       273       276    0    0 00:06:02 Estab   1      1
Spine1#show ipv6 route 
 C        fd00::100/128 [0/0]
           via Loopback0, directly connected
 B E      fd00::1:1/128 [200/0]
           via fd00::2:100, Ethernet1
 B E      fd00::1:2/128 [200/0]
           via fd00::2:102, Ethernet2
 B E      fd00::1:3/128 [200/0]
           via fd00::2:104, Ethernet3
 C        fd00::2:100/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:102/127 [0/1]
           via Ethernet2, directly connected
 C        fd00::2:104/127 [0/1]
           via Ethernet3, directly connected
```
**Spine2**
```
Spine2#show ipv6 bgp
BGP routing table information for VRF default
Router identifier 10.0.2.0, local AS number 64086.60000
         Network                Next Hop            Metric  LocPref Weight  Path
 * >     fd00::200/128          -                     0       0       -       i
 * >     fd00::1:1/128          fd00::2:200           0       100     0       64086.60001 i
 * >     fd00::1:2/128          fd00::2:202           0       100     0       64086.60002 i
 * >     fd00::1:3/128          fd00::2:204           0       100     0       64086.60003 i
Spine2#show ipv6 bgp summary 
BGP summary information for VRF default
Router identifier 10.0.2.0, local AS number 64086.60000
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:200      4  64086.60001        67        64    0    0 00:10:18 Estab   1      1
  fd00::2:202      4  64086.60002        55        52    0    0 00:07:37 Estab   1      1
  fd00::2:204      4  64086.60003        47        43    0    0 00:06:57 Estab   1      1
Spine2#show ipv6 route 
 C        fd00::200/128 [0/0]
           via Loopback0, directly connected
 B E      fd00::1:1/128 [200/0]
           via fd00::2:200, Ethernet1
 B E      fd00::1:2/128 [200/0]
           via fd00::2:202, Ethernet2
 B E      fd00::1:3/128 [200/0]
           via fd00::2:204, Ethernet3
 C        fd00::2:200/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:202/127 [0/1]
           via Ethernet2, directly connected
 C        fd00::2:204/127 [0/1]
           via Ethernet3, directly connected
```
**Leaf1**
```
Leaf1#show ipv6 bgp 
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 64086.60001
         Network                Next Hop            Metric  LocPref Weight  Path
 * >     fd00::100/128          fd00::2:101           0       100     0       64086.60000 i
 * >     fd00::200/128          fd00::2:201           0       100     0       64086.60000 i
 * >     fd00::1:1/128          -                     0       0       -       i
 * >Ec   fd00::1:2/128          fd00::2:201           0       100     0       64086.60000 64086.60002 i
 *  ec   fd00::1:2/128          fd00::2:101           0       100     0       64086.60000 64086.60002 i
 * >Ec   fd00::1:3/128          fd00::2:101           0       100     0       64086.60000 64086.60003 i
 *  ec   fd00::1:3/128          fd00::2:201           0       100     0       64086.60000 64086.60003 i
Leaf1#show ipv6 bgp summary 
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 64086.60001
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:101      4  64086.60000       363       371    0    0 00:10:58 Estab   3      3
  fd00::2:201      4  64086.60000        63        69    0    0 00:10:48 Estab   3      3
Leaf1#show ipv6 route 
 B E      fd00::100/128 [200/0]
           via fd00::2:101, Ethernet1
 B E      fd00::200/128 [200/0]
           via fd00::2:201, Ethernet2
 C        fd00::1:1/128 [0/0]
           via Loopback0, directly connected
 B E      fd00::1:2/128 [200/0]
           via fd00::2:101, Ethernet1
           via fd00::2:201, Ethernet2
 B E      fd00::1:3/128 [200/0]
           via fd00::2:101, Ethernet1
           via fd00::2:201, Ethernet2
 C        fd00::2:100/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:200/127 [0/1]
           via Ethernet2, directly connected
```
**Leaf2**
```
Leaf2#show ipv6 bgp 
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 64086.60002
         Network                Next Hop            Metric  LocPref Weight  Path
 * >     fd00::100/128          fd00::2:103           0       100     0       64086.60000 i
 * >     fd00::200/128          fd00::2:203           0       100     0       64086.60000 i
 * >Ec   fd00::1:1/128          fd00::2:203           0       100     0       64086.60000 64086.60001 i
 *  ec   fd00::1:1/128          fd00::2:103           0       100     0       64086.60000 64086.60001 i
 * >     fd00::1:2/128          -                     0       0       -       i
 * >Ec   fd00::1:3/128          fd00::2:103           0       100     0       64086.60000 64086.60003 i
 *  ec   fd00::1:3/128          fd00::2:203           0       100     0       64086.60000 64086.60003 i
Leaf2#show ipv6 bgp summary 
BGP summary information for VRF default
Router identifier 10.0.0.2, local AS number 64086.60002
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:103      4  64086.60000       318       320    0    0 00:07:59 Estab   3      3
  fd00::2:203      4  64086.60000        52        58    0    0 00:08:33 Estab   3      3
Leaf2#show ipv6 route 
 B E      fd00::100/128 [200/0]
           via fd00::2:103, Ethernet1
 B E      fd00::200/128 [200/0]
           via fd00::2:203, Ethernet2
 B E      fd00::1:1/128 [200/0]
           via fd00::2:103, Ethernet1
           via fd00::2:203, Ethernet2
 C        fd00::1:2/128 [0/0]
           via Loopback0, directly connected
 B E      fd00::1:3/128 [200/0]
           via fd00::2:103, Ethernet1
           via fd00::2:203, Ethernet2
 C        fd00::2:102/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:202/127 [0/1]
           via Ethernet2, directly connected
```
**Leaf3**
```
Leaf3#show ipv6 bgp 
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 64086.60003
         Network                Next Hop            Metric  LocPref Weight  Path
 * >     fd00::100/128          fd00::2:105           0       100     0       64086.60000 i
 * >     fd00::200/128          fd00::2:205           0       100     0       64086.60000 i
 * >Ec   fd00::1:1/128          fd00::2:105           0       100     0       64086.60000 64086.60001 i
 *  ec   fd00::1:1/128          fd00::2:205           0       100     0       64086.60000 64086.60001 i
 * >Ec   fd00::1:2/128          fd00::2:105           0       100     0       64086.60000 64086.60002 i
 *  ec   fd00::1:2/128          fd00::2:205           0       100     0       64086.60000 64086.60002 i
 * >     fd00::1:3/128          -                     0       0       -       i
Leaf3#show ipv6 bgp summary 
BGP summary information for VRF default
Router identifier 10.0.0.3, local AS number 64086.60003
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fd00::2:105      4  64086.60000       354       356    0    0 00:08:22 Estab   3      3
  fd00::2:205      4  64086.60000        76        82    0    0 00:08:20 Estab   3      3
Leaf3#show ipv6 route 
 B E      fd00::100/128 [200/0]
           via fd00::2:105, Ethernet1
 B E      fd00::200/128 [200/0]
           via fd00::2:205, Ethernet2
 B E      fd00::1:1/128 [200/0]
           via fd00::2:105, Ethernet1
           via fd00::2:205, Ethernet2
 B E      fd00::1:2/128 [200/0]
           via fd00::2:105, Ethernet1
           via fd00::2:205, Ethernet2
 C        fd00::1:3/128 [0/0]
           via Loopback0, directly connected
 C        fd00::2:104/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:204/127 [0/1]
           via Ethernet2, directly connected
```
## Тестирование доступности Loopbacks
**Leaf1**
```
Leaf1#ping ipv6 fd00::1:2 source fd00::1:1 repeat 1
PING fd00::1:2(fd00::1:2) from fd00::1:1 : 52 data bytes
60 bytes from fd00::1:2: icmp_seq=1 ttl=63 time=5.63 ms
Leaf1#ping ipv6 fd00::1:3 source fd00::1:1 repeat 1
PING fd00::1:3(fd00::1:3) from fd00::1:1 : 52 data bytes
60 bytes from fd00::1:3: icmp_seq=1 ttl=63 time=5.22 ms
```
**Leaf2**
```
Leaf2(config)#ping ipv6 fd00::1:1 source fd00::1:2 repeat 1
PING fd00::1:1(fd00::1:1) from fd00::1:2 : 52 data bytes
60 bytes from fd00::1:1: icmp_seq=1 ttl=63 time=5.66 ms
Leaf2(config)#ping ipv6 fd00::1:3 source fd00::1:2 repeat 1
PING fd00::1:3(fd00::1:3) from fd00::1:2 : 52 data bytes
60 bytes from fd00::1:3: icmp_seq=1 ttl=63 time=6.10 ms
```
**Leaf3**
```
Leaf3#ping ipv6 fd00::1:1 source fd00::1:3 repeat 1
PING fd00::1:1(fd00::1:1) from fd00::1:3 : 52 data bytes
60 bytes from fd00::1:1: icmp_seq=1 ttl=63 time=5.38 ms
Leaf3#ping ipv6 fd00::1:2 source fd00::1:3 repeat 1
PING fd00::1:2(fd00::1:2) from fd00::1:3 : 52 data bytes
60 bytes from fd00::1:2: icmp_seq=1 ttl=63 time=5.27 ms
```

