# Arista AVD Configuration overview

## Overview

[Arista Networks](https://www.arista.com/) supports Ansible for managing devices running the EOS operating system natively through eapi or CloudVision Portal (CVP). This collection includes a set of ansible roles and modules to help kick-start your automation with Arista. The various roles and templates provided are designed to be customized and extended to your needs!

![Arista Validated Design](docs/medias/../../medias/ansible-avd-overview.gif)

In this post, we will go through all configuration steps to generate [EVPN/VXLAN configuration](https://www.arista.com/custom_data/downloads/?f=/support/download/DesignGuides/EVPN_Deployment_Guide.pdf) for EOS devices. Because some files might be very verbose, we demo repository is available to get complete content of all files: [inetsix/demo-avd-evpn-eve-ng](https://github.com/inetsix/demo-avd-evpn-eve-ng). Idea is not to cover all EVPN/VXLAN feaures, but go through __AVD__ to see how to implement.

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

To support this post, we will use the following topology running on EVE-NG server:

![Lab Topology](docs/../medias/ansible-avd-lab-topology.png)

## Inventory file.

In our inventory, let's list our devices:

- `AVD2_FABRIC` represents complete fabric topology we are going to configure with AVD.
- `AVD2_SPINES` is a group where all spine devices belongs.
- `AVD2_L3LEAFS`: is a group to locate all VTEP devices (LEAFs running EVPN/VXLAN). Part of this group, we create one sub-group for every LEAF (single or MLAG) as highlighted below.
  - `AVD_LEAF1` represent LEAF for POD01 and is for an MLAG pair.
  - `AVD_LEAF3` represent a single LEAF outside of MLAG.
- `AVD2_L2LEAFS` represent Aggregation layer in our current design. This layer is optional, but in our design it is in place. It works like for `AVD2_L3LEAFS` group.
- `AVD2_TENANTS_NETWORKS` is a special group housing 2 groups already configured: `AVD2_L{2|3}LEAFS`. This group will configure VNI/VLAN across the fabric, so we want to make leafs part of the configuration.
- `AVD2_SERVERS` as similar behavior to previous group. Its goal is to configure downlinks to compute nodes.

```yaml
# vim inventories/inetsix-eapi/inventory.yml
---
AVD2:
  children:
    AVD2_FABRIC:
      children:
        AVD2_SPINES:
          hosts:
            AVD2-SPINE1:
              ansible_port: 8001
[... output truncated ...]
        AVD2_L3LEAFS:
          children:
            AVD2_LEAF1:
              hosts:
                AVD2-LEAF1A:
                  ansible_port: 8011
                AVD2-LEAF1B:
                  ansible_port: 8012
            AVD2_LEAF3:
              hosts:
                AVD2-LEAF3A:
                  ansible_port: 8017
[... output truncated ...]
        AVD2_L2LEAFS:
          children:
            AVD2_L2LEAF1:
              hosts:
                AVD2-AGG01:
                  ansible_port: 8021
[... output truncated ...]
    AVD2_TENANTS_NETWORKS:
      children:
        AVD2_L3LEAFS:
        AVD2_L2LEAFS:
    AVD2_SERVERS:
      children:
        AVD2_L3LEAFS:
        AVD2_L2LEAFS:
```

In this file, you can also define management address of all devices. Because in this lab we use Jumphost, we will configure this address in the next section, but at least we configured on which port we will connect to eAPI using `ansible_port` variable.

## AVD Variables

Based on inventory we did in the previous section, it is time to create `group_vars`.

### Generic Fabric Information

All the documentation is available here, but below is a short example. All this information will be configured on all devices.

```yaml
# vim inventories/inetsix-eapi/group_vars/AVD2.yml
---
# local users
local_users:
  admin:
    privilege: 15
    role: network-admin
    sha512_password: "..."

  demo:
    privilege: 15
    role: network-admin
    sha512_password: "..."

# OOB Management network default gateway.
mgmt_gateway: 10.73.254.253
mgmt_destination_networks:
  - 10.255.2.0/24
  - 10.255.3.0/24
  - 0.0.0.0/0

# dns servers.
name_servers:
  - 1.1.1.1
  - 8.8.8.8

# NTP Servers IP or DNS name, first NTP server will be prefered, and sourced from Managment VRF
ntp_servers:
  - uk.pool.ntp.org
  - fr.pool.ntp.org
```

### Configure Fabric topology

Fabric topology is configured in [`inventories/inetsix-eapi/group_vars/AVD2_FABRIC.yml`](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/master/inventories/inetsix-eapi/group_vars/AVD2_FABRIC.yml) which is file that covers `AVD2_FABRIC` group we defined in [inventory](#inventory-file). This file contains all the base information to create initial configuration:

You can also refer to [__Arista Validated Design__ documentation](https://github.com/aristanetworks/ansible-avd/blob/devel/ansible_collections/arista/avd/roles/eos_l3ls_evpn/README.md#fabric-topology-variables) to get a description of every single option available.

- Subnet to use for underlay, loopback and vtep:

```yaml
# Point to Point Network Summary range, assigned as /31 for each
# uplink interfaces
# Assign range larger then total [spines * total potential leafs * 2]
underlay_p2p_network_summary: 172.31.255.0/24

# IP address range for evpn loopback for all switches in fabric,
# assigned as /32s
# Assign range larger then total spines + total leafs switches
overlay_loopback_network_summary: 192.168.255.0/24

# VTEP VXLAN Tunnel source loopback IP for leaf switches, assigned in /32s
# Assign range larger then total leaf switches
vtep_loopback_network_summary: 192.168.254.0/24
```

- MLAG IP information

```yaml
# mlag pair IP assignment - assign blocks - Assign range larger then
# total spines + total leafs switches
mlag_ips:
  leaf_peer_l3: 10.255.251.0/24
  mlag_peer: 10.255.252.0/24
```

Then, you have to describe devices for each role. Don't forget to set management IP here.

- Spine devices

```yaml
# Spine Switches
spine:
  platform: vEOS-LAB
  bgp_as: 65001
  # defines the range of acceptable remote ASNs from leaf switches
  leaf_as_range: 65101-65132
  nodes:
    AVD2-SPINE1:
      id: 1
      mgmt_ip: 10.73.254.1/24
    AVD2-SPINE2:
      id: 2
      mgmt_ip: 10.73.254.2/24
```

- VTEP or L3LEAF devices

```yaml
l3leaf:
  defaults:
    virtual_router_mac_address: 00:1c:73:00:dc:01
    platform: vEOS-LAB
    bgp_as: 65100
    spines: [AVD2-SPINE1, AVD2-SPINE2]
    uplink_to_spine_interfaces: [Ethernet1, Ethernet2]
    mlag_interfaces: [Ethernet3, Ethernet4]
[... output truncated ...]
  node_groups:
    AVD2_LEAF1:
      bgp_as: 65101
      nodes:
        AVD2-LEAF1A:
          id: 1
          mgmt_ip: 10.73.254.11/24
          # Interface configured on SPINES to connect to this leaf
          spine_interfaces: [Ethernet1, Ethernet1]
        AVD2-LEAF1B:
          id: 2
          mgmt_ip: 10.73.254.12/24
          # Interface configured on SPINES to connect to this leaf
          spine_interfaces: [Ethernet2, Ethernet2]
[... output truncated ...]
```

- Aggregation switches: Configuration process is similar to L3LEAFS.

Complete documentation of all available variables is available in [__Arista Validated Design__ documentation](https://github.com/aristanetworks/ansible-avd/blob/devel/ansible_collections/arista/avd/roles/eos_l3ls_evpn/README.md#fabric-topology-variables). You can also look at [variables part of the demo repo](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/3a461e0ce25afb25d4e51f620bafd324018c1189/inventories/inetsix-eapi/group_vars/AVD2_FABRIC.yml) we are using for this post.

> As you can see here, if you want to change name in your inventory, you have to edit filename in group_vars to reflect that change as well as content of this fabric file.

### Configure device type

In each variable file related to a type of devices, we have to instruct AVD what is the role of our devices.

```yaml
---
type: spine     # Must be either spine|l3leaf|l2leaf
```

### Configure VNI/VLAN across the Fabric.

AVD supports mechanism to create VLANs and VNIs and enable traffic forwarding in your overlay. In current version (`v1.0.2`), only following design listed below are supported:

- L2 VLANs
- Symetric IRB model

Model defines a set of tenants (user's defined) where you can configure VRF or `l2vlans` or a mix of them. Let's take a look at how we configure such services.

All these configurations shall be configured in file [`AVD2_TENANTS_NETWORKS.yml`](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/3a461e0ce25afb25d4e51f620bafd324018c1189/inventories/inetsix-eapi/group_vars/AVD2_L3LEAFS.yml)

#### L2 Services

Configure a pure L2 service using `EVPN route type 2` only:

```yaml
---
tenants:
  # Tenant B Specific Information - Pure L2 tenant
  Tenant_B:
    mac_vrf_vni_base: 20000
    l2vlans:
      201:
        name: 'B-ELAN-201'
        tags: [DC1]
```
Tag option allows to configure VLAN only on a subset of the fabric: all devices with this tag will be configured with this vlan. To configure device `TAGS` and `TENANTS` options, go to [__Arista Validated Design__ documentation](https://github.com/aristanetworks/ansible-avd/blob/devel/ansible_collections/arista/avd/roles/eos_l3ls_evpn/README.md#fabric-topology-variables)

In this configuration, VLAN will be created with a tag of `201` and its attached VNI will be configured with `20201`

```
AVD2-LEAF1A#show vlan 201
VLAN  Name                             Status    Ports
----- -------------------------------- --------- -------------------------------
201   B-ELAN-201                       active    Po3, Po5, Vx1


AVD2-LEAF1A#show bgp evpn vni 20201
BGP routing table information for VRF default
Router identifier 192.168.255.3, local AS number 65101
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

         Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 192.168.255.3:20201 mac-ip 20201 5001.0011.0000
                                -                     -       -       0       i
 * >     RD: 192.168.255.3:20201 imet 20201 192.168.254.3
                                -                     -       -       0       i
 * >Ec   RD: 192.168.255.5:20201 imet 20201 192.168.254.5
                                192.168.254.5         -       100     0       65001 65102 i
[... output truncated ...]
```

#### Symetric IRB model

Configure IRB symetric model, use following structure:

```yaml
tenants:
  # Tenant A Specific Information - VRFs / VLANs
  Tenant_A:
    mac_vrf_vni_base: 10000
    vrfs:
      TENANT_A_PROJECT01:
        vrf_vni: 11
        svis:
          110:
            name: 'PR01-DMZ'
            tags: [DC1]
            enabled: true
            ip_address_virtual: 10.1.10.254/24
          111:
            name: 'PR01-TRUST'
            tags: [POD02]
            enabled: true
            ip_address_virtual: 10.1.11.254/24
      TENANT_A_PROJECT02:
        vrf_vni: 12
        vtep_diagnostic:
          loopback: 100
          loopback_ip_range: 10.1.255.0/24
        svis:
          112:
            name: 'PR02-DMZ-GREEN'
            tags: [POD01]
            enabled: true
            ip_address_virtual: 10.1.12.254/24
```

Example will create 2 VRFs :

- `TENANT_A_PROJECT01`
- `TENANT_A_PROJECT02`

In `TENANT_A_PROJECT01`, 2 subnets are created and deployed on devices matching TAGS `DC1` or `POD02`:

- `10.1.10.0/24` with vlan `110` and vni `10110`
- `10.1.11.0/24` with vlan `111` and vni `10111`

In case you deployed this VRF on a MLAG VTEP, an additional vlan is created to allow L3 synchronization within VRF. This vlan is automatically generated with this algorithm:

```jinja2
{{ mlag_ibgp_peering_vrfs.base_vlan + (tenants[tenant].vrfs[vrf].vrf_vni - 1) }}
```

In addition to that, each EOS devices will allocate a dynamic VLAN per VRF to support __L3 VNI__

```
AVD2-LEAF1A#show vlan
VLAN  Name                             Status    Ports
----- -------------------------------- --------- -------------------------------
1     default                          active    Et6, Et7, Et8, PEt6, PEt7, PEt8
110   PR01-DMZ                         active    Cpu, Po3, Po5, Vx1
112   PR02-DMZ-ORANGE                  active    Cpu, Po3, Vx1
201   B-ELAN-201                       active    Po3, Po5, Vx1
1008* VLAN1008                         active    Cpu, Po3, Vx1
1009* VLAN1009                         active    Cpu, Po3, Vx1
3010  MLAG_iBGP_TENANT_A_PROJECT01     active    Cpu, Po3
3011  MLAG_iBGP_TENANT_A_PROJECT02     active    Cpu, Po3
4093  LEAF_PEER_L3                     active    Cpu, Po3
4094  MLAG_PEER                        active    Cpu, Po3

* indicates a Dynamic VLAN


AVD2-LEAF1A#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN        Source       Interface           802.1Q Tag
----------- ----------- ------------ ------------------- ----------
11          1008*       evpn         Vxlan1              1008
12          1009*       evpn         Vxlan1              1009
10110       110         static       Port-Channel5       110
                                     Vxlan1              110
10112       112         static       Vxlan1              112
20201       201         static       Port-Channel5       201
                                     Vxlan1              201
```

In `TENANT_A_PROJECT02`, we can also see an optional feature named __`vtep_diagnostic`__. This option allows you to create a loopback in this VRF and do some connectivity test.

### Configure downlinks

As we have configured L3LS fabric, EVPN/VXLAN overlay, services, it is now time to configure ports to connect servers. Ports should be configured in [`AVD2_SERVERS.yml`](https://github.com/inetsix/demo-avd-evpn-eve-ng/blob/3a461e0ce25afb25d4e51f620bafd324018c1189/inventories/inetsix-eapi/group_vars/AVD2_SERVERS.yml).

You first have to configure port profile. it is basically a description of how the port will be configured (`access` or `trunk`) and which set of vlan(s) will be configured

```yaml
---
port_profiles:
  TENANT_A_B:
    mode: trunk
    vlans: "110-111,201"
  A-PR01-DMZ:
    mode: access
    vlans: "110"
```

> This section uses vlan-id so all of these entries must be configured in [__TENANTS__](#configure-vnivlan-across-the-fabric) file

Then, create port mapping on a per server.

#### Single home server

If server is connected to only one leaf to the fabric, following template can be used

```yaml
servers:
  A-PR01-DMZ-POD01:     # Server name
    rack: POD01         # Informational RACK
    adapters:
      - type: nic
        server_ports: [Eth0]        # Server port to connect
        switch_ports: [Ethernet3]   # Switch port to connect server
        switches: [DC1-AGG01]       # Switch to connect server
        profile: A-PR01-DMZ         # Port profile to apply
```

Whereas most of the information are purely optional as not used by AVD, the last 3 entries are required:

- `switch_ports`: Will be used to configure correct port on the switch.
- `switches`: Must be switch name defined in your inventory.
- `profile`: Profile created previously.

#### Server connected to MLAG

In case of connection to MLAG, data structure is the same and only difference is we need to add information about Port-Channel to configure.

```yaml
servers:
  DCI_RTR01:
    rack: DCI
    adapters:
      - server_ports: [Eth1, Eth2]
        switch_ports: [Ethernet5, Ethernet5 ]
        switches: [SITE01-BL01A, SITE01-BL01B]
        profile: A-PR01-DMZ
        port_channel:
          state: present
          description: PortChannel5
          mode: active
```

## Configure tasks for AVD

We will now configure AVD tasks you may use in your playbook:


### Create directory structure

AVD comes with a role to [generate your folder structure](https://github.com/aristanetworks/ansible-avd/tree/devel/ansible_collections/arista/avd/roles/build_output_folders).

```yaml
tasks:
    - name: build local folders
      tags: [build]
      import_role:
        name: arista.avd.build_output_folders
      vars:
        fabric_dir_name: '{{fabric_name}}'
```

### Transform EVPN datamodel to device data model

AVD provides role [__`eos_l3ls_evpn`__](https://github.com/aristanetworks/ansible-avd/blob/devel/ansible_collections/arista/avd/roles/eos_l3ls_evpn/README.md) role to generate intend YAML device configuration:

```yaml
  tasks:
    - name: generate intend variables
      tags: [build]
      import_role:
        name: arista.avd.eos_l3ls_evpn
```

### Generate device configuration and documentation

After device data have been generated, AVD can build EOS configuration as well as documentation in markdown format.

```yaml
  tasks:
    - name: generate device intended config and documention
      tags: [build]
      import_role:
        name: eos_cli_config_gen
```
