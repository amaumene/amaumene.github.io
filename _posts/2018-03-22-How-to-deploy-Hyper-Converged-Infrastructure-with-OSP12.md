---
layout: post
title: How to deploy Hyper Converged Infrastructure with OSP12
---

... or if you prefer Ceph Hyperconverged with OSP12.

There were a lot of changes to Red Hat OpenStack Platform (or OpenStack Heat TripleO) templates with the release of version 12. In this document I'll briefly highlight what changes you need to pay attention to when you are deploying an hyper converged infrastructure.

There is a few different way on how to achieve this, feel free to discuss mine.

**Important note**: This is for RHCeph2. There is currently several bugs related to RHCeph3. I don't recommend to deploy RHCeph3 with RHOSPd12. (https://bugzilla.redhat.com/show_bug.cgi?id=1538336)

<!-- TOC -->

- [Composable Roles](#composable-roles)
- [Network Interface Templates definition](#network-interface-templates-definition)
- [Network Interface Templates configuration](#network-interface-templates-configuration)
- [Composable Network configuration](#composable-network-configuration)
- [Network Isolation](#network-isolation)
- [Overcloud containers](#overcloud-containers)
- [Ceph Configuration](#ceph-configuration)
- [Download my templates](#download-my-templates)

<!-- /TOC -->

## Composable Roles

If you have a look at the default roles_data.yaml from the tripleo-heat-templates you'll notice a few new fields. The most important one is related to the networks being composable like roles. Before that you were tied to a specific number of networks you could configure on your overcloud nodes. By default a compute needs to have access to those networks:
```
- name: Compute
  description: |
    Basic Compute Node role
  CountDefault: 1
  networks:
    - InternalApi
    - Tenant
    - Storage
```

We need to add StorageMgmt to it like this:
```
- name: Compute
  description: |
    Basic Compute Node role
  CountDefault: 1
  networks:
    - InternalApi
    - Tenant
    - StorageMgmt
```

As before you will also need to add CephOSD service to the compute's service list like this:
```
  ServicesDefault:
    - OS::TripleO::Services::AuditD
[...]
    - OS::TripleO::Services::CephOSD
    - OS::TripleO::Services::CertmongerUser
[...]
```

## Network Interface Templates definition

Nothing new to describe here. You need to add a definition of the StorageMgmt network so it is configured on your compute node.

To briefly explain what needs to be done, you need to have somewhere defined your NIC configurations for your different nodes like this:
```
resource_registry:
  # Network Interface templates to use (these files must exist)
  OS::TripleO::Compute::Net::SoftwareConfig:
    ./single-nic-vlans/compute.yaml
  OS::TripleO::Controller::Net::SoftwareConfig:
    ./single-nic-vlans/controller.yaml
```

## Network Interface Templates configuration

And of the linked file (./single-nic-vlans/compute.yaml in this example), you need to have StorageMgmt define on a NIC like this:
```
resources:
  OsNetConfigImpl:
    type: OS::Heat::SoftwareConfig
[...]
          params:
            $network_config:
              network_config:
              - type: ovs_bridge
         [...]
                members:
                - type: interface
                  name: nic1
                  # force the MAC address of the bridge to this interface
                  primary: true
                - type: vlan
                  vlan_id:
                    get_param: InternalApiNetworkVlanID
                  addresses:
                  - ip_netmask:
                      get_param: InternalApiIpSubnet
   [...]
                - type: vlan
                  vlan_id: {get_param: StorageMgmtNetworkVlanID}
                  addresses:
                    - ip_netmask: {get_param: StorageMgmtIpSubnet}
                - type: vlan
                  vlan_id:
                    get_param: TenantNetworkVlanID
                  addresses:
                  - ip_netmask:
                      get_param: TenantIpSubnet
[...]
```

## Composable Network configuration

Before OSP12 and composable networks, you were defining networks (most probably in the network-environment.yaml file) using theses parameters:
```
  # Customize the IP subnets to match the local environment
  InternalApiNetCidr: 172.17.0.0/24
  StorageNetCidr: 172.18.0.0/24
  StorageMgmtNetCidr: 172.19.0.0/24
  TenantNetCidr: 172.16.0.0/24
  ExternalNetCidr: 10.0.0.0/24
  # Customize the VLAN IDs to match the local environment
  InternalApiNetworkVlanID: 20
  StorageNetworkVlanID: 30
  StorageMgmtNetworkVlanID: 40
  TenantNetworkVlanID: 50
  ExternalNetworkVlanID: 10
  # Customize the IP ranges on each network to use for static IPs and VIPs
  InternalApiAllocationPools: [{'start': '172.17.0.10', 'end': '172.17.0.200'}]
  StorageAllocationPools: [{'start': '172.18.0.10', 'end': '172.18.0.200'}]
  StorageMgmtAllocationPools: [{'start': '172.19.0.10', 'end': '172.19.0.200'}]
  TenantAllocationPools: [{'start': '172.16.0.10', 'end': '172.16.0.200'}]
  # Leave room if the external network is also used for floating IPs
  ExternalAllocationPools: [{'start': '10.0.0.10', 'end': '10.0.0.50'}]
```

While it is still supported, the preferred way is to use the network_data.yaml file. You need to use the -n argument to link it to your deploy command (as opposed to -r for role and -e for environment files). This file is quite straightforward to understand, you need to define your networks like this:
```
- name: StorageMgmt
  vlan: 40
  name_lower: storage_mgmt
  vip: true
  ip_subnet: '172.16.3.0/24'
  allocation_pools: [{'start': '172.16.3.4', 'end': '172.16.3.250'}]
  ipv6_subnet: 'fd00:fd00:fd00:4000::/64'
  ipv6_allocation_pools: [{'start': 'fd00:fd00:fd00:4000::10', 'end': 'fd00:fd00:fd00:4000:ffff:ffff:ffff:fffe'}]
```

This way we are not limited to the predefined OpenStack networks only.

## Network Isolation

So if we are not using the network-environment.yaml variables, and we can create our own networks, what happened to network-isolation.yaml ? It's actually quite simple. We don't use it anymore. Except we actually still need it... It will in fact be generated from jinja2 templates during the deployment. But here is the catch, you still have to link it. Yes you read that correctly. You need to link a non existing file. And you need to link it to your --templates directory. In my case it will be something like this:
```
openstack overcloud deploy --templates /home/stack/my_templates \
-r $(pwd)/roles_data.yaml \
-n $(pwd)/network_data.yaml \
-e $(pwd)/../my_templates/environments/network-isolation.yaml \
-e $(pwd)/overcloud_images.yaml \
-e $(pwd)/environment.yaml \
-e $(pwd)/storage-environment.yaml
```

where /home/stack/my_templates is my copy of /usr/share/openstack-tripleo-heat-templates/ (and this command is run in /home/stack/deployment where I copy my customised templates that I'm linking in this command).

## Overcloud containers

So far I've explained the content of roles_data.yaml, network_data.yaml and the auto-generated but still need to be linked network-isolation.yaml. As you can see from the command above, I should now explain overcloud_images.yaml. This file is describing which version of the containers you want to deploy. This is well explained in the official doc [here](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html-single/director_installation_and_usage/#Configuring-Registry_Details-Remote). That is worth to mention that you will define this file which version (2 or 3) of Ceph you want to use.

I won't describe environment.yaml either as this is where I set my variables like NTP, DNS, etc... and there is nothing new here with OSP12 (that I used).

## Ceph Configuration

With OSP12, Ceph containers are configured using the ceph-ansible module, thus the heat parameters changed too. There is actually quite a lot of changes and I refer you to this doc for more informations. Note that there is a few limitations compare to a puppet/non-containerized deployment. The biggest limitation we currently face is that only A SINGLE journal for all the OSDs on a given machine is supported. Try to remember this when you are designing architectures for your customers !

Otherwise you actually need to configured your Ceph using theses parameters:
```
  CephAnsibleDisksConfig:
    devices:
      - /dev/vdb
    osd_scenario: collocated
    journal_size: 512
```

That's it ! Feel free to comment.

## Download my templates
As always you can download the templates of my example [here](https://blog.maumene.org/files/hci-osp12.tar.gz).
