# Building an SRL Agent

## Introduction

SR Linux is an open NOS that can be extended through the use of Agents. Agents are applications that can be build by anyone in almost any programming language ([golang](https://golang.org), [python](https://www.python.org), [c](https://en.wikipedia.org/wiki/C_(programming_language)), [c++](https://www.cplusplus.com), [java](https://www.java.com/), [javascript](https://www.javascript.com), [rugby](https://www.ruby-lang.org/), [dart](https://dart.dev), [PHP](https://www.php.net)), to extend the SRL NOS and make it suitable for your specific environment. Multiple use case can be considered and the capabilities are endless. E.g.
- Operations
- Telemetry
- Event Handling
- Routing Protocols
- Synthetic OAM
- Security
- Cloud
- etc

In this document we detail the basics of how such agents can be build, packaged and used in production. Through a [hello world](https://www.google.com) example we illustrate the steps


## YANG file

The first step is defining how the agent should be configured, how it is exposing its state and how it is notifying the ouside world about important information it sees. SRL is using YANG, which is a data modeling language defined in IETF, for describing the configuration and state elements of the agent. YANG is defined in [RFC6020](https://tools.ietf.org/html/rfc6020) (YANG 1.0) and [RFC7950](https://tools.ietf.org/html/rfc7950) (YANG 1.1).

The base SRL data YANG files can be found here:
[Base SRL YANG modes]()

To Illustrate how to build a YANG file an example is shown below based on a simple hello world example:
```
module helloworld {

    yang-version 1.1;

    // namespace
    namespace "urn:srl_ndk_apps/helloworld";

    prefix "srl_ndk_apps-helloworld";

    revision "2020-06-05" {
        description "Initial revision";
    }

    grouping helloworld-top {
        description "Top level grouping for hello world sample app";

        container hello {
            presence "Top-level container for the hello world sample app";
            description "Top level enclosing container for helloworld sample app
                         config and operational state data";

            leaf name {
                description "Who am I saying hello to?";
                type string {
                    length 0..255;
                    pattern '[A-Za-z0-9]*';
                }
            }

            leaf response {
                config false;
                description "Response to input";
                type string;
            }
        }
    }

    uses helloworld-top;
}
```

You can build of course more advanced YANG models and there is many more parameters defined in the YANG data modeling language, so it is good to check out [RFC6020](https://tools.ietf.org/html/rfc6020) (YANG 1.0) and [RFC7950](https://tools.ietf.org/html/rfc7950) (YANG 1.1).

I just wanted to explain some basic concepts to get you started.

Elements used in the example are:
- module: name of the module, here we use the name of the agent
- yang-version: We use YANG1.1 as defined in [RFC7950](https://tools.ietf.org/html/rfc7950)
- namespace: Each module is bound to a namespace to make them unique in your environemnt. Hence it is good practice to use the agent name in the namespace element.
- prefix: The "prefix" statement is used to define the prefix associated with the module and its namespace. It is used to access the module
- revision: specifies the editorial revision history of the module
- grouping: The "grouping" statement is used to define a reusable block of nodes, which may be used locally in the module or submodule, and by other modules that import from it
- container: defines an interior data node in the schema tree, in this example hello
- leaf: used to define a leaf node in the schema tree
    - config: false, means this is a state 
    - type: defines the types, similar to programming languages like string, integers, enums, etc
- uses: 

By using pyang you can validate your yang file and check for inconsistencies. If the result is positive, you can start writing your code logic.

command:
```
pyang -f tree -p . helloworld.yang
```
output:
```
module: helloworld
  +--rw hello!
     +--rw name?       string
     +--ro response?   string
```

## Code

Now that we understand how the data of the application is modeled we have to write code that performs the specific logic for you application.

As mentioned in the beginning, almost any programming language can be used: [golang](https://golang.org), [python](https://www.python.org), [c](https://en.wikipedia.org/wiki/C_(programming_language)), [c++](https://www.cplusplus.com), [java](https://www.java.com/), [javascript](https://www.javascript.com), [rugby](https://www.ruby-lang.org/), [dart](https://dart.dev), [PHP](https://www.php.net)

TO BE EXTENDED

## YAML file

SRL has a build in system management that handle the lifecycle of the agent. The configuration parameters are defined in a YAML file.
An example is here:

```
hello_world:
   path: /opt/srlinux/usr/bin/
   launch-command: ./helloworld
   search-command: ./helloworld
   failure-threshold: 100
   failure-action: "wait=60"
   wait-for-config: Yes
   yang-modules:
       names: 
           - "helloworld"
       source-directories:
           - "/opt/helloworld/yang/"
```

Some clarifiction of the information we see in the example
- hello_world: agent name 
- path: the directory where the agent binary is located
- launch-command: the command used to start the applciation
- search-command: the command used to search the application
- wait-for-config: ensures the agent is not started unless there is a YANG configuration for this agent in the system. it allows to optimize resources and dont run processes when not needed
- failure-threshold and failure-action: failure handling and back-off for the system when the agent fails
- yang-modules: defines where the yang files for this agent are located
    - name: yang files
    - source-directories: location of the yang files

## Packaging

To package your application up in a rpm package you can use e.g. [nfpm](https://github.com/goreleaser/nfpm) a simple deb and rpm packager written in [golang](https://golang.org).

In order to us [nfpm](https://github.com/goreleaser/nfpm) define a nfpm.yml file like this
```
# nfpm example config file
name: "gohelloworld"
arch: "amd64"
platform: "linux"
version: "v1"
section: "default"
priority: "extra"
replaces:
- gohelloworld
provides:
- gohelloworld
maintainer: "Bruce Wallis <bruce.wallis@nokia.com>"
description: |
  Simple hello world agent written in go.
vendor: "Nokia"
license: "BSD 2"
bindir: "/opt/srlinux/usr/bin/"
files:
  ./helloworld: "/opt/srlinux/usr/bin/helloworld"
  ./helloworld.yang: "/opt/helloworld/yang/helloworld.yang"
  ./helloworld_config.yml: "/etc/opt/srlinux/appmgr/helloworld_config.yml"
config_files:
overrides:
  rpm:
    scripts:
```

The following objects should be defined:
- name: agent rpm package name + add the same name in replaces/provides
- maintainer: person who maintains this package
- vendor
- license
- bindir: location of the binary, which shouls match with the YML file you create before
- files: the files and their respective location on the SRL system; this would be the binary file, the yang files and the YAML file for the lifecycle management of the app

Generating the rpm file:
```
nfpm pkg --packager rpm 
```
this generates the rpm package
```
gohelloworld-1.0.0.x86_64.rpm
```

## Installing

### RPM

Using the rpm package it is easy to install by copying it in the system and using rpm install:

```
scp gohelloworld-1.0.0.x86_64.rpm admin@<ip address of srl system>:/tmp
sudo rpm -U /tmp/gohelloworld-1.0.0.x86_64.rpm
```

You can of course automate this using ansible or terraform or another automation solution

### ANSIBLE

[Ansible](https://www.ansible.com) is another way to install the files, but you have to mange the files individually. An example is available in this repo [ansible-srl-agent install](https://github.com/srl-wim/ansible-srl-agent-install).


## Configuration

Now that your agent is installed in the system we have to activate it in the system. We do this in the following way.

login into the system
```
ssh admin@<ip address> 
```
First we need to load the agent:
```
/ tools system app-management application app_mgr reload
```
When you show the applications running on the system the agent should be visible
```
A:ant1-dc-fab-f1p1-leaf1# show system application
  +------------------+---------+---------+----------------------+--------------------------+
  |       Name       |   PID   |  State  |       Version        |       Last Change        |
  +==================+=========+=========+======================+==========================+
  | aaa_mgr          | 604     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.882Z |
  | acl_mgr          | 616     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.883Z |
  | app_mgr          | 524     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.945Z |
  | arp_nd_mgr       | 628     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.883Z |
  | bfd_mgr          | 639     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.883Z |
  | bgp_mgr          | 1121    | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:53.176Z |
  | chassis_mgr      | 648     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.883Z |
  | dev_mgr          | 541     | running |                      | 2020-07-28T16:58:52.199Z |
  | dhcp_client_mgr  | 660     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.884Z |
  | dnsmasq-mgmt     | 1359    | running |                      | 2020-07-28T16:58:56.986Z |
  | fib_mgr          | 669     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.884Z |
  | gnmi_server      | 1147    | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:53.200Z |
  | hello_world      | 1342230 | running | v19.11.7-76-ged9563a | 2020-07-30T08:53:21.092Z |
  | idb_server       | 565     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.424Z |
  | json_rpc         | 1149    | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:53.208Z |
  | linux_mgr        | 681     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.884Z |
  | lldp_mgr         | 1144    | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:53.190Z |
  | log_mgr          | 690     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.884Z |
  | mgmt_server      | 700     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.884Z |
  | mpls_mgr         | 719     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.885Z |
  | net_inst_mgr     | 739     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.885Z |
  | oam_mgr          | 766     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.885Z |
  | plcy_mgr         | 791     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.886Z |
  | qos_mgr          | 1113    | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:53.128Z |
  | sdk_mgr          | 819     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.886Z |
  | sshd-mgmt        | 1349    | running |                      | 2020-07-28T16:58:56.983Z |
  | static_route_mgr | 852     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.887Z |
  | supportd         | 532     | running |                      | 2020-07-28T16:58:52.087Z |
  | testagent        | 16435   | running | v19.11.7-76-ged9563a | 2020-07-28T17:14:17.302Z |
  | xdp_cpm          | 889     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.887Z |
  | xdp_lc_1         | 927     | running | v19.11.7-76-ged9563a | 2020-07-28T16:58:52.887Z |
  +------------------+---------+---------+----------------------+--------------------------+
```
You see the helloworld agent appear in the application list. It has a PID of 1342230 in this example. If you defined in the YML file that the helloworld agent should have waited for configuration, the PID would not have been allocated since there was no configuration in the system and hence the agent process would not have started.
Next step is configuring the agent
Given SRL is a fully transactional system you first have to enter in the candidate datastore.
```
enter candidate
```
Next you navigate through the CLI based on the YANG tree you defined.
```
/ hello
commit stay
```
You did not configure anything so far but you triggered the agent to look at the data. When looking at the state of the agent you would see the following
```
info from state
```
output
```
    name ""
    response "Hello, do tell me your name"
```
The agent is waiting for a name to be configured
```
/ hello
name wim
```
Commit the changes
```
commit stay
```
When you look at the state again you should see the following information updates
```
info from state
```
output
```
    name wim
    response "Hello, wim"
```

## Consuming the Agent

Now that the agent is configured and running lets consume the agent from the outside worrld through GNMI. We use the following [gnmi client](https://gnmic.kmrd.dev) to show the examples.

SRL is configured for GNMI in the following way:
```
--{ candidate shared }--[ system gnmi-server ]--
A:ant1-dc-fab-f1p1-leaf1# info
    admin-state enable
    timeout 7200
    rate-limit 60
    session-limit 20
    network-instance mgmt {
        admin-state enable
        use-authentication true
        port 57400
        tls-profile tls-profile-1
    }
    unix-socket {
        admin-state disable
        use-authentication true
    }
--{ candidate shared }--[ system gnmi-server ]--
```
We use port 57400 in the mgmt network instance.

Lets first get the information
```
gnmic -a 10.1.1.2:57400 -u admin -p admin --skip-verify -e json_ietf get --path /hello
```
output
```json
{
  "source": "10.1.1.2:57400",
  "timestamp": 1596100364586956453,
  "time": "2020-07-30T09:12:44.586956453Z",
  "updates": [
    {
      "Path": "/helloworld:hello",
      "values": {
        "helloworld:hello": {
          "name": "wim",
          "response": "Hello, wim"
        }
      }
    }
  ]
}
```
we can also get streaming telemetry from the system
```
gnmic -a 10.1.1.2:57400 -u admin -p admin --skip-verify -e json_ietf sub --path "/hello"
```
output
```json
{
  "source": "10.1.1.2:57400",
  "subscription-name": "default",
  "timestamp": 1596101041094828877,
  "time": "2020-07-30T09:24:01.094828877Z",
  "updates": [
    {
      "Path": "helloworld:hello",
      "values": {
        "helloworld:hello": {
          "name": "wim",
          "response": "Hello, wim"
        }
      }
    }
  ]
}
```
when you would update the name in the hello yang tree you will be notified on the change of the data.
```
--{ candidate shared }--[ hello ]--
A:ant1-dc-fab-f1p1-leaf1# name bruce
--{ candidate shared }--[ hello ]--
A:ant1-dc-fab-f1p1-leaf1# commit say
```
GNMI subscription output
```json
{
  "source": "10.1.1.2:57400",
  "subscription-name": "default",
  "timestamp": 1596101063590975148,
  "time": "2020-07-30T09:24:23.590975148Z",
  "updates": [
    {
      "Path": "helloworld:hello",
      "values": {
        "helloworld:hello": {
          "name": "bruce",
          "response": "Hello, bruce"
        }
      }
    }
  ]
}
```

As you can see hopefully SRL is a very open NOS which provides you a complete extendable framework. SRL leverages YANG as the data modeling language, which is very easy to consume using GNMI and you get a full managable and observable system that can be tailored according your needs. The open NOS.

## Logging

Information that the agent is providing is also send to /var/log/srlinux/stdout/<agentname>.log and can be sent to syslog, etc.

## Conclusion

I hope you enjoyed the read/experience of the process to go through to extend the SRL NOS using agents. Now it is time to find the time, passion and pleasure to knock yourself out and create your own destiny in networking by leveraging the SRL NOS.