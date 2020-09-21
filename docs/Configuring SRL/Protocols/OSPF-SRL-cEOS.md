# Configuring OSPF between Nokia SRL and Arista cEOS

This note describes how to setup ospf between Nokia SRLinux and Arista cEOS. We use a triangle topology with 2 SRL and 1 cEOS device as an example

```
         WAN1 (SRL) (e1-1) -------(e1-1) WAN2 (SRL)
            (e1-2)                        (e1-2)
               |                            |
               +--(Et2) WAN3 (cEOS) (Et1)---+
```

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
    }
--{ running }--[ interface ethernet-1/1 ]--
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
    }
```

### WAN3 - cEOS router

Configure the interface on WAN3 to WAN2 SRL

```
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
!
```

Configure the interface on WAN3 to WAN2 SRL

```
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
!
```

## Configuring the router instance

Before we can configure OSPF we should enable the router instance on the various devices. On SRL we create a network-instance, which is like a VRF-Lite and can be used for the global routing table. In cEOS we enable ip routing.

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
```

## Configuring OSPF

After the interfaces are configured we will configure an OSPF Backbone area 0.0.0.0 between the various routers.

### WAN1 - SRL router
On SRL we create a OSPF protocol instance

```
--{ running }--[ network-instance default ]--
A:wan1# protocols ospf instance lab
```

After we configure the ospf parameters like:

* admin-state: enables the ospf instance
* version: allows to select the OSPF version OSPFv2 or OSPFv3
* router-id: typically setup using the system loopback of the router, which is the same ip address as configured on lo0.1
* area: here we use area 0.0.0.0 with its interfaces

There are many more features supported by SRL which can be read in the configuration guides

```
--{ running }--[ network-instance default protocols ospf instance lab ]--
A:wan1# info
    admin-state enable
    version OSPF_V2
    router-id 10.1.1.1
    area 0.0.0.0 {
        interface ethernet-1/1.1 {
            admin-state enable
            interface-type point-to-point
        }
        interface ethernet-1/2.1 {
            admin-state enable
            interface-type point-to-point
        }
        interface lo1.1 {
            admin-state enable
            passive true
        }
    }

```
### WAN2 - SRL router
A similar configuration is setup on WAN2 router

```
--{ running }--[ network-instance default protocols ospf instance lab ]--
A:wan2# info
    admin-state enable
    version OSPF_V2
    router-id 10.1.1.2
    area 0.0.0.0 {
        interface ethernet-1/1.1 {
            admin-state enable
            interface-type point-to-point
        }
        interface ethernet-1/2.1 {
            admin-state enable
            interface-type point-to-point
        }
        interface lo1.1 {
            admin-state enable
            passive true
        }
    }
```

### WAN3 - cEOS router
On cEOS the configuration is slightly different,

First we enable the ospf instance
```
router ospf 1
   router-id 10.1.1.3
   max-lsa 12000
!
```

After we map the interfaces to the backbone area
```
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
```
```
interface Ethernet2
   no switchport
   ip address 192.1.3.2/24
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
```

## Viewing OSPF state

Now that we configured OSPF on the various devices we can look at the operational state of the various devices

### OSPF neighbors

On SRL devices

```
--{ running }--[ network-instance default protocols ospf instance lab ]--
A:wan1# show ospf lab neighbor
===================================================================================================================================
Net-Inst default OSPFv2 Instance lab Neighbors
===================================================================================================================================
+--------------------------------------------------------------------+
| Interface-Name    Rtr Id    State   Pri   RetxQ   Time Before Dead |
+====================================================================+
| ethernet-1/1.1   10.1.1.2   FULL    1     0       39               |
| ethernet-1/2.1   10.1.1.3   FULL    0     0       36               |
+--------------------------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------
No. of Neighbors: 2
===================================================================================================================================
```

On cEOS device

```
wan3(config-if-Et1)#sh ip ospf 1 neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface

% Internal error
% To see the details of this error, run the command 'show error 6'
```

### OSPF database
On SRL
```
command not yet available
```
On cEOS
```
wan3(config-if-Et1)#sh ip ospf 1 database

            OSPF Router with ID(10.1.1.3) (Instance ID 1) (VRF default)


                 Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age         Seq#         Checksum Link count
255.255.255.255 255.255.255.255 3107        0x8000000d   0x6e67   5
10.1.1.2        10.1.1.2        805         0x80000014   0x3d75   5
10.1.1.1        10.1.1.1        1087        0x80000014   0xf1c1   5
```

### Routing table
on SRL

```
A:wan1# show / network-instance default route-table all
-----------------------------------------------------------------------------------------------------------------------------------
IPv4 Unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------
+-----------------------+------+-----------+----------------+--------+------+--------------------------------+--------------+
|        Prefix         |  ID  |  Active   |     Owner      | Metric | Pref |        Next-hop (Type)         |   Next-hop   |
|                       |      |           |                |        |      |                                |  Interface   |
+=======================+======+===========+================+========+======+================================+==============+
| 1.1.1.1/32            | 0    | true      | host           | 0      | 0    | None (extract)                 | None         |
| 1.1.1.2/32            | 0    | true      | ospfv2         | 1      | 10   | 192.1.1.2 (direct)             | ethernet-1/1 |
|                       |      |           |                |        |      |                                | .1           |
| 192.1.1.0/24          | 0    | true      | local          | 0      | 0    | 192.1.1.1 (direct)             | ethernet-1/1 |
|                       |      |           |                |        |      |                                | .1           |
| 192.1.1.1/32          | 0    | true      | host           | 0      | 0    | None (extract)                 | None         |
| 192.1.1.255/32        | 0    | true      | host           | 0      | 0    | None (broadcast)               | None         |
| 192.1.2.0/24          | 0    | true      | ospfv2         | 2      | 10   | 192.1.1.2 (direct)             | ethernet-1/1 |
|                       |      |           |                |        |      |                                | .1           |
| 192.1.3.0/24          | 0    | true      | local          | 0      | 0    | 192.1.3.1 (direct)             | ethernet-1/2 |
|                       |      |           |                |        |      |                                | .1           |
| 192.1.3.1/32          | 0    | true      | host           | 0      | 0    | None (extract)                 | None         |
| 192.1.3.255/32        | 0    | true      | host           | 0      | 0    | None (broadcast)               | None         |
+-----------------------+------+-----------+----------------+--------+------+--------------------------------+--------------+
-----------------------------------------------------------------------------------------------------------------------------------
9 IPv4 routes total
9 IPv4 prefixes with active routes
0 IPv4 prefixes with active ECMP routes
------------------------------------------------------------------------------------------------------
```
on cEOS
```
wan3(config-if-Et1)#sh ip route

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

 O        1.1.1.1/32 [110/10] via 192.1.3.1, Ethernet2
 O        1.1.1.2/32 [110/10] via 192.1.2.1, Ethernet1
 O        192.1.1.0/24 [110/11] via 192.1.2.1, Ethernet1
                                via 192.1.3.1, Ethernet2
 C        192.1.2.0/24 is directly connected, Ethernet1
 C        192.1.3.0/24 is directly connected, Ethernet2
```