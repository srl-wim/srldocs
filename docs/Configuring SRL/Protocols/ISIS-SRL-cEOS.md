# Configuring ISIS between Nokia SRL and Arista cEOS

This note describes how to setup isis between Nokia SRLinux and Arista cEOS. We use a triangle topology with 2 SRL and 1 cEOS device as an example

```
         WAN1 (SRL) (e1-1) -------(e1-1) WAN2 (SRL)
            (e1-2)                        (e1-2)
               |                            |
               +--(Et2) WAN3 (cEOS) (Et1)---+
```

We configure both IPv4 and IPv6 to show both address families working through ISIS


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

Before we can configure ISIS we should enable the router instance on the various devices. On SRL we create a network-instance, which is like a VRF-Lite and can be used for the global routing table. In cEOS we enable ip routing and ipv6 routing.

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

## Configuring ISIS

After the interfaces are configured we will configure an ISIS with level-L1-L2 capability between the various routers.

### WAN1 - SRL router
On SRL we create a ISIS protocol instance lab

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

## Viewing BGP state

Now that we configured OSPF on the various devices we can look at the operational state of the various devices

### BGP neighbor

On SRL devices

```
```

On cEOS device

```
```

### Routing table

on SRL devices

```
```

on cEOS device

```
```

## Full cEOS config

```
wan3(config-router-bgp)#sh running-config
! Command: show running-config
! device: wan3 (cEOSLab, EOS-4.24.2.1F-18613460.42421F (engineering build))
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
agent Bfd shutdown
agent PowerManager shutdown
agent LedPolicy shutdown
agent Thermostat shutdown
agent PowerFuse shutdown
agent StandbyCpld shutdown
agent LicenseManager shutdown
!
hostname wan3
!
spanning-tree mode mstp
!
no aaa root
!
interface Ethernet1
   no switchport
   ip address 192.1.2.2/24
   ipv6 address 2000:192:2::2/64
!
interface Ethernet2
   no switchport
   ip address 192.1.3.2/24
   ipv6 address 2000:192:3::2/64
!
ip routing
!
ipv6 unicast-routing
!
router bgp 300
   neighbor 192.1.2.1 remote-as 200
   neighbor 192.1.2.1 maximum-routes 12000
   neighbor 192.1.3.1 remote-as 100
   neighbor 192.1.3.1 maximum-routes 12000
!
end
```