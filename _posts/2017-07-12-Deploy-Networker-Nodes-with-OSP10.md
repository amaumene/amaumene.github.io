---
layout: post
title: Deploy Networker Nodes with OSP10
---

By default all the Neutron services are running on the controller nodes. This is great when the North-South traffic through floating IP isn't to heavy. When you can't use provider networks to scale it can become challenging for your controllers to cope with this North-South traffic. Thanks to Composable Roles, which was released with OSP10, we can now deploy arbitrary nodes and configure the services assigned. This documentation is trying to be a starting point if you are new to this.

<!-- TOC -->

- [Composable Roles](#composable-roles)
- [Network Interface Template definition](#network-interface-template-definition)
- [Network Interface Template configuration](#network-interface-template-configuration)
- [Neutron specific configuration](#neutron-specific-configuration)
- [Neutron LBaaSv2 configuration](#neutron-lbaasv2-configuration)
- [Download my templates](#download-my-templates)

<!-- /TOC -->

## Composable Roles

The first thing you need to do is to write a custom roles_network.yaml. On this file you will need to remove all the Neutron services form the Controller role (except Neutron API as we want to keep neutron-server running on the controllers).

The definition of the role will then look like:
```
- name: Controller
 CountDefault: 1
 ServicesDefault:
[...]
 - OS::TripleO::Services::MySQL
 - OS::TripleO::Services::NeutronApi
 - OS::TripleO::Services::NeutronCorePlugin
 - OS::TripleO::Services::RabbitMQ
[...]
```

We then need to create a basic new role for our Networkers than will look like:
```
- name: Networker
 ServicesDefault:
 - OS::TripleO::Services::CACerts
 - OS::TripleO::Services::NeutronDhcpAgent
 - OS::TripleO::Services::NeutronL3Agent
 - OS::TripleO::Services::NeutronMetadataAgent
 - OS::TripleO::Services::NeutronCorePlugin
 - OS::TripleO::Services::NeutronOvsAgent
 - OS::TripleO::Services::Kernel
 - OS::TripleO::Services::Ntp
 - OS::TripleO::Services::Snmp
 - OS::TripleO::Services::Sshd
 - OS::TripleO::Services::Timezone
 - OS::TripleO::Services::TripleoPackages
 - OS::TripleO::Services::TripleoFirewall
 - OS::TripleO::Services::SensuClient
 - OS::TripleO::Services::FluentdClient
 - OS::TripleO::Services::VipHosts
```

We want to deploy a baremetal node running:
* neutron-dhcp-agent
* neutron-l3-agent
* neutron-metada-agent
* neutron-ovs-agent

So those services have been taken out from the Controller role and defined on the new one called Networker. We also need to push the common services configuration like CACerts, Kernel, NTP, etc...

## Network Interface Template definition

The new Networker role will obviously need to have its NIC configured. To do this you will need to first call it like this (by default on network-environment.yaml file):
```
resource_registry:
 # Network Interface templates to use (these files must exist)
 OS::TripleO::Compute::Net::SoftwareConfig:
 ./nic-configs/compute.yaml
 OS::TripleO::Controller::Net::SoftwareConfig:
 ./nic-configs/controller.yaml
 OS::TripleO::CephStorage::Net::SoftwareConfig:
 ./nic-configs/ceph-storage.yaml
 OS::TripleO::Networker::Net::SoftwareConfig:
 ./nic-configs/networker.yaml
```

## Network Interface Template configuration

In my case the ./nic-configs/networker.yaml linked above looks like:
```
heat_template_version: 2015-04-30

description: >
 Software Config to drive os-net-config with 2 bonded nics on a bridge
 with VLANs attached for the networker role.

parameters:
 ControlPlaneIp:
 default: ''
 description: IP address/subnet on the ctlplane network
 type: string
 ExternalIpSubnet:
 default: ''
 description: IP address/subnet on the external network
 type: string
 InternalApiIpSubnet:
 default: ''
 description: IP address/subnet on the internal API network
 type: string
 StorageIpSubnet:
 default: ''
 description: IP address/subnet on the storage network
 type: string
 StorageMgmtIpSubnet:
 default: ''
 description: IP address/subnet on the storage mgmt network
 type: string
 TenantIpSubnet:
 default: ''
 description: IP address/subnet on the tenant network
 type: string
 ManagementIpSubnet: # Only populated when including environments/network-management.yaml
 default: ''
 description: IP address/subnet on the management network
 type: string
 BondInterfaceOvsOptions:
 default: 'bond_mode=active-backup'
 description: The ovs_options string for the bond interface. Set things like
 lacp=active and/or bond_mode=balance-slb using this option.
 type: string
 constraints:
 - allowed_pattern: "^((?!balance.tcp).)*$"
 description: |
 The balance-tcp bond mode is known to cause packet loss and
 should not be used in BondInterfaceOvsOptions.
 ExternalNetworkVlanID:
 default: 10
 description: Vlan ID for the external network traffic.
 type: number
 InternalApiNetworkVlanID:
 default: 20
 description: Vlan ID for the internal_api network traffic.
 type: number
 StorageNetworkVlanID:
 default: 30
 description: Vlan ID for the storage network traffic.
 type: number
 StorageMgmtNetworkVlanID:
 default: 40
 description: Vlan ID for the storage mgmt network traffic.
 type: number
 TenantNetworkVlanID:
 default: 50
 description: Vlan ID for the tenant network traffic.
 type: number
 ManagementNetworkVlanID:
 default: 60
 description: Vlan ID for the management network traffic.
 type: number
 ControlPlaneDefaultRoute: # Override this via parameter_defaults
 description: The default route of the control plane network.
 type: string
 ExternalInterfaceDefaultRoute:
 default: '10.0.0.1'
 description: default route for the external network
 type: string
 ManagementInterfaceDefaultRoute: # Commented out by default in this template
 default: unset
 description: The default route of the management network.
 type: string
 ControlPlaneSubnetCidr: # Override this via parameter_defaults
 default: '24'
 description: The subnet CIDR of the control plane network.
 type: string
 DnsServers: # Override this via parameter_defaults
 default: []
 description: A list of DNS servers (2 max for some implementations) that will be added to resolv.conf.
 type: comma_delimited_list
 EC2MetadataIp: # Override this via parameter_defaults
 description: The IP address of the EC2 metadata server.
 type: string

resources:
 OsNetConfigImpl:
 type: OS::Heat::StructuredConfig
 properties:
 group: os-apply-config
 config:
 os_net_config:
 network_config:
 -
 type: interface
 name: nic1
 use_dhcp: false
 addresses:
 -
 ip_netmask:
 list_join:
 - '/'
 - - {get_param: ControlPlaneIp}
 - {get_param: ControlPlaneSubnetCidr}
 routes:
 -
 ip_netmask: 169.254.169.254/32
 next_hop: {get_param: EC2MetadataIp}
 -
 type: ovs_bridge
 name: {get_input: bridge_name}
 dns_servers: {get_param: DnsServers}
 members:
 -
 type: ovs_bond
 name: bond1
 mtu: 9000
 ovs_options: {get_param: BondInterfaceOvsOptions}
 members:
 -
 type: interface
 name: nic3
 mtu: 9000
 primary: true
 -
 type: interface
 mtu: 9000
 name: nic6
 -
 type: vlan
 device: bond1
 mtu: 9000
 vlan_id: {get_param: InternalApiNetworkVlanID}
 addresses:
 -
 ip_netmask: {get_param: InternalApiIpSubnet}
 -
 type: vlan
 device: bond1
 vlan_id: {get_param: TenantNetworkVlanID}
 mtu: 9000
 addresses:
 -
 ip_netmask: {get_param: TenantIpSubnet}
 -
 type: ovs_bridge
 name: br-prod
 mtu: 9000
 dns_servers: {get_param: DnsServers}
 members:
 -
 type: ovs_bond
 name: bond2
 mtu: 9000
 ovs_options: {get_param: BondInterfaceOvsOptions}
 members:
 -
 type: interface
 name: nic2
 mtu: 9000
 primary: true
 -
 type: interface
 mtu: 9000
 name: nic4
 -
 type: vlan
 device: bond2
 mtu: 9000
 vlan_id: {get_param: ManagementNetworkVlanID}
 addresses:
 -
 ip_netmask: {get_param: ManagementIpSubnet}
 routes:
 -
 default: true
 next_hop: {get_param: ManagementInterfaceDefaultRoute}
 -
 type: ovs_bridge
 name: br-nonprod
 mtu: 9000
 dns_servers: {get_param: DnsServers}
 members:
 -
 type: interface
 name: nic5
 mtu: 9000
 primary: true

outputs:
 OS::stack_id:
 description: The OsNetConfigImpl resource.
 value: {get_resource: OsNetConfigImpl}
```

In this case you can see I'm configuring nic1 as the provisioning interface (ctlplane network), then a bond to run the default bridge for internal api and tenant networks, then a bond called "br-prod" that will be then mapped to a physical network called "prod" and at the end a bridge called "br-nonprod" running on a single interface (nic5) that will be mapped as a "non-prod" physical network. In this case 2 kinds of physical network running on different NICs will carry the traffic from the outside to the networkers (that will then forward the traffic to the compute through the Tenant network, which each tenant using its own VxLAN). For more on this see this [documentation](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/10/html/networking_guide/bridge-mappings). You can use a more simple template, just remember to configure at least ctlplane, internal api and tenant networks.

## Neutron specific configuration

As seen above, we are using 2 different physical network, that needs to be configured (most probably on the network-environment.yaml first):
```
 # Set to empty string to enable multiple external networks or VLANs
 NeutronExternalNetworkBridge: "''"
 # The tunnel type for the tenant network (vxlan or gre). Set to '' to disable tunneling.
 NeutronTunnelTypes: 'vxlan'
 NeutronBridgeMappings: 'datacentre:br-ex,prod:br-prod,nonprod:br-nonprod'
 NeutronNetworkVLANRanges: 'datacentre:1:4000,prod:1:4000,nonprod:1:4000'
```

As you can see I keep the default br-ex bridge mapped to a datacentre physical network (in case the customer would eventually use it, you'd never know) and then map prod and non production to their respective bridges. For the VLAN ranges I set the value arbitrarily (but I know the value won't be over 4000). And for the traffic between Networkers and Computes, it will be carry on a VxLAN tunnel for each tenant (on top of a Tenant VLAN ID). This is specific to this example and can be skipped if you use a more simple configuration with just the default bridge.

## Neutron LBaaSv2 configuration

The first piece of configuration is to deploy neutron-lbaasv2-agent on the Networker nodes. This part isn't handled by heat-templates but but straight by Puppet configuration:
```
NetworkerExtraConfig:
networker_classes:
- ::neutron::agents::lbaas
neutron::agents::lbaas::user_group: haproxy
neutron::agents::lbaas::manage_haproxy_package: false
```

The second will be to have Controllers configured (since they are still running the neutron-server part) to offer LBaaSv2:
```
 ControllerExtraConfig:
 neutron::server::service_providers: ['LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default']
```

And the last part is to add LBaaSv2 service to the list of services provided by Neutron:
```
NeutronServicePlugins: "router,qos,trunk,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2"
```

That's it ! Feel free to comment

## Download my templates
As always you can download the templates of my example [here](https://blog.maumene.org/files/networker-osp10.tar.gz).
