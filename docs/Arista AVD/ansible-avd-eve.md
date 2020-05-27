# Arista AVD on EVE-NG

## Overview

Arista has published a collection named [Arista Validated Design](https://github.com/aristanetworks/ansible-avd/) to automate build and deploy an __EVPN/VXLAN__ fabric using Red Hat Ansible. This collection supports different deployment scenario such as pure EOS using eAPI or using cloudvision as a change manager.

In this post, we will see how to create a local environment to leverage __AVD Collection__ to build EVPN/VXLAN configuration for a set of devices and how to deploy configuration using EOS eAPI.

## Lab topology

Lab topology is running on a EVE-NG server with following elements:

- vEOS version: Any version from `4.21.8M` (lab currently runs `4.24.0F`)
- Out of band management: `10.255.1.0/24`
- A jumphost to isolate Out of band management from current network. Jumphost server is running `Ubuntu` but any Linux flavor should work

To make it easy to start, a baseline repository is available [on github](https://github.com/inetsix/demo-avd-evpn-eve-ng). You can clone or fork to start working with it.

![Lab Topology](medias/ansible-avd-lab-topology.png)

Topology is based on the following design:

- 1 single site: No DCI feature involved here, only __EVPN/VXLAN__ fabric spread across all nodes.
- 2 Spines
- 2 PODs running MLAG VTEP: provide a redundant connectivity from POD to the fabric.
- 2 POD using single homed VTEP
- 2 Aggregation switches each attached to its respective MLAG VTEP.
- 1 POD to emulate DCI connectivity. This POD does not run any specific DCI design and it uses to get visibility of control plane on MLAG out of any server connection.

### EOS Initial Configuration

Management interface of all Arista EOS devices are connected to a `Private Cloud` in our EVE-NG topology and they are all using the follwing address plan:

- Spine 01: `10.73.254.1`
- Spine 02: `10.73.254.2`
- Leaf 01A: `10.73.254.11`
- Leaf 01B: `10.73.254.12`
- Leaf 02A: `10.73.254.13`
- Leaf02B: `10.73.254.14`
- Border 01A: `10.73.254.15`
- Border 01B: `10.73.254.16`
- Leaf 03A: `10.73.254.17`
- Leaf 04A: `10.73.254.18`
- AGG01: `10.73.254.21`
- AGG02: `10.73.254.22`

Initial configuration deployed to devices is based on the following short template:

```jinja
!
transceiver qsfp default-mode 4x10G
!
hostname {{hostname}}
ip name-server vrf MGMT 10.255.1.253
!
ntp local-interface vrf MGMT Management1
ntp server vrf MGMT 10.255.1.1 prefer
!
spanning-tree mode mstp
spanning-tree mst 0 priority 16384
!
no aaa root
!
username admin privilege 15 role network-admin secret sha512 {{password_sha512}}
!
vrf instance MGMT
!
interface Ethernet1
!
interface Ethernet2
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
!
interface Management1
   description oob_management
   vrf MGMT
   ip address {{ management_ip }}/24
!
ip routing
no ip routing vrf MGMT
!
management api http-commands
   no shutdown
   !
   vrf MGMT
      no shutdown
!
end
```

### Jumphost Configuration

Because our OOB network is completely isolated from our network, a jumphost will be configured to expose eAPI port for all EOS devices. `ens3` is configured in OOB network with ip address `10.73.254.253/24` and [an __IPtables__ configuration](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/master/iptables-port-forward.sh) is deployed to serve NAT.

This script must be executed with root permissions. Script maps port __`443`__ of EOS to a port on __`800x`__ range.

```shell
$ cat expose-eapi.sh

#!/bin/bash

echo '* Activate IP Forwarding'
sysctl -w net.ipv4.ip_forward=1

echo '* Flush Current IPTables settings'
iptables --flush
iptables --delete-chain
iptables --table nat --flush
iptables --table nat --delete-chain

echo '* Activate masquerading'

iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens2 -j MASQUERADE

echo '* Activate eAPI forwarding with base port 800x'

iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8001 -j DNAT --to-destination 10.73.254.1:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8002 -j DNAT --to-destination 10.73.254.2:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8011 -j DNAT --to-destination 10.73.254.11:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8012 -j DNAT --to-destination 10.73.254.12:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8013 -j DNAT --to-destination 10.73.254.13:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8014 -j DNAT --to-destination 10.73.254.14:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8015 -j DNAT --to-destination 10.73.254.15:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8016 -j DNAT --to-destination 10.73.254.16:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8017 -j DNAT --to-destination 10.73.254.17:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8018 -j DNAT --to-destination 10.73.254.18:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8021 -j DNAT --to-destination 10.73.254.21:443
iptables -t nat -A PREROUTING -p tcp -i ens1 --dport 8022 -j DNAT --to-destination 10.73.254.22:443

echo '* Activate Forward'
iptables -A FORWARD -p tcp -d 10.73.254.0/24 --dport 443 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

echo '* Process done'
```

You can validate everything is well configured with commands below:

#### Check NAT Settings

It is important to validate NAT settings are all set as expected

```shell
$ sudo iptables -t nat -L -n
[sudo] password for jumper:

Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8001 to:10.73.254.1:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8002 to:10.73.254.2:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8011 to:10.73.254.11:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8012 to:10.73.254.12:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8013 to:10.73.254.13:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8014 to:10.73.254.14:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8015 to:10.73.254.15:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8016 to:10.73.254.16:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8017 to:10.73.254.17:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8018 to:10.73.254.18:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8021 to:10.73.254.21:443
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8022 to:10.73.254.22:443

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0
```

#### Check access to eAPI

Once NAT as been correctly set, check you can reach your devices using jumphost address from your laptop

```shell
$ curl -i -k https://10.73.1.17:8001

HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Wed, 27 May 2020 08:09:52 GMT
Content-Type: text/html
Content-Length: 13
Connection: keep-alive
Cache-control: no-store
Cache-control: no-cache
Cache-control: must-revalidate
Cache-control: max-age=0
Cache-control: pre-check=0
Cache-control: post-check=0
Pragma: no-cache
Location: https://10.73.1.17:8001/explorer.html
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'

Page redirect
```

## Create Arista Validated Design project.

Ok, enough of lab building and we can start to play with Ansible and AVD collection.

### Create a repository

You can organize your work in many different way, but a structure I find useful is something like this:

- A folder for all your inventories with one sub-folder per inventory. An inventory folder contains all your variables for a given environment like `host_vars`, `group_vars`, `inventory.yml`
- A folder to store all playbooks. So it is easy to reuse playbooks whatever the inventory is (if you use a coherent syntax)
- ansible.cfg at the root of this repository

```shell
$ tree -L 3 -d
.
├── inventories
│   ├── inetsix-cvp
│   │   ├── group_vars
│   │   ├── host_vars
│   │   └── media
│   └── inetsix-eapi
│       ├── group_vars
│       └── medias
└── playbooks
```

From there, you can run ansible playbooks with following syntax (as an example) :

```shell
$ ansible-playbook playbooks/my-playbook -i inventories/inetsix-eapi
```

So to make it easy to start, you can clone this repository as baseline:

```shell
$ git clone https://github.com/inetsix/demo-avd-evpn-eve-ng.git
```

## Configure Arista Valdiated Design

Arista validated design can be installed in 2 different ways:

- Ansible Galaxy to get a stable version.
- Git to use latest development version.

In this post we will only use ansible-galaxy command to install this collection. If you want to use very latest version, please refer to [Github repository](https://github.com/aristanetworks/ansible-avd#installation) to see how to install using git.


```shell
# install python requirements
$ wget https://github.com/aristanetworks/ansible-avd/blob/devel/development/requirements.txt
$ pip3 install -r requirements.txt

# install collection
$ ansible-galaxy collection install arista.avd
```

### Configure Arista Validate Design variables

To generate content based on AVD collection, we have to configure 2 things:

- Inventory file (`inventory.yml`) to list devices in their respective groups
- `group_vars/` files to provide all information required.

> Information in `inventory.yml` and within `group_vars` folder are linked all together. So when changing a hostname may require to edit couple of files.

### Configure connection information

In file [`inventories/inetsix-eapi/group_vars/all.yml`](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/master/inventories/inetsix-eapi/group_vars/all.yml), use the following information:

```yaml
---
ansible_connection: httpapi
ansible_httpapi_port: '{{ansible_port}}'
ansible_httpapi_host: '{{ ansible_host }}'
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: false

ansible_network_os: eos
ansible_user: admin
ansible_ssh_pass: arista123
ansible_become: yes
ansible_become_method: enable

ansible_host: <YOUR JUMPHOST IP>
```

### Configure Arista Valdiated Design variables

You can check [this post where we detail how to use and configure AVD](./ansible-avd-config.md) to build EVPN/VXLAN configuration. And you can also read [complete documentation from the project repository](https://github.com/aristanetworks/ansible-avd/tree/devel/ansible_collections/arista/avd).

### Create a playbook to deploy

To deploy configuration, we need to create a playbook under `playbooks/` folder. This playbook will do 4 different tasks:

- Build output directories.
- Transform EVPN data into YAML files per devices.
- Build configuration and documentation.
- Deploy configuration and do backup

```yaml
---
- name: Build Switch configuration
  hosts: all
  collections:
    - arista.avd
  tasks:
    - name: build local folders
      tags: [build]
      import_role:
        name: arista.avd.build_output_folders
      vars:
        fabric_dir_name: '{{fabric_name}}'
    - name: generate intend variables
      tags: [build]
      import_role:
        name: arista.avd.eos_l3ls_evpn
    - name: generate device intended config and documentation
      tags: [build]
      import_role:
        name: arista.avd.eos_cli_config_gen
    - name: deploy configuration to device
      tags: [deploy, never]
      import_role:
         name: arista.avd.eos_config_deploy_eapi
```

As you can see, we put 2 different [ansible tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html) to be more granular:

- __`build`__: give us an option to just generate configuration and review offline all the config files.
- __`deploy`__: is set to explicit and push your new configuration to device.

```shell
$ ansible-playbook playbooks/avd-eapi-generic.yml -i inventories/inetsix-eapi/inventory.yml --tags build

```

Playbook will create all files and folder in your inventory and here is an output example:

```shell
tree -L 3 -d
.
├── configlets
├── cvp-debug-logs
├── inventories
│   └── inetsix-eapi
│       ├── config_backup
│       ├── documentation
│       ├── group_vars
│       └── intended
├── output.variables
└── playbooks
```

To push configuration to devices, just use the other tags:

```shell
$ ansible-playbook playbooks/avd-eapi-generic.yml -i inventories/inetsix-eapi/inventory.yml --tags "build, deploy"
```

## Check & Verification

> __If it is first time devices activate arBgp, it is required to reboot topology__

Connect to your devices and validate your configuration has been deployed:

```shell
AVD2-LEAF1A login: admin
Password:

AVD2-LEAF1A>en

AVD2-LEAF1A#show run
! Command: show running-config
! device: AVD2-LEAF1A (vEOS, EOS-4.24.0F)
!
! boot system flash:/vEOS-lab.swi
!
vlan internal order ascending range 1006 1199
!
transceiver qsfp default-mode 4x10G
!
[... output truncated ...]
```

You can also check LLDP neighbor:

```
AVD2-LEAF1A#show lldp neighbors
Last table change time   : 2 days, 20:37:13 ago
Number of table inserts  : 25
Number of table deletes  : 9
Number of table drops    : 0
Number of table age-outs : 0

Port       Neighbor Device ID               Neighbor Port ID           TTL
Et1        AVD2-SPINE1                      Ethernet1                  120
Et2        AVD2-SPINE2                      Ethernet1                  120
Et3        AVD2-LEAF1B                      Ethernet3                  120
Et4        AVD2-LEAF1B                      Ethernet4                  120
Et5        AVD2-AGG01                       Ethernet1                  120
Ma1        AVD2-BL01A                       Management1                120
Ma1        AVD2-LEAF2A                      Management1                120
Ma1        AVD2-LEAF3A                      Management1                120
Ma1        AVD2-AGG01                       Management1                120
Ma1        AVD2-BL01B                       Management1                120
Ma1        AVD2-LEAF4A                      Management1                120
Ma1        AVD2-AGG02                       Management1                120
Ma1        AVD2-LEAF2B                      Management1                120
Ma1        AVD2-LEAF1B                      Management1                120
Ma1        AVD2-SPINE2                      Management1                120
Ma1        AVD2-SPINE1                      Management1                120
```

Meantime, MLAG should be all set:

```shell
AVD2-LEAF1A#show mlag
MLAG Configuration:
domain-id              :          AVD2_LEAF1
local-interface        :            Vlan4094
peer-address           :        10.255.252.1
peer-link              :       Port-Channel3
hb-peer-address        :        10.73.254.12
hb-peer-vrf            :                MGMT
peer-config            :          consistent

MLAG Status:
state                  :              Active
negotiation status     :           Connected
peer-link status       :                  Up
local-int status       :                  Up
system-id              :   0e:1d:c0:f0:bd:d1
dual-primary detection :          Configured

MLAG Ports:
Disabled               :                   0
Configured             :                   0
Inactive               :                   0
Active-partial         :                   0
Active-full            :                   1

AVD2-LEAF1A#show mlag interfaces
                                                                   local/remote
  mlag     desc                       state     local     remote         status
-------- -------------------- --------------- --------- ---------- ------------
     5     AVD2_L2LEAF1_Po1     active-full       Po5        Po5          up/up
AVD2-LEAF1A#
```

Based on our setup, you can also check that L3LEAF to AGG devices are well configured:

```
AVD2-LEAF1A#show inter description| grep AGG
Et5                            up             up                 AVD2-AGG01_Ethernet1
AVD2-LEAF1A#show lacp neighbor
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting, X = state machine expired,
       D = Distributing, d = default neighbor state
                 |                        Partner
 Port    Status  | Sys-id                    Port#   State     OperKey  PortPri
------ ----------|------------------------- ------- --------- --------- -------
Port Channel Port-Channel3:
 Et3     Bundled | 8000,0c-1d-c0-f4-7b-39        3   ALGs+CD    0x0003    32768
 Et4     Bundled | 8000,0c-1d-c0-f4-7b-39        4   ALGs+CD    0x0003    32768
Port Channel Port-Channel5*:
 Et5     Bundled | 8000,0c-1d-c0-90-13-79        1   ALGs+CD    0x0001    32768

* - Only local interfaces for MLAGs are displayed. Connect to the peer to
    see the state for peer interfaces.
```

And then check your BGP sessions:

- IPv4 Underlay sessions

```
AVD2-LEAF1A#show bgp ipv4 unicast summary
BGP summary information for VRF default
Router identifier 192.168.255.3, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.255.251.1     4  65101           4910      4894    0    0    2d20h Estab   14     14
  172.31.255.0     4  65001           4907      4899    0    0    2d21h Estab   11     11
  172.31.255.2     4  65001           4897      4892    0    0    2d21h Estab   11     11
```

- EVPN Overlay sessions

```
AVD2-LEAF1A#show bgp evpn summary
BGP summary information for VRF default
Router identifier 192.168.255.3, local AS number 65101
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  192.168.255.1    4  65001           4931      4914    0    0    2d21h Estab   22     22
  192.168.255.2    4  65001           4938      4939    0    0    2d21h Estab   22     22
```

Then you are all set to continue your journey with Arista Validated design


## References

- [Arista Recommended design Guide](https://www.arista.com/en/solutions/design-guides)
- [Arista Validated Deisgn](https://github.com/aristanetworks/ansible-avd/)
- [EVE-NG website](https://www.eve-ng.net/)
- [How to configure AVD](./ansible-avd-config.md)