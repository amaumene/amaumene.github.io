---
layout: post
title: OKD or OpenShift in one box with libvirt/KVM
---
This post is based on the [baremetal documentation](https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html). If you just want to have OKD/OpenShift up and running on your KVM host please refer to the official [documentation](https://github.com/openshift/installer/blob/master/docs/dev/libvirt/README.md). By doing it manually I wanted to learn a bit more about deploying OKD/OpenShift.

<!-- TOC -->

- [Hypervisor](#hypervisor)
- [Network](#network)
- [Virtual Machines](#virtual-machines)
- [DNS](#dns)
- [Load balancer](#load-balancer)
- [install-config.yaml](#install-config.yaml)
- [SELinux](#selinux)
- [Automated installation](#automated-installation)
- [Non automated part](#non-automated-part)
- [Fine-tuning for 1 master 1 worker setup](#fine-tuning-for-1-master-1-worker-setup)
- [Configure Image Registry](#configure-image-registry)
- [Pre-fly checks](#pre-fly-checks)
- [Access the console](#access-the-console)
- [SSL with Let's Encrypt](#ssl-with-lets-encrypt)
- [NFS for persistent storage](#nfs-for-persistent-storage)
- [Conclusion](#conclusion)

<!-- /TOC -->

## Hypervisor
For this example I assume you have a working KVM Hypervisor ready, in my case based on Fedora 32. I also created a folder /data where all my VMs will live.

On Fedora, CentOS/RHEL:
```
sudo yum install libvirt-devel libvirt-daemon-kvm libvirt-client libguestfs-tools-c virt-install
```
Then start libvirtd:
```
sudo systemctl enable --now libvirtd
```
and you should be good to go.

## Network
I will be using the default libvirt Network, but since I want the deployment to be eventually automated with a simple shell script, I edited it to match MAC address to hostnames (my VMs will be deployed with those MAC):
```
<network connections='2'>
  <name>default</name>
  <uuid>b5d30777-17a5-4303-8a23-94fab8b21782</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:9b:66:95'/>
  <dns>
    <host ip='192.168.122.20'>
      <hostname>bootstrap.okd.example.com</hostname>
    </host>
    <host ip='192.168.122.21'>
      <hostname>master.okd.example.com</hostname>
    </host>
    <host ip='192.168.122.22'>
      <hostname>worker.okd.example.com</hostname>
    </host>
  </dns>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='52:54:00:00:00:20' ip='192.168.122.20'/>
      <host mac='52:54:00:00:00:21' ip='192.168.122.21'/>
      <host mac='52:54:00:00:00:22' ip='192.168.122.22'/>
    </dhcp>
  </ip>
</network>
```
I've configured the libvirt zone in firewalld with those rules:
```
Services:             TCP:            UDP:
all-ports-openshift 	1025-65535      1025-65535
dns	                  53              53
ssh                   22
http	                80
https	                443
dhcp	                                67
dhcpv6-client	                        546
rpc-bind	            111             111 (NFS for persistent storage)
```

## Virtual Machines
In this example I will only deploy the bare minimum of: 1 master, 1 worker and 1 boostrap (that will be deleted after the master is setup). You need at least 4CPUs and 16 GB or RAM for each.
Thanks to libvirt, my VMs will be setup with those IPs (this is the content of /etc/hosts of the host):
```
192.168.122.20 bootstrap.okd.example.com bootstrap
192.168.122.21 master.okd.example.com master
192.168.122.22 worker.okd.example.com worker
192.168.122.1  api.okd.example.com api-int.okd.example.com
```

## DNS
For OKD/OpenShift to be able to deploy you need at least:
```
api.okd.example.com
api-int.okd.example.com
*.apps.okd.example.com
```
pointing to your load balancer. Since both APIs DNS are configured in my /etc/hosts, libvirt's dnsmasq will answer based on it. In my case I needed to have only the apps reachable from the outside (all oc commands will be run from the host). I've created a CNAME for *.apps.okd.example.com at my DNS provider to point to my host external IP.

## Load balancer
OKD/OpenShift won't deploy any load-balancers for you. In my case I installed HAproxy on the host server. This is my configuration:
```
global
    log         127.0.0.1 local2 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    timeout connect         5s
    timeout client          30s
    timeout server          30s
    log                     global

frontend kubernetes_api
    bind 0.0.0.0:6443
    default_backend kubernetes_api
    mode tcp
    option tcplog

backend kubernetes_api
    balance source
    mode tcp
    server bootstrap 192.168.122.20:6443 check
    server master 192.168.122.21:6443 check

frontend machine_config
    bind 0.0.0.0:22623
    default_backend machine_config
    mode tcp
    option tcplog

backend machine_config
    balance source
    mode tcp
    server bootstrap 192.168.122.20:22623 check
    server master 192.168.122.21:22623 check

frontend router_https
    bind 0.0.0.0:443
    default_backend router_https
    mode tcp
    option tcplog

backend router_https
    balance source
    mode tcp
    server worker-1 192.168.122.22:443 check

frontend router_http
    bind 0.0.0.0:80
    default_backend router_http
    mode tcp
    option tcplog

backend router_http
    balance source
    mode tcp
    server worker-1 192.168.122.22:80 check
```
The idea is to have Kubernetes APIs pointing to bootstrap server first, then when the bootstrap is done and the VM is stopped, they will point to the master. For the OKD/OpenShift console, the traffic will be passed to the worker node (as OKD/OpenShift is deployed as Kubernetes workload).

## install-config.yaml
```
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 1
controlPlane:
  hyperthreading: Enabled   
  name: master
  replicas: 1
metadata:
  name: okd
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.122.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths": ...}'
sshKey: 'ssh-ed25519 AAAA...'
```
Make sure to update pullSecret and sshKey with your own values.

## SELinux
I didn't disable SELinux, this is the configuration I used:
```
semanage import <<EOF
boolean -D
login -D
interface -D
user -D
port -D
node -D
fcontext -D
module -D
ibendport -D
ibpkey -D
permissive -D
boolean -m -1 virt_sandbox_use_all_caps
boolean -m -1 virt_use_nfs
port -a -t http_port_t -r 's0' -p tcp 6443
port -a -t http_port_t -r 's0' -p tcp 22623
fcontext -a -f a -t virt_image_t -r 's0' '/data'
fcontext -a -f a -t virt_image_t -r 's0' '/data(/.*)?'
fcontext -a -f a -t virt_image_t -r 's0' '/data/okd(/.*)?'
EOF
```
Copy-paste it to your shell as root or just disable SELinux.

## Automated installation
I wrote a script that will download and create the VM for you. Make sure to create the /data directory where all the VMs disks will live, I'm running this script from my home directory where my install-config.yaml is stored:
```
# Make sure all VMs are stopped
for i in $(virsh list --all | awk ' { print $2 }' | tail -n +3);
do
	virsh undefine $i
	virsh destroy $i
done
# Delete all existing datas
rm -rf /data/okd
rm -rf /data/*.qcow2
rm -rf /data/*.raw
rm -rf ~/.ssh/known_hosts
rm openshift-install-linux* openshift-client-linux* oc kubectl openshift-install

# Create okd directory of openshift-install files
mkdir /data/okd
# Copy the install-config.yaml
cp install-config.yaml /data/okd/

# Download openshift-install and openshift-client
wget $(curl https://api.github.com/repos/openshift/okd/releases/latest | grep openshift-install-linux | grep browser_download_url | cut -d\" -f4)
wget $(curl https://api.github.com/repos/openshift/okd/releases/latest | grep openshift-client-linux | grep browser_download_url | cut -d\" -f4)
tar xvzf openshift-install-linux*
tar xvzf openshift-client-linux*

# Create the ignition files
./openshift-install create ignition-configs --dir=/data/okd

# Download the latest Fedora CoreOS
coreos-installer download -p qemu -C /data -d -f qcow2.xz
chown -R qemu:qemu /data

j=0
for VM_NAME in bootstrap master worker;
do
	IGNITION_CONFIG="/data/okd/${VM_NAME}.ign"
	IMAGE="/data/${VM_NAME}.raw"
	VCPUS="8"
	RAM_MB="30000"
	SIZE="200G"
	MAC="52:54:00:00:00:2$j" #Generate the MAC addresses to match the libvirt network configuration

	qemu-img create "${IMAGE}" "${SIZE}" -f raw

	virt-resize --expand /dev/sda4 /data/fedora-coreos-*.qcow2 "${IMAGE}"
  # When I'm creating the worker, I'm stopping the bootstrap VM
	if [ "$VM_NAME" = "worker" ]; then
		virsh destroy bootstrap
	fi;

	virt-install --connect="qemu:///system" --name="${VM_NAME}" --vcpus="${VCPUS}" --memory="${RAM_MB}" \
		--os-variant="fedora-coreos-stable" --import --graphics="none" --network network=default,model=virtio,mac="${MAC}" \
		--disk="${IMAGE},cache=none" --cpu="host-passthrough" --noautoconsole \
		--qemu-commandline="-fw_cfg name=opt/com.coreos/config,file=${IGNITION_CONFIG}"
  # If I'm deploying the master it means the bootstrap is launched already and I can wait for the bootstrap process to end before I setup my worker.
	if [ "$VM_NAME" = "master" ]; then
		./openshift-install --dir=/data/okd wait-for bootstrap-complete
	fi;

	j=$((j+1))
done
```
As you can see I'm not using PXE boot to feed the ignition config to the VM, instead I'm using the --qemu-commandline="-fw_cfg [...]" argument of virt-install to pass it to the VM. This saved me the hassle of having another dnsmasq running on the host.

## Non automated part
Kubernetes should now be sort of working, we need to set the credentials to our environment:
```
export KUBECONFIG=/data/okd/auth/kubeconfig
```

You will still need to approve CSR manually, so monitor it through "./oc get csr":
```
NAME        AGE    SIGNERNAME                                    REQUESTOR                                                                   CONDITION
csr-2hnrw   33m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-8zxr6   43m    kubernetes.io/kubelet-serving                 system:node:master.okd.example.com                                              Approved,Issued
csr-blcbr   3m6s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-cc2h9   43m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-mxj6s   18m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
```

I wrote this script to automatically approve the request:
```
for csr in $(./oc get csr 2> /dev/null | grep -w 'Pending' | awk '{print $1}'); do
  echo -n '  --> Approving CSR: ';
  ./oc adm certificate approve "$csr" 2> /dev/null || true
  output_delay=0
done
 ```
You need to approve one request for the master and one request a bit later for the worker.

## Fine-tuning for 1 master 1 worker setup
Since we are running only 1 master and 1 worker we should scale down some replicas to save some ressources:
```
./oc scale --replicas=1 ingresscontroller/default -n openshift-ingress-operator
./oc scale --replicas=1 deployment.apps/console -n openshift-console
./oc scale --replicas=1 deployment.apps/downloads -n openshift-console
./oc scale --replicas=1 deployment.apps/oauth-openshift -n openshift-authentication
./oc scale --replicas=1 deployment.apps/packageserver -n openshift-operator-lifecycle-manager

./oc scale --replicas=1 deployment.apps/prometheus-adapter -n openshift-monitoring
./oc scale --replicas=1 deployment.apps/thanos-querier -n openshift-monitoring
./oc scale --replicas=1 statefulset.apps/prometheus-k8s -n openshift-monitoring
./oc scale --replicas=1 statefulset.apps/alertmanager-main -n openshift-monitoring
```

We also need to patch the etc quorum to scale it down to 1 replica. Create the file etcd_quorum.yaml with this content:
```
- op: add
  path: /spec/overrides
  value:
  - kind: Deployment
    group: apps/v1
    name: etcd-quorum-guard
    namespace: openshift-machine-config-operator
    unmanaged: true
```
Then apply it and scale it down to 1:
```
./oc patch clusterversion version --type json -p "$(cat etcd_quorum.yaml)"
./oc scale --replicas=1 deployment/etcd-quorum-guard -n openshift-machine-config-operator
```
Finally we need to patch the ingresscontroller to the only worker we deployed:
```
./oc label node worker.okd.example.com node-role.kubernetes.io/infra="true"
./oc patch -n openshift-ingress-operator ingresscontroller/default --patch '{"spec":{"replicas": 1,"nodePlacement":{"nodeSelector":{"matchLabels":{"node-role.kubernetes.io/infra":"true"}}}}}' --type=merge
```

## Configure Image Registry
Patch the imageregistry to set the default route to true.
```
./oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

## Pre-fly checks
Execute the command "./oc get clusteroperators" and make sure your output look like this:
```
NAME                                       VERSION                         AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
cloud-credential                           4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
cluster-autoscaler                         4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
config-operator                            4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
console                                    4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
csi-snapshot-controller                    4.5.0-0.okd-2020-10-15-235428   True        False         False      26h
dns                                        4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
etcd                                       4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
image-registry                             4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
ingress                                    4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
insights                                   4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
kube-apiserver                             4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
kube-controller-manager                    4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
kube-scheduler                             4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
kube-storage-version-migrator              4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
machine-api                                4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
machine-approver                           4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
machine-config                             4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
marketplace                                4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
monitoring                                 4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
network                                    4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
node-tuning                                4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
openshift-apiserver                        4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
openshift-controller-manager               4.5.0-0.okd-2020-10-15-235428   True        False         False      26h
openshift-samples                          4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
operator-lifecycle-manager                 4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
operator-lifecycle-manager-catalog         4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
operator-lifecycle-manager-packageserver   4.5.0-0.okd-2020-10-15-235428   True        False         False      31h
service-ca                                 4.5.0-0.okd-2020-10-15-235428   True        False         False      2d2h
storage                                    4.5.0-0.okd-2020-10-15-235428   True        False         False      2d1h
```

## Access the console
Execute the command "./oc get routes --all-namespaces |grep console-openshift" and get your console URL:
```
openshift-console          console             console-openshift-console.apps.okd.example.com                       console             https   reencrypt/Redirect     None
```
Your kubeadmin password is stored in "/data/okd/auth/kubeadmin-password".

# SSL with Let's Encrypt
I wanted to use this environment for demo purposes, and I didn't want to bother to have all my attendees adding the self signed certificate for each app (including the console). I created a wildcard certificate for *.apps.okd.example.com. Install certbot and run:
```
certbot certonly  -d *.apps.okd.example.com --manual
```
It will generate SSL certificate and keys in "/etc/letsencrypt/live/apps.okd.example.com".
Create a secret with fullchain.pem and privkey.pem:
```
./oc create secret tls router-certs --cert=/etc/letsencrypt/live/apps.okd.plop.im/fullchain.pem --key=/etc/letsencrypt/live/apps.okd.plop.im/privkey.pem -n openshift-ingress
```
And then patch the ingresscontroller to use it:
```
./oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}'
```

## NFS for persistent storage
Create the /data/nfs directory, then the /etc/exports file with this content:
```
/data/nfs 192.168.122.0/24(rw)
```
Start NFS server:
```
systemctl enable --now rpcbind nfs-server
```
Go to your console, and add persistent storage:
<img src="https://blog.maumene.org/files/okd-create-persistent-storage.png" width="600"/>

## Conclusion
You can now play with the lightest-and-automated OKD/OpenShift I could think of, this deployment was interesting for me to learn about this product more. With those scripts I can re-deploy a fresh install (and latest version) in less than 30 minutes in case I break it (that happened a lot actually...).
