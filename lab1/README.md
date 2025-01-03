# Проектирование адресного пространства
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для Underlay cети
3. Настроить адресацию на оборудовании
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab1/screenshots/Topology.PNG)
## Адресное пространство для Underlay сети
В качестве метода распределения IP-адресов используется способ, изученный на курсе 

Dn - Диапазон в зависимости от номера ЦОДа  
**Dn = [8(N-1)..8(N)-1]**, где N - номер ЦОДа, если N = 1 --> Dn = [0..7]  
**Lo Spine = 8(N-1)**, если N = 1 --> Lo = 0  
**Lo Leaf = 8(N-1)+1**, если N = 1 --> Lo = 1  
**p2p Leaf-Spine = 8(N-1)+2**, если N = 1 --> p2p = 2  
**reserved = 8(N-1)+3**,  если N = 1 --> reserved = 3  
**services = 8(N-1)+[4..7]**,  если N = 1 --> services = [4..7]

**IP Spine = 10.Dn.Sn.0/32**, где Sn - номер Spine  
**IP Leaf = 10.Dn.0.Ln/32**, где Ln - номер Leaf  
**IP p2p - 10.Dn.Sn.2(Ln-1)/31**  

Данный метод распределения IP-адресов позволяет быстро понять с каким типом IP-адреса мы имеем дело, а также можем оптимально сделать суммаризацию. Его  следует применять на больших ЦОДах, при проектировании маленького ЦОДа (как наш лабораторный POD) используемая адресация является неоптимальной, однако мы будем использовать именно ее для лучшего закрепления изученного материала.  

Адресация для нашей топологии сети представлена в двух таблицах  
|Device |Loopback    |
|:-----:|:----------:|
|Spine1 |10.0.1.0/32 |
|Spine2 |10.0.2.0/32 |
|Leaf1  |10.0.0.1/32 |
|Leaf2  |10.0.0.2/32 |
|Leaf3  |10.0.0.3/32 |

|p2p  |Spine1      |Spine2      |
|:----------:|:----------:|:----------:|
|Leaf1       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf2       |10.2.1.2/31 |10.2.2.2/31 |
|Leaf3       |10.2.1.4/31 |10.2.2.4/31 |

## Настройки адресации на оборудовании
**Spine1**
```
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
```
**Spine2**
```
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
```
**Leaf1**
```
!
hostname Leaf1
!
interface Ethernet1
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.0/31
!
interface Loopback0
   ip address 10.0.0.1/32
!
ip routing
!
```
**Leaf2**
```
!
hostname Leaf2
!
interface Ethernet1
   no switchport
   ip address 10.2.1.2/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/31
!
interface Loopback0
   ip address 10.0.0.2/32
!
ip routing
!
```
**Leaf3**
```
!
hostname Leaf3
!
interface Ethernet1
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
!
interface Loopback0
   ip address 10.0.0.3/32
!
ip routing
!
```
