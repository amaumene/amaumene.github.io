---
layout: post
title: OSP10 to OSP13 Quick guide to perform fast forward upgrade
---

This document will guide you through a basic fast forward upgrade from OSP10 to OSP13. All the templates will be updated to match the best practices of OSP13.

<!-- TOC -->

- [OSP10 Overcloud](#osp10-overcloud)
    - [VirtualBMC](#virtualbmc)
    - [Templates](#templates)
- [Fast Forward Upgrade to 13](#fast-forward-upgrade-to-13)
    - [Undercloud](#undercloud)
        - [Upgrade from 10 to 11 to 12 to 13](#upgrade-from-10-to-11-to-12-to-13)
    - [Overcloud](#overcloud)
        - [Templates upgrade](#templates-upgrade)
            - [Ceph](#ceph)
        - [Local registry](#local-registry)
        - [Preparations](#preparations)
        - [Apply the preparations](#apply-the-preparations)
        - [Push the updates](#push-the-updates)

<!-- /TOC -->

## OSP10 Overcloud

I deployed a basic overcloud with the 20180503.1 OSP10 image, with 3 controllers, 2 computes and 3 ceph all on VMs. The undercloud is based on RHEL 7.5.
```
[stack@undercloud ~]$ openstack server list
+--------------------------------------+--------+--------+-------------------------+----------------+
| ID                                   | Name   | Status | Networks                | Image Name     |
+--------------------------------------+--------+--------+-------------------------+----------------+
| 18d42f88-9fa9-49b8-ac6e-7b2729656c33 | ctrl-0 | ACTIVE | ctlplane=192.168.100.10 | overcloud-full |
| 64090d10-a420-443f-90e0-2418815c9f8a | ctrl-2 | ACTIVE | ctlplane=192.168.100.14 | overcloud-full |
| 024cb1d3-2fe2-4a9f-b4fb-f5d8c2b54df9 | ctrl-1 | ACTIVE | ctlplane=192.168.100.6  | overcloud-full |
| 7a96390b-0488-4583-90b2-de3c6c9635e3 | ceph-2 | ACTIVE | ctlplane=192.168.100.19 | overcloud-full |
| cdd3c338-ac65-4c01-b2b5-2932ef6976f5 | cmpt-0 | ACTIVE | ctlplane=192.168.100.13 | overcloud-full |
| 3988826a-c39e-41a3-a76f-17a5dc9beb00 | ceph-0 | ACTIVE | ctlplane=192.168.100.15 | overcloud-full |
| f6f4e7fd-8387-4029-af6a-264c535d47cf | ceph-1 | ACTIVE | ctlplane=192.168.100.8  | overcloud-full |
| 335dfe60-39f3-4e36-9a78-0ebc49336ddf | cmpt-1 | ACTIVE | ctlplane=192.168.100.12 | overcloud-full |
+--------------------------------------+--------+--------+-------------------------+----------------+
```

### VirtualBMC

I installed virtualbmc on the hypervisor node to be able to use the Ironic IPMI driver (as the SSH driver is deprecated). With OSP10, Ironic is making a comparison between IP address but it doesn't compare the port. You then can't have multiple nodes  with the same IP address but different ports for IPMI, which would be what you would do with virtualbmc. The quickest work around is to configure on IP per VM on your hypervisor and run one virtualbmc per IP like this:
```
(vbmc) [root@hypervisor ~]# vbmc list
+-------------+---------+-----------------+------+
| Domain name |  Status |     Address     | Port |
+-------------+---------+-----------------+------+
|    ceph-0   | running | 192.168.100.205 | 6235 |
|    ceph-1   | running | 192.168.100.206 | 6236 |
|    ceph-2   | running | 192.168.100.207 | 6237 |
|    cmpt-0   | running | 192.168.100.203 | 6233 |
|    cmpt-1   | running | 192.168.100.204 | 6234 |
|    ctrl-0   | running | 192.168.100.200 | 6230 |
|    ctrl-1   | running | 192.168.100.201 | 6231 |
|    ctrl-2   | running | 192.168.100.202 | 6232 |
+-------------+---------+-----------------+------+
```

And then your instackenv.json will look like:
```
{
    "nodes":[
        {
            "mac":[
                "52:54:00:32:c0:87"
            ],
            "cpu":"4",
            "memory":"6144",
            "disk":"40",
            "arch":"x86_64",
            "pm_type":"pxe_ipmitool",
            "pm_user":"admin",
            "pm_password":"password",
            "pm_port":"6230",
            "name":"ctrl-0",
            "pm_addr":"192.168.100.200"
        },
        {
            "mac":[
                "52:54:00:8a:37:15"
            ],
            "cpu":"4",
            "memory":"6144",
            "disk":"40",
            "arch":"x86_64",
            "pm_type":"pxe_ipmitool",
            "pm_user":"admin",
            "pm_password":"password",
            "pm_port":"6231",
            "name":"ctrl-1",
            "pm_addr":"192.168.100.201"
        },
      [...]
    ]
}
```

### Templates

This is the command I used to deploy my OSP10:
```
[stack@undercloud ~]$ openstack overcloud deploy --templates /home/stack/my_templates \
  -e $(pwd)/network-isolation.yaml \
  -e $(pwd)/storage-environment-osp10.yaml \
  -e $(pwd)/network-environment.yaml \
  -e $(pwd)/ips-from-pool-all.yaml \
  --timeout 180
```

As you can see, I'm have the default templates to /home/stack/my_templates (to me it's a best practice to use a copy), basic network isolation (I just don't deploy a VIP on the Storage Management network as I'm only deploying Ceph and no Swift), a basic Ceph configuration with Journal/OSD on the same disk as the system, I'm using only 1 NIC with OVS bridge and VLAN isolation per networks. I've also configured a template to use predictable IPs for the nodes and for the VIP. Other than that everything is standard and I've used only the packages from the CDN.


## Fast Forward Upgrade to 13
### Undercloud

Like the previous and regular upgrades, you need to upgrade the undercloud first.

In case it hasn't been clearly communicated to you (and from my experience this is quite common...) the FFWD upgrade is NOT a direct upgrade from 10 to 13 ! You will have to go 10 -> 11 -> 12 -> 13 for both the undercloud AND the overcloud (as OpenStack upstream only supports N+1 upgrade). 

#### Upgrade from 10 to 11 to 12 to 13

First replace the OSP10 repositories with OSP11 repositories.

Update the director's packages:
```
[stack@undercloud ~]$ sudo yum update -y python-tripleoclient
```

Then upgrade the undercloud:
```
[stack@undercloud ~]$ openstack undercloud upgrade
```

Grab a coffee and wait for the success message:
```
#############################################################################
Undercloud upgrade complete.
The file containing this installation's passwords is at
/home/stack/undercloud-passwords.conf.
There is also a stackrc file at /home/stack/stackrc.
These files are needed to interact with the OpenStack services, and should be
secured.
#############################################################################
```

Reboot (not really needed but better to be safe than sorry):
```
[stack@undercloud ~]$ sudo reboot
```

Repeat the steps to upgrade from11 to 12 and then from 12 to 13. You should have an OSP13 undercloud ready at the end. This is quite straightforward so let's move on the interesting part.

### Overcloud
#### Templates upgrade

Between OSP10 and OSP13 the TripleO Heat Templates changed quite a bit with each releases. It is necessary to update them to 13 before.

##### Ceph

Since OSP12 Ceph is configured by Ansible and not by Puppet anymore. You will then need to install ceph-ansible:
```
(undercloud) [stack@undercloud deployment]$ sudo yum install ceph-ansible
```

Then you will need to update your OSP10 template:
```
(undercloud) [stack@undercloud deployment]$ diff storage-environment-osp10.yaml /usr/share/openstack-tripleo-heat-templates/environments/storage-environment.yaml
```

Here are the sections you have to pay attention to:
```
<   OS::TripleO::Services::CephMon: ../my_templates_osp10/puppet/services/ceph-mon.yaml
<   OS::TripleO::Services::CephOSD: ../my_templates_osp10/puppet/services/ceph-osd.yaml
<   OS::TripleO::Services::CephClient: ../my_templates_osp10/puppet/services/ceph-client.yaml
---
>   OS::TripleO::Services::CephMgr: ../docker/services/ceph-ansible/ceph-mgr.yaml
>   OS::TripleO::Services::CephMon: ../docker/services/ceph-ansible/ceph-mon.yaml
>   OS::TripleO::Services::CephOSD: ../docker/services/ceph-ansible/ceph-osd.yaml
>   OS::TripleO::Services::CephClient: ../docker/services/ceph-ansible/ceph-client.yaml
```
```
>   CephAnsiblePlaybook: ['/usr/share/ceph-ansible/site-docker.yml.sample']
```
```
<   ExtraConfig:
<     ceph::profile::params::osd_journal_size: 1000
<     ceph::profile::params::osds:
<         '/dev/vdc':
<           journal: '/dev/vdb'
---
>   ## OSDs configuration
>   ## See https://github.com/ceph/ceph-ansible/blob/stable-3.0/docs/source/osds/scenarios.rst
>   # CephAnsibleDisksConfig:
>   #   devices:
>   #   - /dev/vdb
>   #   osd_scenario: collocated
```

#### Local registry

To generate the yaml for the docker images, run this command:
```
(undercloud) [stack@undercloud deployment]$ openstack overcloud container image prepare \
  --namespace=registry.access.redhat.com/rhosp13 \
  --prefix=openstack- \
  --push-destination=192.168.24.1:8787 \
  -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
  --set ceph_namespace=registry.access.redhat.com/rhceph \
  --set ceph_image=rhceph-3-rhel7 \
  --tag-from-label {version}-{release} \
  --output-env-file=/home/stack/deployment/overcloud_images.yaml \
  --output-images-file /home/stack/deployment/local_registry_images.yaml
```

#### Preparations

My command used to deploy OSP10 was:
```
[stack@undercloud deployment]$ cat deploy.sh
source ~/stackrc
openstack overcloud deploy --templates /home/stack/my_templates_osp10 \
  -e $(pwd)/network-isolation-osp10.yaml \
  -e $(pwd)/storage-environment-osp10.yaml \
  -e $(pwd)/network-environment.yaml \
  -e $(pwd)/ips-from-pool-all-osp10.yaml \
  --timeout 180
```

Based on it and this [bug](https://bugzilla.redhat.com/show_bug.cgi?id=1558788) and the work around in it, I wrote the ffwd-prepare.sh script:
```
(undercloud) [stack@undercloud deployment]$ cat ffwd-prepare.sh
source ~/stackrc
openstack overcloud ffwd-upgrade prepare --templates /home/stack/my_templates_osp12 \
  -e $(pwd)/network-isolation-osp13.yaml \
  -e $(pwd)/storage-environment-osp13.yaml \
  -e $(pwd)/network-environment.yaml \
  -e $(pwd)/ips-from-pool-all-osp13.yaml \
  -e $(pwd)/ffwd-repos.yaml \
  --container-registry-file $(pwd)/overcloud_images.yaml \
  --timeout 180
```

You need to make a copy of the templates:
```
(undercloud) [stack@undercloud deployment]$ cp -r /usr/share/openstack-tripleo-heat-templates/ /home/stack/my_templates_osp13/
```

You need to have a correct OSP repositories configured for each upgrades, or use the CDN like this exemple:
```
(undercloud) [stack@undercloud deployment]$ cat ffwd-repos.yaml
parameter_defaults:
  FastForwardRepoType: custom-script
  FastForwardCustomRepoScriptContent: |
    set -e
    subscription-manager repos --disable=*
    rm -rf /etc/yum.repos.d/*
    case $1 in
      ocata)
        subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-11-rpms
        ;;
      pike)
        subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-12-rpms
        ;;
      queens)
        subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-13-rpms
        ;;
      *)
        echo "unknown release $1" >&2
        exit 1
    esac
```
Run the ffwd-prepare.sh script and wait for the success message:
```
2018-05-09 08:22:42Z [overcloud]: UPDATE_COMPLETE  Stack UPDATE completed successfully

 Stack overcloud UPDATE_COMPLETE 

Started Mistral Workflow tripleo.package_update.v1.get_config. Execution ID: 6e1c9009-009b-4f58-88ac-d5ad92e2dc01
Waiting for messages on queue 'tripleo' with no timeout.

Success
FFWD Upgrade Prepare on stack overcloud complete.
```

#### Apply the preparations

We are now ready to run the upgrade itself. The first time I ran it it failed quite quickly because of this [bug](https://bugzilla.redhat.com/show_bug.cgi?id=1558788), Ansible is actually trying to SSH to the overcloud using the "overcloud" username instead of the default "heat-admin".

**UPDATE from 15/05/2018:** This has been fixed, no need to patch anymore.

You need to apply this patch to fix it:
```
(undercloud) [stack@undercloud deployment]$ sudo diff -Naur /usr/lib/python2.7/site-packages/tripleoclient/v1/overcloud_ffwd_upgrade.py.orig /usr/lib/python2.7/site-packages/tripleoclient/v1/overcloud_ffwd_upgrade.py
--- /usr/lib/python2.7/site-packages/tripleoclient/v1/overcloud_ffwd_upgrade.py.orig 2018-05-09 04:45:48.122000000 -0400
+++ /usr/lib/python2.7/site-packages/tripleoclient/v1/overcloud_ffwd_upgrade.py 2018-05-09 04:47:12.917000000 -0400
@@ -141,10 +141,11 @@
         clients = self.app.client_manager
         :
+        ssh_user = parsed_args.ssh_user

         # Run ansible:
         inventory = oooutils.get_tripleo_ansible_inventory(
-            parsed_args.static_inventory, stack)
+            parsed_args.static_inventory, ssh_user, stack)
         # Don't expost limit_hosts. We need this on the whole overcloud.
         limit_hosts = ''
         oooutils.run_update_ansible_action(
```

Unfortunately I hit another bug that remove the content of /etc/os-net-config/config.json ! There is a bug opened for this one too. The work-around is in this [commit](https://review.openstack.org/#/c/566348/) but in a nutshell before running the ffwd-upgrade command remove this file on ALL your overcloud nodes:
```
sudo rm -f /usr/libexec/os-apply-config/templates/etc/os-net-config/config.json
```

With those 2 workarounds (that will probably fixed for the G.A), you're ready to start the ffwd-upgrade preparation by running this command:
```
(undercloud) [stack@undercloud ~]$ openstack overcloud ffwd-upgrade run
```

This will take a VERY long time so run it in a screen and wait for the success message:
```
 u'PLAY RECAP *********************************************************************',
 u'192.168.100.10             : ok=257  changed=135  unreachable=0    failed=0   ',
 u'192.168.100.12             : ok=82   changed=18   unreachable=0    failed=0   ',
 u'192.168.100.13             : ok=86   changed=20   unreachable=0    failed=0   ',
 u'192.168.100.14             : ok=195  changed=78   unreachable=0    failed=0   ',
 u'192.168.100.15             : ok=77   changed=14   unreachable=0    failed=0   ',
 u'192.168.100.19             : ok=73   changed=12   unreachable=0    failed=0   ',
 u'192.168.100.6              : ok=195  changed=78   unreachable=0    failed=0   ',
 u'192.168.100.8              : ok=73   changed=12   unreachable=0    failed=0   ',
 u'']
Success
```

#### Push the updates

We are now ready to apply the upgrades, let's start first with the controllers:
```
(undercloud) [stack@undercloud deployment]$ openstack overcloud upgrade run --roles Controller --skip-tags validation
```
Then push on the computes:
```
(undercloud) [stack@undercloud deployment]$ openstack overcloud upgrade run --roles Compute --skip-tags validation
```
Then push on the ceph:
```
(undercloud) [stack@undercloud deployment]$ openstack overcloud upgrade run --roles CephStorage --skip-tags validation
```
```
(undercloud) [stack@undercloud deployment]$ openstack overcloud ceph-upgrade run --templates \
[...]
 --ceph-ansible-playbook '/usr/share/ceph-ansible/infrastructure-playbooks/switch-from-non-containerized-to-containerized-ceph-daemons.yml,/usr/share/ceph-ansible/infrastructure-playbooks/rolling_update.yml'
```

And finally converge everything:
```
openstack overcloud ffwd-upgrade converge --templates \
[...]
```