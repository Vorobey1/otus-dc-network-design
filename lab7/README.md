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
## Настройка MLAG

## Конфигурация АСО

## Конфигурация

