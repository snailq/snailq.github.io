---
bg: "sea2.jpg"
layout: post
title:  "ipv6 ospf quagga"
crawlertitle: "ipv6 ospf quagga"
summary: "ipv6 ospf quagga"
date:   2019-03-04
categories: posts
author: snailqh
---
fe80:: link-local 链路本地地址，根据MAC地址自动生成

#### /etc/sysconfig/network-scripts/ifcfg-dpdk1.kni
```shell
DEVICE=dpdk1.kni
NAME=dpdk1.kni
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
HWADDR=48:df:37:20:38:dd

IPV6INIT=yes
IPV6_AUTOCONF=no
IPV6_FAILURE_FATAL=no
IPV6_DEFROUTE=yes
IPV6ADDR=fd00::4/64
IPV6_DEFAULTGW=fd00::1

# ip ad
lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 fd01::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever

dpdk1.kni: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 48:df:37:20:38:dd brd ff:ff:ff:ff:ff:ff
    inet6 fd00::4/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::4adf:37ff:fe20:38dd/64 scope link
       valid_lft forever preferred_lft forever
```

##### /etc/quagga/ospf6d.conf
```shell
!
password 8 yyyy.com
enable password 8 yyyy.com
log file /var/log/quagga/ospfd.log
service password-encryption
!
!
!
!
interface dpdk0.kni
!
interface dpdk1.kni
ipv6 ospf6 hello-interval 2
ipv6 ospf6 dead-interval 10
!ipv6 ospf6 retransmit-interval 5
!ipv6 ospf6 priority 1
!ipv6 ospf6 transmit-delay 1
 ipv6 ospf6 instance-id 0
interface lo
!
router ospf6
 router-id 172.17.1.1
 area 0.0.0.10 range fd01::/64
 interface dpdk1.kni area 0.0.0.10
 interface lo area 0.0.0.10
!
line vty
!
# show ipv6  ospf6  neighbor
Neighbor ID     Pri    DeadTime  State/IfState         Duration I/F[State]
172.16.1.1      100    00:00:37   Full/DR              00:14:12 dpdk1.kni[DROther]
172.16.1.2      120    00:00:35   Full/BDR             00:14:14 dpdk1.kni[DROther]

```
### h3c 6800
```shell
interface LoopBack0
 ip address 172.16.1.1 255.255.255.255
#
interface Vlan-interface10
 ospfv3 10 area 0.0.0.10
 ospfv3 dr-priority 100
 ospfv3 timer hello 2
 ospfv3 timer dead 10
 ipv6 address FD00::1/64

 # dis ipv6 neighbors all
Type: S-Static    D-Dynamic    O-Openflow     R-Rule    I-Invalid
IPv6 address              Link layer     VID  Interface/Link ID   State T  Age
FD00::2                   88df-9e61-ac7f 10   FGE1/0/49           STALE D  1826
FD00::3                   48df-3720-b3d5 10   FGE1/0/49           STALE D  1824
FE80::4ADF:37FF:FE20:B3D5 48df-3720-b3d5 10   FGE1/0/49           STALE D  1819
FE80::8ADF:9EFF:FE61:AC7F 88df-9e61-ac7f 10   FGE1/0/49           STALE D  863
FD00::4                   48df-3720-38dd 10   XGE1/0/1            STALE D  1834
FE80::4ADF:37FF:FE20:38DD 48df-3720-38dd 10   XGE1/0/1            STALE D  865

# dis ospfv3 peer

               OSPFv3 Process 1 with Router ID 0.0.0.0

               OSPFv3 Process 10 with Router ID 172.16.1.1

 Area: 0.0.0.10
-------------------------------------------------------------------------
 Router ID       Pri State             Dead-Time InstID Interface
 172.16.1.2      120 2-Way/ -          00:00:10  0      Vlan10
 172.17.1.1      1   Full/DR           00:00:10  0      Vlan10
 172.17.1.2      1   Full/BDR          00:00:10  0      Vlan10
```
