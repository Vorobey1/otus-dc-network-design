**FW1**

'''
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
'''
**FW2**

'''
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
'''

