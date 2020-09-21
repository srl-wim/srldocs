# MAC-VRF with IRB

In order to optimize the use of ip interfaces and subnets SRL support the concept of a MAC-VRF/IRB, which essentially is a bridged domain connected to an IP router interface. MAC-VRF/IRB allows to optimize the connectivity to client (e.g. server). With MAC-VRF/IRB we allow to aggregate all client in a bridged domain and hence the need for a dedicated IP interface is avoided.

We will illustrate this with an example.

In this example we have a SRL switch/router (leaf1) that connects 2 client through a MAC-VRF/IRB. We use 2 linux clients in this example, client 1 connected to port ethernet-1/5 on vlan 10 and client 2 connected to port ethernet-1/6 on vlan 20.

We use the container environment for this setup.

## Configuring MAC-VRF instance and sub-interfaces using IRB

### Configure VLAN interfaces

on SRL leaf1 interface ethernet-1/5

```
    admin-state enable
    vlan-tagging true
    subinterface 1 {
        type bridged
        admin-state enable
        vlan {
            encap {
                single-tagged {
                    vlan-id 10
                }
            }
        }
    }
```
the important parameters are:
- vlan-tagging: true ; allows for VLAN enabled interfaces
- subinterface type: bridged; Bridged sub-interfaces can be associated to MAC-VRF instances, that allow MAC learning and layer-2 forwarding.
- vlan-id: 10 is used on this interface to client1

On SRL leaf1 interface ethernet-1/6 we have a similar configuration, but we use vlan 20 instead of vlan 10, to show that SRL support local VLAN significance.

```
    admin-state enable
    vlan-tagging true
    subinterface 1 {
        type bridged
        admin-state enable
        vlan {
            encap {
                single-tagged {
                    vlan-id 20
                }
            }
        }
    }
```

### Configure IRB interface

Now that we have the vlan interfaces configured, we can create an IRB interface. The IRB interface is like a loopback interface. hence you dont see an association with the physical ethernet interface

```
/ interface irb1
```

IRB interface configuraton

```
    admin-state enable
    subinterface 1 {
        admin-state enable
        ipv4 {
            address 192.168.10.1/24 {
            }
        }
        ipv6 {
            address 3000:10::1/64 {
            }
        }
    }
```

The IRB interface is of type routed which is the default in the system and hence you don't need to explicitly configure it. Also you create your IPv4 and IPv6 interface IP addresses, which will be the gateway IP(s) for the linux clients that are connected tot he IRB instance

### Configure MAC-VRF

The next step is creating the MAC-VRF network instance and associate the bridged and IRB interfaces to it.

```
/ network-instance mac-vrf10
```

Associate interfaces

```
    type mac-vrf
    admin-state enable
    interface ethernet-1/5.1 {
    }
    interface ethernet-1/6.1 {
    }
    interface irb1.1 {
    }
```

### Attach IRB interface to the default network-instance

Associate the same IRB interface to the network-instance default in order to attach it to the router context.

```
/ network-instance default
```

Associate the IRB interface on top of the other interfaces which connected the leaf to the spine layer e.g.
```
    type ip-vrf
    admin-state enable
    description "GRT / Default VRF"
    ip-forwarding {
        receive-ipv4-check true
        receive-ipv6-check true
    }
    interface ethernet-1/1.1 {
    }
    interface ethernet-1/2.1 {
    }
    interface ethernet-1/3.1 {
    }
    interface ethernet-1/4.1 {
    }
    interface irb1.1 {
    }
    interface lo0.1 {
    }
```

Now the SRL switch/router is ready to handle the clients so lets connect and configure the clients

## Configuring Clients

We use a docker container image which was setup through a container image

### Client1

We first spin up the container image.
```
docker run -d --privileged --name client1 --network srlinux-mgmt --hostname client1 henderiw client-alpine:1.0.0
```
After we connect it to the SRL container image using a veth pair.

Once this is done we can configure the eth1 interface which is used to connect to the SRL switch/router.

```
ethtool --offload eth1 rx off tx off
ip link add link eth1 name eth1.10 type vlan id 10
ip link set dev eth1.10 up
sysctl -w net.ipv6.conf.eth1.10.disable_ipv6=0
ip addr add 192.168.10.10/24 dev eth1.10
ip -6 addr add 3000:10::10/64 dev eth1.10
```
Lets see if this configuration got applied
```
client1:~# ifconfig eth1.10
eth1.10   Link encap:Ethernet  HWaddr DE:13:FA:CF:FF:58
          inet addr:192.168.10.10  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::dc13:faff:fecf:ff58/64 Scope:Link
          inet6 addr: 3000:10::10/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:24 errors:0 dropped:0 overruns:0 frame:0
          TX packets:24 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1646 (1.6 KiB)  TX bytes:1928 (1.8 KiB)
```

### Client2

We do the same on client 2.

```
docker run -d --privileged --name client2 --network srlinux-mgmt --hostname client2 henderiw client-alpine:1.0.0
```
After we connect it to the SRL container image using a veth pair.

Once this is done we can configure the eth1 interface which is used to connect to the SRL switch/router.

```
ethtool --offload eth1 rx off tx off
ip link add link eth1 name eth1.20 type vlan id 20
ip link set dev eth1.20 up
sysctl -w net.ipv6.conf.eth1.20.disable_ipv6=0
ip addr add 192.168.10.20/24 dev eth1.20
ip -6 addr add 3000:10::20/64 dev eth1.20
```
Lets see if this configuration got applied
```
client2:~# ifconfig eth1.20
eth1.20   Link encap:Ethernet  HWaddr 32:64:9C:98:9E:FD
          inet addr:192.168.10.20  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::3064:9cff:fe98:9efd/64 Scope:Link
          inet6 addr: 3000:10::20/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:9 errors:0 dropped:0 overruns:0 frame:0
          TX packets:20 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:540 (540.0 B)  TX bytes:1592 (1.5 KiB)
```

### Lets verfy the connectivity

#### From CLient 1

ping SRL routed gateway
```
client1:~# ping6 3000:10::1
PING 3000:10::1 (3000:10::1): 56 data bytes
64 bytes from 3000:10::1: seq=0 ttl=64 time=3.338 ms
64 bytes from 3000:10::1: seq=1 ttl=64 time=1.423 ms
```

Ping CLient2
```
client1:~# ping6 3000:10::20
PING 3000:10::20 (3000:10::20): 56 data bytes
64 bytes from 3000:10::20: seq=0 ttl=64 time=0.312 ms
64 bytes from 3000:10::20: seq=1 ttl=64 time=0.342 ms
64 bytes from 3000:10::20: seq=2 ttl=64 time=0.354 ms
```

#### On SRL

Show MAC table
```json
A:ant1-dc-fab-f1p1-leaf1# show / network-instance mac-vrf10 bridge-table mac-table all | as json
{
  "Network": [
    {
      "Name": "mac-vrf10",
      "Mac table": [
        {
          "address": "00:01:01:FF:00:41",
          "Destination": "irb-interface",
          "Dest Index": 0,
          "Type": "irb-interface",
          "Active": true,
          "Aging": "N/A",
          "Last Update": "2020-08-05T04:33:51.000Z"
        },
        {
          "address": "32:64:9C:98:9E:FD",
          "Destination": "ethernet-1/6.1",
          "Dest Index": 8,
          "Type": "learnt",
          "Active": true,
          "Aging": "255",
          "Last Update": "2020-08-05T05:29:43.000Z"
        },
        {
          "address": "DE:13:FA:CF:FF:58",
          "Destination": "ethernet-1/5.1",
          "Dest Index": 7,
          "Type": "learnt",
          "Active": true,
          "Aging": "255",
          "Last Update": "2020-08-05T05:29:37.000Z"
        }
      ]
    }
  ],
  "Total Statistics": [
    {
      "Counter name": "Total Irb Macs",
      "Total": 1,
      "Active": 1
    },
    {
      "Counter name": "Total Static Macs",
      "Total": 0,
      "Active": 0
    },
    {
      "Counter name": "Total Duplicate Macs",
      "Total": 0,
      "Active": 0
    },
    {
      "Counter name": "Total Learnt Macs",
      "Total": 2,
      "Active": 2
    },
    {
      "Counter name": "Total Macs",
      "Total": 3,
      "Active": 3
    }
  ]
}
```
