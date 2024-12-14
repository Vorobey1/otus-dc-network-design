# Проектирование Underlay сети (OSPF)
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространсво для Underlay сети
3. Настроить OSPF для Underlay сети
4. Конфигурация АСО
5. Вывод таблиц маршрутизации
## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/lab2/screenshots/Topology.PNG)
## Адресное пространство для Underlay сети
Адресацию для АСО я переиспользовал из работы "[Проектирование адресного пространства](https://github.com/Vorobey1/otus-dc-network-design/edit/main/lab1/README.md)". Она представлена в следующих двух таблицах  
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
## Настройка OSPF для Underlay сети
При настройке OSPF в топологиях CLOS можно руководствоваться следующим рекомендациям:
1. Использовать тип сети point-to-point для p2p links (При данном типе сети отсутствуют выборы DR/BDR)
2. Использовать команду *passive-interface default*. Для интерфейсов, на которых должны отправляться Hello сообщения, необходимо ввести команду *no passive-interface __interface__*
3. Использовать команду *ip ospf area* вместо *network*
4. Использовать аутентификацию при установлении соседства, что позволяет повысить как безопасность, так и защиту от ошибок, связанных с человеческим фактором
5. Использовать BFD (Даже при использовании прямых линков между соседями BFD может помочь при ситуации, когда линки остаются в состоянии UP, но трафик между ними не передается)
6. Установить reference-bandwidth равным максимальной пропускной способности в домене OSPF или больше (В данной работе я установил этот параметр равным 100000)
## Конфигурация АСО
**Spine1**
```
!
hostname Spine1
!
interface Ethernet1
   no switchport
   ip address 10.2.1.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 OQ62NhxhqcbWEps4eZjZOg==
!
interface Ethernet2
   no switchport
   ip address 10.2.1.3/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Ethernet3
   no switchport
   ip address 10.2.1.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Loopback0
   ip address 10.0.1.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.1.0
   auto-cost reference-bandwidth 100000
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 OQ62NhxhqcbWEps4eZjZOg==
!
interface Ethernet2
   no switchport
   ip address 10.2.2.3/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Ethernet3
   no switchport
   ip address 10.2.2.5/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Loopback0
   ip address 10.0.2.0/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.2.0
   auto-cost reference-bandwidth 100000
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 OQ62NhxhqcbWEps4eZjZOg==
!
interface Ethernet2
   no switchport
   ip address 10.2.2.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Loopback0
   ip address 10.0.0.1/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.0.1
   auto-cost reference-bandwidth 100000
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 OQ62NhxhqcbWEps4eZjZOg==
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Loopback0
   ip address 10.0.0.2/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.0.2
   auto-cost reference-bandwidth 100000
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
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
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 OQ62NhxhqcbWEps4eZjZOg==
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 1 md5 7 WBFBMU6tWMnnTGHupOoTGQ==
!
interface Loopback0
   ip address 10.0.0.3/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.0.0.3
   auto-cost reference-bandwidth 100000
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   max-lsa 12000
!
```
## Вывод таблиц маршрутизации
**Spine1**
**Spine2**
**Leaf1**
**Leaf2**
**Leaf3**

