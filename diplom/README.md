# Проектирование сегментированной сети в датацентре на базе EVPN\VXLAN c использованием Multi-Fabric
## План работ
1. Собрать топологию сети CLOS
2. Распределить адресное пространство для cети
3. Настроить маршрутизацию между VRF FW
4. Настроить Multi-Fabric
5. Выводы show commands
6. Тестирование связности
7. Конфигурация АСО

## Топология сети CLOS
Топология сети была собрана в эмуляторе EVE-NG. В качестве оборудования Leaf и Spine используется AristaEOS, FW - ASAv, L3VPN - Cisco vIOS Router.

![alt-текст](https://github.com/Vorobey1/otus-dc-network-design/blob/main/diplom/screenshots/Topology.PNG)  
## Адресное пространство для сети
Адресацию для АСО я переиспользовал из работы "[Проектирование адресного пространства](https://github.com/Vorobey1/otus-dc-network-design/edit/main/lab1/README.md)".

<details>
<summary>Адресное пространство POD1</summary>

|Device  |Loopback    |
|:------:|:----------:|
|Spine11 |10.0.1.0/32 |
|Spine12 |10.0.2.0/32 |
|Leaf11  |10.0.0.1/32 |
|Leaf12  |10.0.0.2/32 |
|Leaf13  |10.0.0.3/32 |
|Leaf14  |10.0.0.4/32 |

|p2p          |Spine11     |Spine12     |
|:-----------:|:----------:|:----------:|
|Leaf11       |10.2.1.0/31 |10.2.2.0/31 |
|Leaf12       |10.2.1.2/31 |10.2.2.2/31 |
|Leaf13       |10.2.1.4/31 |10.2.2.4/31 |
|Leaf14       |10.2.1.6/31 |10.2.2.6/31 |

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

|Device  |Loopback    |
|:------:|:----------:|
|Spine21 |10.8.1.0/32 |
|Spine22 |10.8.2.0/32 |
|Leaf21  |10.8.0.1/32 |
|Leaf22  |10.8.0.2/32 |
|Leaf23  |10.8.0.3/32 |
|Leaf24  |10.8.0.4/32 |

|p2p          |Spine21      |Spine22      |
|:-----------:|:-----------:|:-----------:|
|Leaf21       |10.10.1.0/31 |10.10.2.0/31 |
|Leaf22       |10.10.1.2/31 |10.10.2.2/31 |
|Leaf23       |10.10.1.4/31 |10.10.2.4/31 |
|Leaf24       |10.10.1.6/31 |10.10.2.6/31 |

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

## Настроийка маршрутизации между VRF чере Firewall

**FW1**
```
!
hostname FW1
names
no mac-address auto
zone DEV
zone STAGE
zone PROD
!
interface GigabitEthernet0/0
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/0.3000
 vlan 3000
 nameif DEV1
 security-level 90
 zone-member DEV
 ip address 10.4.254.1 255.255.255.252 
!
interface GigabitEthernet0/0.3001
 vlan 3001
 nameif STAGE1
 security-level 90
 zone-member STAGE
 ip address 10.5.254.1 255.255.255.252 
!
interface GigabitEthernet0/0.3002
 vlan 3002
 nameif PROD1
 security-level 90
 zone-member PROD
 ip address 10.6.254.1 255.255.255.252 
!
interface GigabitEthernet0/1
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/1.4000
 vlan 4000
 nameif DEV2
 security-level 90
 zone-member DEV
 ip address 10.4.254.5 255.255.255.252 
!
interface GigabitEthernet0/1.4001
 vlan 4001
 nameif STAGE2
 security-level 90
 zone-member STAGE
 ip address 10.5.254.5 255.255.255.252 
!
interface GigabitEthernet0/1.4002
 vlan 4002
 nameif PROD2
 security-level 90
 zone-member PROD
 ip address 10.6.254.5 255.255.255.252 
!             
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface
mtu DEV1 1500
mtu STAGE1 1500
mtu PROD1 1500
mtu DEV2 1500 
mtu STAGE2 1500
mtu PROD2 1500
no failover
no failover wait-disable
no monitor-interface DEV1
no monitor-interface STAGE1
no monitor-interface PROD1
no monitor-interface DEV2
no monitor-interface STAGE2
no monitor-interface PROD2
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
class-map inspection_default
 match default-inspection-traffic
!
policy-map type inspect dns migrated_dns_map_1
 parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
policy-map global_policy
 class inspection_default
  inspect dns migrated_dns_map_1 
  inspect ftp 
  inspect h323 h225 
  inspect h323 ras 
  inspect ip-options 
  inspect netbios 
  inspect rsh 
  inspect rtsp 
  inspect skinny  
  inspect esmtp 
  inspect sqlnet 
  inspect sunrpc 
  inspect tftp 
  inspect sip  
  inspect snmp 
policy-map type inspect dns migrated_dns_map_2
 parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
!
service-policy global_policy global
!
```
**FW2**
```
!
hostname FW2
enable password ***** pbkdf2
service-module 0 keepalive-timeout 4
service-module 0 keepalive-counter 6
names
no mac-address auto
zone DEV
zone STAGE
zone PROD

!
interface GigabitEthernet0/0
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/0.3000
 vlan 3000
 nameif DEV1
 security-level 90
 zone-member DEV
 ip address 10.4.255.1 255.255.255.252 
!
interface GigabitEthernet0/0.3001
 vlan 3001
 nameif STAGE1
 security-level 90
 zone-member STAGE
 ip address 10.5.255.1 255.255.255.252 
!
interface GigabitEthernet0/0.3002
 vlan 3002
 nameif PROD1
 security-level 90
 zone-member PROD
 ip address 10.6.255.1 255.255.255.252 
!
interface GigabitEthernet0/1
 no nameif
 no security-level
 no ip address
!
interface GigabitEthernet0/1.4000
 vlan 4000
 nameif DEV2
 security-level 90
 zone-member DEV
 ip address 10.4.255.5 255.255.255.252 
!
interface GigabitEthernet0/1.4001
 vlan 4001
 nameif STAGE2
 security-level 90
 zone-member STAGE
 ip address 10.5.255.5 255.255.255.252 
!
interface GigabitEthernet0/1.4002
 vlan 4002
 nameif PROD2
 security-level 90
 zone-member PROD
 ip address 10.6.255.5 255.255.255.252 
!             
ftp mode passive
same-security-traffic permit inter-interface
same-security-traffic permit intra-interface
pager lines 23
mtu DEV1 1500
mtu STAGE1 1500
mtu PROD1 1500
mtu DEV2 1500
mtu STAGE2 1500
mtu PROD2 1500
failover
no failover wait-disable
no monitor-interface DEV1
no monitor-interface STAGE1
no monitor-interface PROD1
no monitor-interface DEV2
no monitor-interface STAGE2
no monitor-interface PROD2
icmp unreachable rate-limit 1 burst-size 1
no asdm history enable
arp timeout 14400
no arp permit-nonconnected
arp rate-limit 8192
router bgp 64086.59997
 bgp log-neighbor-changes
 bgp asnotation dot
 address-family ipv4 unicast
  neighbor 10.4.255.2 remote-as 64086.60002
  neighbor 10.4.255.2 activate
  neighbor 10.4.255.2 default-originate
  neighbor 10.5.255.2 remote-as 64086.60002
  neighbor 10.5.255.2 activate
  neighbor 10.5.255.2 default-originate
  neighbor 10.6.255.2 remote-as 64086.60002
  neighbor 10.6.255.2 activate
  neighbor 10.6.255.2 default-originate
  neighbor 10.4.255.6 remote-as 64086.60002
  neighbor 10.4.255.6 activate
  neighbor 10.4.255.6 default-originate
  neighbor 10.5.255.6 remote-as 64086.60002
  neighbor 10.5.255.6 activate
  neighbor 10.5.255.6 default-originate
  neighbor 10.6.255.6 remote-as 64086.60002
  neighbor 10.6.255.6 activate
  neighbor 10.6.255.6 default-originate
  maximum-paths 2
  no auto-summary
  no synchronization
 exit-address-family
!
class-map inspection_default
 match default-inspection-traffic
!
policy-map type inspect dns preset_dns_map
 parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
policy-map global_policy
 class inspection_default
  inspect ip-options 
  inspect netbios 
  inspect rtsp 
  inspect sunrpc 
  inspect tftp 
  inspect dns preset_dns_map 
  inspect ftp 
  inspect h323 h225 
  inspect h323 ras 
  inspect rsh 
  inspect esmtp 
  inspect sqlnet 
  inspect sip  
  inspect skinny  
  inspect snmp 
policy-map type inspect dns migrated_dns_map_2
 parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
policy-map type inspect dns migrated_dns_map_1
 parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
!
service-policy global_policy global
!
```

**BLeaf13**
```
BLeaf13#show run
! Command: show running-config
! device: BLeaf13 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
link tracking group MLAG_UPLINK_TRACK
   recovery delay 60
!
hostname BLeaf13
!
spanning-tree mode mstp
no spanning-tree vlan-id 4094
!
vlan 10-11,20
!
vlan 3000
   name DEV
!
vlan 3001
   name STAGE
!
vlan 3002
   name PROD
!
vrf instance DEV
!
vrf instance PROD
!
vrf instance STAGE
!
interface Ethernet1
   no switchport
   ip address 10.2.1.4/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.4/31
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   switchport trunk allowed vlan 3000-3002
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Management1
!
interface Vlan1
!
interface Vlan10
   vrf DEV
   ip address 10.4.0.253/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf STAGE
   ip address 10.5.0.253/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf PROD
   ip address 10.6.0.253/24
   ip virtual-router address 10.6.0.254
!
interface Vlan3000
   vrf DEV
   ip address 10.4.254.2/30
!
interface Vlan3001
   vrf STAGE
   ip address 10.5.254.2/30
!
interface Vlan3002
   vrf PROD
   ip address 10.6.254.2/30
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf DEV vni 1000
   vxlan vrf PROD vni 3000
   vxlan vrf STAGE vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf DEV
ip routing vrf PROD
ip routing vrf STAGE
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.0.3
   timers bgp 3 9
   maximum-paths 2
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60001
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
   !
   vrf DEV
      rd 10.0.0.3:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.254.1 remote-as 64086.59998
      neighbor 10.4.254.1 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf PROD
      rd 10.0.0.3:3000
      route-target import evpn 3:3000
      route-target export evpn 3:3000
      neighbor 10.6.254.1 remote-as 64086.59998
      neighbor 10.6.254.1 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf STAGE
      rd 10.0.0.3:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.254.1 remote-as 64086.59998
      neighbor 10.5.254.1 route-map EX_MACIP_RM out
      redistribute connected
!
end
```
**BLeaf14**
```

Leaf14#show run
! Command: show running-config
! device: Leaf14 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
link tracking group MLAG_UPLINK_TRACK
   recovery delay 60
!
hostname Leaf14
!
spanning-tree mode mstp
no spanning-tree vlan-id 4094
!
vlan 10-11,20
!
vlan 4000
   name DEV
!
vlan 4001
   name STAGE
!
vlan 4002
   name PROD
!
vrf instance DEV
!
vrf instance PROD
!
vrf instance STAGE
!
interface Ethernet1
   no switchport
   ip address 10.2.1.6/31
!
interface Ethernet2
   no switchport
   ip address 10.2.2.6/31
!
interface Ethernet3
   no switchport
   ip address 10.1.1.1/30
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   switchport trunk allowed vlan 4000-4002
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.4/32
!
interface Management1
!
interface Vlan10
   vrf DEV
   ip address 10.4.0.252/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf STAGE
   ip address 10.5.0.252/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf PROD
   ip address 10.6.0.252/24
   ip virtual-router address 10.5.1.254
   ip virtual-router address 10.6.0.254
!
interface Vlan4000
   vrf DEV
   ip address 10.4.254.6/30
!
interface Vlan4001
   vrf STAGE
   ip address 10.5.254.6/30
!
interface Vlan4002
   vrf PROD
   ip address 10.6.254.6/30
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf DEV vni 1000
   vxlan vrf PROD vni 3000
   vxlan vrf STAGE vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf DEV
ip routing vrf PROD
ip routing vrf STAGE
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60001
   bgp asn notation asdot
   router-id 10.0.0.4
   timers bgp 3 9
   maximum-paths 2
   neighbor DCI peer group
   neighbor DCI remote-as 64086.59999
   neighbor DCI bfd
   neighbor DCI password 7 4FkDjOFG29mSM/5rvAI1UQ==
   neighbor DCI send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60001
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor 10.1.1.2 peer group DCI
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
      neighbor DCI activate
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor SPINE activate
      network 10.0.0.4/32
      network 10.1.1.0/30
   !
   vrf DEV
      rd 10.0.0.4:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.254.5 remote-as 64086.59998
      neighbor 10.4.254.5 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf PROD
      rd 10.0.0.4:3000
      route-target import evpn 3:3000
      route-target export evpn 3:3000
      neighbor 10.6.254.5 remote-as 64086.59998
      neighbor 10.6.254.5 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf STAGE
      rd 10.0.0.4:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.254.5 remote-as 64086.59998
      neighbor 10.5.254.5 route-map EX_MACIP_RM out
      redistribute connected
!
end
```
**BLeaf23**
```
Leaf23#show run
! Command: show running-config
! device: Leaf23 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf23
!
spanning-tree mode mstp
!
vlan 10-11,20
!
vlan 3000
   name DEV
!
vlan 3001
   name STAGE
!
vlan 3002
   name PROD
!
vrf instance DEV
!
vrf instance PROD
!
vrf instance STAGE
!
interface Ethernet1
   no switchport
   ip address 10.2.101.204/31
!
interface Ethernet2
   no switchport
   ip address 10.2.102.204/31
!
interface Ethernet3
   no switchport
   ip address 10.1.2.1/30
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   switchport trunk allowed vlan 3000-3002
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.103/32
!
interface Management1
!
interface Vlan1
!
interface Vlan10
   vrf DEV
   ip address 10.4.0.251/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf STAGE
   ip address 10.5.0.251/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf PROD
   ip address 10.6.0.251/24
   ip virtual-router address 10.6.0.254
!
interface Vlan3000
   vrf DEV
   ip address 10.4.255.2/30
!
interface Vlan3001
   vrf STAGE
   ip address 10.5.255.2/30
!
interface Vlan3002
   vrf PROD
   ip address 10.6.255.2/30
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf DEV vni 1000
   vxlan vrf PROD vni 3000
   vxlan vrf STAGE vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf DEV
ip routing vrf PROD
ip routing vrf STAGE
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60002
   bgp asn notation asdot
   router-id 10.0.0.103
   timers bgp 3 9
   maximum-paths 2
   neighbor DCI peer group
   neighbor DCI remote-as 64086.59999
   neighbor DCI bfd
   neighbor DCI password 7 4FkDjOFG29mSM/5rvAI1UQ==
   neighbor DCI send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60002
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor 10.1.2.2 peer group DCI
   neighbor 10.2.101.205 peer group SPINE
   neighbor 10.2.102.205 peer group SPINE
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
      neighbor DCI activate
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor SPINE activate
      network 10.0.0.103/32
      network 10.1.2.0/30
   !
   vrf DEV
      rd 10.0.0.103:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.255.1 remote-as 64086.59997
      neighbor 10.4.255.1 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf PROD
      rd 10.0.0.103:3000
      route-target import evpn 3:3000
      route-target export evpn 3:3000
      neighbor 10.6.255.1 remote-as 64086.59997
      neighbor 10.6.255.1 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf STAGE
      rd 10.0.0.103:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.255.1 remote-as 64086.59997
      neighbor 10.5.255.1 route-map EX_MACIP_RM out
      redistribute connected
!
end
```
**BLeaf24**
```
Leaf24#show run
! Command: show running-config
! device: Leaf24 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname Leaf24
!
spanning-tree mode mstp
!
vlan 10-11,20
!
vlan 4000
   name DEV
!
vlan 4001
   name STAGE
!
vlan 4002
   name PROD
!
vrf instance DEV
!
vrf instance PROD
!
vrf instance STAGE
!
interface Ethernet1
   no switchport
   ip address 10.2.101.206/31
!
interface Ethernet2
   no switchport
   ip address 10.2.102.206/31
!
interface Ethernet3
   no switchport
   ip address 10.1.3.1/30
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
   switchport trunk allowed vlan 4000-4002
   switchport mode trunk
!
interface Loopback0
   ip address 10.0.0.104/32
!
interface Management1
!
interface Vlan10
   vrf DEV
   ip address 10.4.0.250/24
   ip virtual-router address 10.4.0.254
!
interface Vlan11
   vrf STAGE
   ip address 10.5.0.250/24
   ip virtual-router address 10.5.0.254
!
interface Vlan20
   vrf PROD
   ip address 10.6.0.250/24
   ip virtual-router address 10.5.1.254
!
interface Vlan4000
   vrf DEV
   ip address 10.4.255.6/30
!
interface Vlan4001
   vrf STAGE
   ip address 10.5.255.6/30
!
interface Vlan4002
   vrf PROD
   ip address 10.6.255.6/30
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 11 vni 10011
   vxlan vlan 20 vni 10020
   vxlan vrf DEV vni 1000
   vxlan vrf PROD vni 3000
   vxlan vrf STAGE vni 2000
   vxlan learn-restrict any
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf DEV
ip routing vrf PROD
ip routing vrf STAGE
!
ip prefix-list EX_MACIP_PL
   seq 10 permit 0.0.0.0/0 le 31
!
route-map EX_MACIP_RM permit 10
   match ip address prefix-list EX_MACIP_PL
!
router bgp 64086.60002
   bgp asn notation asdot
   router-id 10.0.0.104
   timers bgp 3 9
   maximum-paths 2
   neighbor DCI peer group
   neighbor DCI remote-as 64086.59999
   neighbor DCI bfd
   neighbor DCI password 7 4FkDjOFG29mSM/5rvAI1UQ==
   neighbor DCI send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 64086.60002
   neighbor SPINE bfd
   neighbor SPINE password 7 EH+yVyyau5QNVADGud/EtQ==
   neighbor SPINE send-community extended
   neighbor 10.1.3.2 peer group DCI
   neighbor 10.2.101.207 peer group SPINE
   neighbor 10.2.102.207 peer group SPINE
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
      neighbor DCI activate
      neighbor SPINE activate
   !
   address-family ipv4
      neighbor DCI activate
      neighbor SPINE activate
      network 10.0.0.104/32
      network 10.1.3.0/30
   !
   vrf DEV
      rd 10.0.0.104:1000
      route-target import evpn 1:1000
      route-target export evpn 1:1000
      neighbor 10.4.255.5 remote-as 64086.59997
      neighbor 10.4.255.5 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf PROD
      rd 10.0.0.104:3000
      route-target import evpn 3:3000
      route-target export evpn 3:3000
      neighbor 10.6.255.5 remote-as 64086.59997
      neighbor 10.6.255.5 route-map EX_MACIP_RM out
      redistribute connected
   !
   vrf STAGE
      rd 10.0.0.104:2000
      route-target import evpn 2:2000
      route-target export evpn 2:2000
      neighbor 10.5.255.5 remote-as 64086.59997
      neighbor 10.5.255.5 route-map EX_MACIP_RM out
      redistribute connected
!
end
```
