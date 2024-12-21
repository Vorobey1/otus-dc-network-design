# Построение Underlay сети (ISIS)
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для Underlay cети
3. Настроить ISIS для Underlay сети
4. Конфигурация АСО
5. Вывод show commands (LSDB, Neighbors, ipv6 route)
6. Тестирование доступности Loopbacks (Ping)
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab3/screenshots/Topology.PNG)
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

## Настройка ISIS для Underlay сети
При настройке ISIS в топологиях CLOS можно руководствоваться следующим рекомендациям:
1. Использовать тип сети point-to-point для p2p links (При данном типе сети отсутствуют выборы DIS)
2. В одном POD достаточно соединить все линками L1
3. Использовать аутентификацию при установлении соседства, что позволяет повысить как безопасность, так и защиту от ошибок, связанных с человеческим фактором
4. Использовать BFD (Даже при использовании прямых линков между соседями BFD может помочь при ситуации, когда линки остаются в состоянии UP, но трафик между ними не передается)  

Для настройке net я использовал следующий способ:  
Для Spine net - 49.PODn.PODn.Sn.0000.00  
Для Leaf net - 49.PODn.PODn.0000.Ln.00  

## Конфигурация АСО
**Spine1**
```
!
hostname Spine1
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:101/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:103/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet3
   no switchport
   ipv6 address fd00::2:105/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Loopback0
   ipv6 address fd00::100/128
   isis enable POD1
!
no ip routing
!
ipv6 unicast-routing
!
router isis POD1
   net 49.0001.0001.0001.0000.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv6 unicast
      bfd all-interfaces
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
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:203/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet3
   no switchport
   ipv6 address fd00::2:205/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Loopback0
   ipv6 address fd00::200/128
   isis enable POD1
!
no ip routing
!
ipv6 unicast-routing
!
router isis POD1
   net 49.0001.0001.0002.0000.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv6 unicast
      bfd all-interfaces
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
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:200/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Loopback0
   ipv6 address fd00::1:1/128
   isis enable POD1
!
no ip routing
!
ipv6 unicast-routing
!
router isis POD1
   net 49.0001.0001.0000.0001.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv6 unicast
      bfd all-interfaces
!
```
**Leaf2**
```
!
hostname Leaf2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ipv6 address fd00::2:102/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:202/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Loopback0
   ipv6 address fd00::1:2/128
   isis enable POD1
!
no ip routing
!
ipv6 unicast-routing
!
router isis POD1
   net 49.0001.0001.0000.0002.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv6 unicast
      bfd all-interfaces
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
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Ethernet2
   no switchport
   ipv6 address fd00::2:204/127
   isis enable POD1
   isis circuit-type level-1
   isis network point-to-point
   isis authentication mode md5
   isis authentication key 7 2pd/xh7ZVL71OF4V0/L3hA==
!
interface Loopback0
   ipv6 address fd00::1:3/128
   isis enable POD1
!
no ip routing
!
ipv6 unicast-routing
!
router isis POD1
   net 49.0001.0001.0000.0003.00
   is-type level-1
   log-adjacency-changes
   !
   address-family ipv6 unicast
      bfd all-interfaces
!
```
## Вывод show commands
**Spine1**
```
Spine1#show isis database 

IS-IS Instance: POD1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf1.00-00                  11  29072  1177    196 L1 <>
    Leaf2.00-00                   3  55332   556    196 L1 <>
    Leaf3.00-00                   4   2532   627    196 L1 <>
    Spine1.00-00                  8  46572   713    246 L1 <>
    Spine2.00-00                  9  57532  1137    246 L1 <>
Spine1#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
POD1      default  Leaf1            L1   Ethernet1          P2P               UP    26          0D                  
POD1      default  Leaf2            L1   Ethernet2          P2P               UP    25          0D                  
POD1      default  Leaf3            L1   Ethernet3          P2P               UP    28          0D                  
Spine1#show ipv6 route

 C        fd00::100/128 [0/0]
           via Loopback0, directly connected
 I L1     fd00::200/128 [115/30]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
           via fe80::5200:ff:fe03:3766, Ethernet2
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 I L1     fd00::1:1/128 [115/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 I L1     fd00::1:2/128 [115/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 I L1     fd00::1:3/128 [115/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 C        fd00::2:100/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:102/127 [0/1]
           via Ethernet2, directly connected
 C        fd00::2:104/127 [0/1]
           via Ethernet3, directly connected
 I L1     fd00::2:200/127 [115/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 I L1     fd00::2:202/127 [115/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 I L1     fd00::2:204/127 [115/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
```
**Spine2**
```
Spine2#show isis database 

IS-IS Instance: POD1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf1.00-00                   8  30605   701    196 L1 <>
    Leaf2.00-00                   3  55332   857    196 L1 <>
    Leaf3.00-00                   4   2532   927    196 L1 <>
    Spine1.00-00                  8  46572  1014    246 L1 <>
    Spine2.00-00                  5  10616   565    246 L1 <>
Spine2#show isis neighbors
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
POD1      default  Leaf1            L1   Ethernet1          P2P               UP    29          0E                  
POD1      default  Leaf2            L1   Ethernet2          P2P               UP    28          0E                  
POD1      default  Leaf3            L1   Ethernet3          P2P               UP    29          0E                  
Spine2#show ipv6 route 

 I L1     fd00::100/128 [115/30]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
           via fe80::5200:ff:fe03:3766, Ethernet2
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 C        fd00::200/128 [0/0]
           via Loopback0, directly connected
 I L1     fd00::1:1/128 [115/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 I L1     fd00::1:2/128 [115/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 I L1     fd00::1:3/128 [115/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 I L1     fd00::2:100/127 [115/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet1
 I L1     fd00::2:102/127 [115/20]
           via fe80::5200:ff:fe03:3766, Ethernet2
 I L1     fd00::2:104/127 [115/20]
           via fe80::5200:ff:fe15:f4e8, Ethernet3
 C        fd00::2:200/127 [0/1]
           via Ethernet1, directly connected
 C        fd00::2:202/127 [0/1]
           via Ethernet2, directly connected
 C        fd00::2:204/127 [0/1]
           via Ethernet3, directly connected
```
**Leaf1**
```
Leaf1#show isis database 

IS-IS Instance: POD1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf1.00-00                   8  30605   761    196 L1 <>
    Leaf2.00-00                   3  55332   917    196 L1 <>
    Leaf3.00-00                   4   2532   988    196 L1 <>
    Spine1.00-00                  8  46572  1074    246 L1 <>
    Spine2.00-00                  5  10616   625    246 L1 <>
Leaf1#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
POD1      default  Spine1           L1   Ethernet1          P2P               UP    24          0D                  
POD1      default  Spine2           L1   Ethernet2          P2P               UP    23          0D                  
Leaf1#show ipv6 route 

 I L1     fd00::100/128 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::200/128 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::1:1/128 [0/0]
           via Loopback0, directly connected
 I L1     fd00::1:2/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::1:3/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::2:100/127 [0/1]
           via Ethernet1, directly connected
 I L1     fd00::2:102/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::2:104/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 C        fd00::2:200/127 [0/1]
           via Ethernet2, directly connected
 I L1     fd00::2:202/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::2:204/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
```
**Leaf2**
```
Leaf2#show isis database 

IS-IS Instance: POD1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf1.00-00                   8  30605   803    196 L1 <>
    Leaf2.00-00                   3  55332   959    196 L1 <>
    Leaf3.00-00                   4   2532  1029    196 L1 <>
    Spine1.00-00                  8  46572  1116    246 L1 <>
    Spine2.00-00                  5  10616   667    246 L1 <>
Leaf2#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
POD1      default  Spine1           L1   Ethernet1          P2P               UP    23          0E                  
POD1      default  Spine2           L1   Ethernet2          P2P               UP    29          0E                  
Leaf2#show ipv6 route 

 I L1     fd00::100/128 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::200/128 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::1:1/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::1:2/128 [0/0]
           via Loopback0, directly connected
 I L1     fd00::1:3/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::2:100/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 C        fd00::2:102/127 [0/1]
           via Ethernet1, directly connected
 I L1     fd00::2:104/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::2:200/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::2:202/127 [0/1]
           via Ethernet2, directly connected
 I L1     fd00::2:204/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
```
**Leaf3**
```
Leaf3#show isis database 

IS-IS Instance: POD1 VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS Flags
    Leaf1.00-00                   8  30605   868    196 L1 <>
    Leaf2.00-00                   3  55332  1023    196 L1 <>
    Leaf3.00-00                   4   2532  1094    196 L1 <>
    Spine1.00-00                  8  46572  1181    246 L1 <>
    Spine2.00-00                  5  10616   731    246 L1 <>
Leaf3#show isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
POD1      default  Spine1           L1   Ethernet1          P2P               UP    24          0F                  
POD1      default  Spine2           L1   Ethernet2          P2P               UP    29          0F
Leaf3#show ipv6 route

 I L1     fd00::100/128 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::200/128 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::1:1/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::1:2/128 [115/30]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::1:3/128 [0/0]
           via Loopback0, directly connected
 I L1     fd00::2:100/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 I L1     fd00::2:102/127 [115/20]
           via fe80::5200:ff:fed7:ee0b, Ethernet1
 C        fd00::2:104/127 [0/1]
           via Ethernet1, directly connected
 I L1     fd00::2:200/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 I L1     fd00::2:202/127 [115/20]
           via fe80::5200:ff:fecb:38c2, Ethernet2
 C        fd00::2:204/127 [0/1]
           via Ethernet2, directly connected
```
## Тестирование доступности Loopbacks
**Spine1**
```
Spine1#ping fd00::200 source lo0 repeat 1
PING fd00::200(fd00::200) from fd00::100 : 52 data bytes
60 bytes from fd00::200: icmp_seq=1 ttl=63 time=5.64 ms

Spine1#ping fd00::1:1 source lo0 repeat 1
PING fd00::1:1(fd00::1:1) from fd00::100 : 52 data bytes
60 bytes from fd00::1:1: icmp_seq=1 ttl=64 time=2.70 ms

Spine1#ping fd00::1:2 source lo0 repeat 1
PING fd00::1:2(fd00::1:2) from fd00::100 : 52 data bytes
60 bytes from fd00::1:2: icmp_seq=1 ttl=64 time=4.34 ms

Spine1#ping fd00::1:3 source lo0 repeat 1
PING fd00::1:3(fd00::1:3) from fd00::100 : 52 data bytes
60 bytes from fd00::1:3: icmp_seq=1 ttl=64 time=3.05 ms
```
**Leaf3**
```
Leaf3#ping fd00::100 source fd00::1:3 repeat 1
PING fd00::100(fd00::100) from fd00::1:3 : 52 data bytes
60 bytes from fd00::100: icmp_seq=1 ttl=64 time=2.86 ms

Leaf3#ping fd00::200 source fd00::1:3 repeat 1
PING fd00::200(fd00::200) from fd00::1:3 : 52 data bytes
60 bytes from fd00::200: icmp_seq=1 ttl=64 time=3.04 ms

Leaf3#ping fd00::1:1 source fd00::1:3 repeat 1
PING fd00::1:1(fd00::1:1) from fd00::1:3 : 52 data bytes
60 bytes from fd00::1:1: icmp_seq=1 ttl=63 time=5.34 ms

Leaf3#ping fd00::1:2 source fd00::1:3 repeat 1
PING fd00::1:2(fd00::1:2) from fd00::1:3 : 52 data bytes
60 bytes from fd00::1:2: icmp_seq=1 ttl=63 time=5.41 ms
```
