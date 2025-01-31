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
<summary>Spine1</summary>
   
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
</details>
