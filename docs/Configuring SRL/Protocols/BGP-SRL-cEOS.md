# Configuring eBGP between Nokia SRL and Arista cEOS

This note describes how to setup isis between Nokia SRLinux and Arista cEOS. We use a triangle topology with 2 SRL and 1 cEOS device as an example

```
         WAN1 (SRL) (e1-1) -------(e1-1) WAN2 (SRL)
            (e1-2)                        (e1-2)
               |                            |
               +--(Et2) WAN3 (cEOS) (Et1)---+
```

We configure both IPv4 and IPv6 to show dual stack BGP environment


## Configuring interfaces

First we have to configure the interfaces on the various devices

### WAN1 - SRL router

Configure the interface on WAN1 to WAN2 SRL

```
--{ running }--[ interface ethernet-1/1 ]--
A:wan1# info
    admin-state enable
    subinterface 1 {
        admin-state enable
        ipv4 {
            address 192.1.1.1/24 {
            }
        }
        ipv6 {
            address 2000:192:1::1/64 {
            }
        }
    }
```
Configure the interface on WAN1 to WAN3 cEOS

```
--{ running }--[ interface ethernet-1/2 ]--
A:wan1# info
    admin-state enable
    subinterface 1 {
        admin-state enable
        ipv4 {
            address 192.1.3.1/24 {
            }
        }
        ipv6 {
            address 2000:192:3::1/64 {
            }
        }
    }
``` 
### WAN2- SRL router

Configure the interface on WAN2 to WAN1 SRL

```
--{ running }--[ interface ethernet-1/1 ]--
A:wan2# info
    subinterface 1 {
        admin-state enable
        ipv4 {
            address 192.1.1.2/24 {
            }
        }
        ipv6 {
            address 2000:192:1::2/64 {
            }
        }
    }
```

Configure the interface on WAN2 to WAN3 cEOS

```
--{ running }--[ interface ethernet-1/2 ]--
A:wan2# info
    admin-state enable
    subinterface 1 {
        admin-state enable
        ipv4 {
            address 192.1.2.1/24 {
            }
        }
        ipv6 {
            address 2000:192:2::1/64 {
            }
        }
    }
```

### WAN3 - cEOS router

Configure the interface on WAN3 to WAN2 SRL

```
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
   ipv6 address 2000:192:2::2/64
```

Configure the interface on WAN3 to WAN2 SRL

```
interface Ethernet2
   no switchport
   ip address 192.1.3.2/24
   ipv6 address 2000:192:3::2/64
```

## Configuring the router instance

Before we can configure BGP we should enable the router instance on the various devices. On SRL we create a network-instance, which is like a VRF-Lite and can be used for the global routing table. In cEOS we enable ip routing and ipv6 routing.

### WAN1 - SRL router

```
--{ running }--[ network-instance default ]--
A:wan1# info
    type default
    admin-state enable
    interface ethernet-1/1.1 {
    }
    interface ethernet-1/2.1 {
    }
    interface lo1.1 {
    }
```

### WAN2- SRL router

```
--{ running }--[ network-instance default ]--
A:wan2# info
    type default
    admin-state enable
    interface ethernet-1/1.1 {
    }
    interface ethernet-1/2.1 {
    }
    interface lo1.1 {
    }
```

### WAN3 - cEOS router

```
ip routing
!
ipv6 unicast-routing
!
```

## Configuring BGP

After the interfaces are configured we will configure dual stack eBGP between the various routers.

### WAN1 - SRL router
On SRL we create a BGP protocol instance lab

```
--{ candidate private private-root }--[ network-instance default protocols ]--
A:wan1# isis instance lab
```
After we configure the isis parameters like:

* admin-state: enables the isis instance
* interface configuration for ethernet1-1.1, ethernet1-2/1 and lo1.1
* IPv4-unicast
* IPv6-unicast
* net: The NET is 8-20 octets long and consists of 3 parts: 

	1. Area ID — A variable length field between 1 and 13 bytes. This includes the Authority and Format Identifier (AFI) as the most significant byte and the area ID.
	2. System ID — A 6-byte system identification. This value is not configurable. The system ID is derived from the system or router ID.
	3. Selector ID — A 1-byte selector identification that must contain zeros when configuring a NET. This value is not configurable. The selector ID is always 00.


There are many more features supported by SRL which can be read in the configuration guides

```
A:wan1# instance lab
--{ running }--[ network-instance default protocols isis instance lab ]--
A:wan1# info
    admin-state enable
    net [
        49.0001.0001.0000.0000.00
    ]
    ipv4-unicast {
        admin-state enable
    }
    ipv6-unicast {
        admin-state enable
    }
    interface ethernet-1/1.1 {
        admin-state enable
        circuit-type point-to-point
    }
    interface ethernet-1/2.1 {
        admin-state enable
        circuit-type point-to-point
    }
    interface lo1.1 {
        passive true
    }
```
### WAN2 - SRL router
A similar configuration is setup on WAN2 router

```
--{ running }--[ network-instance default protocols isis instance lab ]--
A:wan2# info
    admin-state enable
    net [
        49.0001.0002.0000.0000.00
    ]
    ipv4-unicast {
        admin-state enable
    }
    ipv6-unicast {
        admin-state enable
    }
    interface ethernet-1/1.1 {
        admin-state enable
        circuit-type point-to-point
    }
    interface ethernet-1/2.1 {
        admin-state enable
        circuit-type point-to-point
    }
    interface lo1.1 {
        passive true
    }
```

### WAN3 - cEOS router
On cEOS the configuration is slightly different,

First we enable the isis instance with the specific net/system id and disable hello-padding

```
router isis lab
   no hello padding
   net 49.0001.0003.0000.0000.00
   log-adjacency-changes
   !
   address-family ipv4 unicast
   !
   address-family ipv6 unicast
!
```

After we map the interfaces to the isis instance

```
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
   ipv6 address 2000:192:2::2/64
   isis enable lab
   isis network point-to-point
!
interface Ethernet2
   no switchport
   ip address 192.1.3.2/24
   ipv6 address 2000:192:3::2/64
   isis enable lab
   isis network point-to-point
!
```

## Viewing ISIS state

Now that we configured OSPF on the various devices we can look at the operational state of the various devices

### ISIS adjacency

On SRL devices

```
A:wan1# show / network-instance default protocols isis adjacency
Network Instance: default
Instance        : lab
+------------------+-----------------+----------------+------------+-----------------+-------+-----------------+-----------------+
| Neighbor System  | Adjacency Level | Interface Name | Ip Address |  Ipv6 Address   | State | Last transition |    Remaining    |
|        Id        |                 |                |            |                 |       |                 |    holdtime     |
+==================+=================+================+============+=================+=======+=================+=================+
| 0002.0000.0000   | L2              | ethernet-1/1.1 | 192.1.1.2  | fe80::201:1ff:f | up    | 2020-08-30T16:3 | 22              |
|                  |                 |                |            | eff:0           |       | 4:10.100Z       |                 |
+------------------+-----------------+----------------+------------+-----------------+-------+-----------------+-----------------+
+------------------+-----------------+----------------+------------+-----------------+-------+-----------------+-----------------+
| Neighbor System  | Adjacency Level | Interface Name | Ip Address |  Ipv6 Address   | State | Last transition |    Remaining    |
|        Id        |                 |                |            |                 |       |                 |    holdtime     |
+==================+=================+================+============+=================+=======+=================+=================+
| 0003.0000.0000   | L1L2            | ethernet-1/2.1 | 192.1.3.2  | fe80::44f4:d4ff | up    | 2020-08-30T16:3 | 22              |
|                  |                 |                |            | :fec9:1af1      |       | 4:20.500Z       |                 |
+------------------+-----------------+----------------+------------+-----------------+-------+-----------------+-----------------+
+-------------------+-----------------+----------------+------------+--------------+-------+-----------------+-------------------+
|  Neighbor System  | Adjacency Level | Interface Name | Ip Address | Ipv6 Address | State | Last transition |     Remaining     |
|        Id         |                 |                |            |              |       |                 |     holdtime      |
+===================+=================+================+============+==============+=======+=================+===================+
+-------------------+-----------------+----------------+------------+--------------+-------+-----------------+-------------------+
Adjacency Count: 2
==================================================================================================================================
```

On cEOS device

```

```

### ISIS database

On SRL devices

```
A:wan1# show / network-instance default protocols isis database
----------------------------------------------------------------------------------------------------------------------------------
Network Instance: default
Instance        : lab
+--------------+----------------------+----------+----------+----------+------------+
| Level Number |        Lsp Id        | Sequence | Checksum | Lifetime | Attributes |
+==============+======================+==========+==========+==========+============+
| 2            | 0001.0000.0000.00-00 | 0x1d     | 0x6dab   | 834      | L1 L2      |
| 2            | 0002.0000.0000.00-00 | 0x4b     | 0xd217   | 809      | L1 L2      |
| 2            | 0003.0000.0000.00-00 | 0x13     | 0xa8e5   | 937      | L1 L2      |
+--------------+----------------------+----------+----------+----------+------------+
LSP Count: 3
----------------------------------------------------------------------------------------------------------------------------------
```

On cEOS device

```
wan3(config-router-isis)#sh isis database

IS-IS Instance: lab VRF: default
  IS-IS Level 1 Link State Database
    LSPID                 Seq Num   Cksum  Life  IS Flags
    wan3.00-00            14        20104  466   L2 <>
  IS-IS Level 2 Link State Database
    LSPID                 Seq Num   Cksum  Life  IS Flags
    wan1.00-00            30        27564  985   L2 <>
    wan2.00-00            76        53272  907   L2 <>
    wan3.00-00            19        43237  444   L2 <>
```

### Routing table

on SRL devices

```
A:wan1# show / network-instance default route-table all
----------------------------------------------------------------------------------------------------------------------------------
IPv4 Unicast route table of network instance default
----------------------------------------------------------------------------------------------------------------------------------
+-----------------------+------+-----------+----------------+--------+------+-------------------------------+--------------+
|        Prefix         |  ID  |  Active   |     Owner      | Metric | Pref |        Next-hop (Type)        |   Next-hop   |
|                       |      |           |                |        |      |                               |  Interface   |
+=======================+======+===========+================+========+======+===============================+==============+
| 1.1.1.1/32            | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 1.1.1.2/32            | 0    | true      | isis           | 10     | 18   | 192.1.1.2 (direct)            | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
| 192.1.1.0/24          | 0    | true      | local          | 0      | 0    | 192.1.1.1 (direct)            | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
| 192.1.1.1/32          | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 192.1.1.255/32        | 0    | true      | host           | 0      | 0    | None (broadcast)              | None         |
| 192.1.2.0/24          | 0    | true      | isis           | 20     | 18   | 192.1.1.2 (direct)            | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
| 192.1.3.0/24          | 0    | true      | local          | 0      | 0    | 192.1.3.1 (direct)            | ethernet-1/2 |
|                       |      |           |                |        |      |                               | .1           |
| 192.1.3.1/32          | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 192.1.3.255/32        | 0    | true      | host           | 0      | 0    | None (broadcast)              | None         |
+-----------------------+------+-----------+----------------+--------+------+-------------------------------+--------------+
----------------------------------------------------------------------------------------------------------------------------------
9 IPv4 routes total
9 IPv4 prefixes with active routes
0 IPv4 prefixes with active ECMP routes
----------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------
IPv6 Unicast route table of network instance default
----------------------------------------------------------------------------------------------------------------------------------
+-----------------------+------+-----------+----------------+--------+------+-------------------------------+--------------+
|        Prefix         |  ID  |  Active   |     Owner      | Metric | Pref |        Next-hop (Type)        |   Next-hop   |
|                       |      |           |                |        |      |                               |  Interface   |
+=======================+======+===========+================+========+======+===============================+==============+
| 2000:192:1::/64       | 0    | true      | local          | 0      | 0    | 2000:192:1::1 (direct)        | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
| 2000:192:1::1/128     | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 2000:192:2::/64       | 0    | true      | isis           | 20     | 18   | fe80::201:1ff:feff:0 (direct) | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
| 2000:192:3::/64       | 0    | true      | local          | 0      | 0    | 2000:192:3::1 (direct)        | ethernet-1/2 |
|                       |      |           |                |        |      |                               | .1           |
| 2000:192:3::1/128     | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 3000::1/128           | 0    | true      | host           | 0      | 0    | None (extract)                | None         |
| 3000::2/128           | 0    | true      | isis           | 10     | 18   | fe80::201:1ff:feff:0 (direct) | ethernet-1/1 |
|                       |      |           |                |        |      |                               | .1           |
+-----------------------+------+-----------+----------------+--------+------+-------------------------------+--------------+
----------------------------------------------------------------------------------------------------------------------------------
7 IPv6 routes total
7 IPv6 prefixes with active routes
0 IPv6 prefixes with active ECMP routes
----------------------------------------------------------------------------------------------------------------------------------
```

on cEOS device

```
wan3(config)#sh ip route

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

Gateway of last resort is not set

 I L2     1.1.1.1/32 [115/10] via 192.1.3.1, Ethernet2
 I L2     1.1.1.2/32 [115/10] via 192.1.2.1, Ethernet1
 I L2     192.1.1.0/24 [115/20] via 192.1.2.1, Ethernet1
                                via 192.1.3.1, Ethernet2
 C        192.1.2.0/24 is directly connected, Ethernet1
 C        192.1.3.0/24 is directly connected, Ethernet2

wan3(config)#sh ipv6 route

VRF: default
Displaying 5 of 11 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3, B - BGP, R - RIP, A B - BGP Aggregate, I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP, NG - Nexthop Group Static Route, M - Martian, DP - Dynamic Policy Route, L - VRF Leaked, RC - Route Cache Route

 I L2     2000:192:1::/64 [115/20]
           via fe80::201:1ff:feff:1, Ethernet1
           via fe80::201:ff:feff:1, Ethernet2
 C        2000:192:2::/64 [0/1]
           via Ethernet1, directly connected
 C        2000:192:3::/64 [0/1]
           via Ethernet2, directly connected
 I L2     3000::1/128 [115/10]
           via fe80::201:ff:feff:1, Ethernet2
 I L2     3000::2/128 [115/10]
           via fe80::201:1ff:feff:1, Ethernet1
```