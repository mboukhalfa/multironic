# multironic
# Requirements:
CentOS9-20220330
4c / 16gb / 100gb

# Install libvirt and prepare nodes 
```
sudo dnf -y install qemu-kvm libvirt virt-install 
sudo systemctl enable --now libvirtd
```
Create a pool

`sudo mkdir /opt/mypool`
```XML
<pool type='dir'>
  <name>mypool</name>
  <target>
    <path>/opt/mypool</path>
  </target>
</pool>
```

```
sudo virsh pool-define mypool.xml
sudo virsh pool-start mypool
sudo virsh pool-autostart mypool
sudo virsh vol-create-as mypool node-1.qcow2 30G --format qcow2
sudo virsh vol-create-as mypool node-2.qcow2 30G --format qcow2
```
## Create provisioning network 
`provisioning-1.xml`
```XML
<network
	xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
	<dnsmasq:options>
		<!-- Risk reduction for CVE-2020-25684, CVE-2020-25685, and CVE-2020-25686. See: https://access.redhat.com/security/vulnerabilities/RHSB-2021-001 -->
		<dnsmasq:option value="cache-size=0"/>
	</dnsmasq:options>
	<name>provisioning-1</name>
	<bridge name='provisioning-1'/>
	<forward mode='bridge'></forward>
</network>
```
`provisioning-2.xml`
```XML
<network
	xmlns:dnsmasq='http://libvirt.org/schemas/network/dnsmasq/1.0'>
	<dnsmasq:options>
		<!-- Risk reduction for CVE-2020-25684, CVE-2020-25685, and CVE-2020-25686. See: https://access.redhat.com/security/vulnerabilities/RHSB-2021-001 -->
		<dnsmasq:option value="cache-size=0"/>
	</dnsmasq:options>
	<name>provisioning-2</name>
	<bridge name='provisioning-2'/>
	<forward mode='bridge'></forward>
</network>
```

```
sudo virsh net-define provisioning-1.xml
sudo virsh net-define provisioning-2.xml
sudo yum install net-tools

```
host interfaces :
```
ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 8900
        inet 10.201.10.192  netmask 255.255.255.0  broadcast 10.201.10.255
        inet6 fe80::f816:3eff:fece:85c5  prefixlen 64  scopeid 0x20<link>
        ether fa:16:3e:ce:85:c5  txqueuelen 1000  (Ethernet)
        RX packets 76010  bytes 249918715 (238.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 52618  bytes 7388741 (7.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 46  bytes 3744 (3.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 46  bytes 3744 (3.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:25:08:76  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```
```
sudo virsh net-start provisioning-1
sudo virsh net-autostart provisioning-1
sudo virsh net-start provisioning-2
sudo virsh net-autostart provisioning-2

```
## Define domains 

```XML
<domain type='kvm'>
	<name>node-1</name>
	<memory unit='MiB'>8000</memory>
	<vcpu>2</vcpu>
	<os>
		<type arch='x86_64' machine='q35'>hvm</type>
		<nvram>/var/lib/libvirt/qemu/nvram/node-1.fd</nvram>
		<boot dev='network'/>
		<bootmenu enable='no'/>
	</os>
	<features>
		<acpi/>
		<apic/>
		<pae/>
	</features>
	<cpu mode='host-passthrough'/>
	<clock offset='utc'/>
	<on_poweroff>destroy</on_poweroff>
	<on_reboot>restart</on_reboot>
	<on_crash>restart</on_crash>
	<devices>
		<disk type='volume' device='disk'>
			<driver name='qemu' type='qcow2' cache='unsafe'/>
			<source pool='mypool' volume='node-1.qcow2'/>
			<target dev='sda' bus='scsi'/>
		</disk>
		<controller type='scsi' model='virtio-scsi' />
		<interface type='bridge'>
			<mac address='00:5c:52:31:3a:9c'/>
			<source bridge='provisioning-1'/>
			<model type='virtio'/>
		</interface>
		<console type='pty'/>
		<input type='mouse' bus='ps2'/>
		<graphics type='vnc' port='-1' autoport='yes'/>
		<video>
			<model type='cirrus' vram='9216' heads='1'/>
		</video>
	</devices>
</domain>
```

```
sudo virsh define node-1.xml
```
Create the bridge
```
echo -e "DEVICE=provisioning-1\nTYPE=Bridge\nONBOOT=yes\nBOOTPROTO=static\nIPADDR=172.22.0.1\nNETMASK=255.255.255.0" | sudo dd of=/etc/sysconfig/network-scripts/ifcfg-provisioning-1
echo -e "DEVICE=provisioning-2\nTYPE=Bridge\nONBOOT=yes\nBOOTPROTO=static\nIPADDR=172.23.0.1\nNETMASK=255.255.255.0" | sudo dd of=/etc/sysconfig/network-scripts/ifcfg-provisioning-2
sudo systemctl restart NetworkManager.service
# check that the interface is up if down turn it up 
```
Install minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube config set driver kvm2
minikube config set memory 4096
usermod --append --groups libvirt `whoami`
minikube start  --insecure-registry  # 172.22.0.1:5000 / but I used the host ip 
minikube stop
sudo virsh attach-interface --domain minikube --model virtio --source provisioning-1 --type network --config
sudo virsh attach-interface --domain minikube --model virtio --source provisioning-2 --type network --config
```
Add container images to a local registry
```
sudo podman run -d -p 5000:5000 --name registry docker.io/library/registry:2.7.1
```
# have to check why we need this
```
sudo podman pod create infra-pod
sudo podman pod create ironic-pod
```
# TODO download ironic images later
```
sudo mkdir -p /opt/ironic/html/images
```
# Pull images
```
quay.io/metal3-io/vbmc
quay.io/metal3-io/sushy-tools
quay.io/metal3-io/ironic-ipa-downloader
quay.io/metal3-io/ironic:latest
quay.io/metal3-io/ironic-client
~~quay.io/metal3-io/mariadb~~
quay.io/metal3-io/keepalived
quay.io/metal3-io/baremetal-operator
```
# Tag images

```
sudo podman tag quay.io/metal3-io/vbmc 127.0.0.1:5000/localimages/vbmc
sudo podman tag quay.io/metal3-io/sushy-tools 127.0.0.1:5000/localimages/sushy-tools
sudo podman tag quay.io/metal3-io/ironic-ipa-downloader 127.0.0.1:5000/localimages/ironic-ipa-downloader
sudo podman tag quay.io/metal3-io/ironic-client 127.0.0.1:5000/localimages/ironic-client
sudo podman tag quay.io/metal3-io/keepalived 127.0.0.1:5000/localimages/keepalived
sudo podman tag quay.io/metal3-io/ironic:latest 127.0.0.1:5000/localimages/ironic:latest
sudo podman tag quay.io/metal3-io/baremetal-operator:latest 127.0.0.1:5000/localimages/baremetal-operator:latest
```
# Push images

```
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/keepalived </pre>
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic-client
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic:latest
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic-ipa-downloader
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/sushy-tools
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/vbmc
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/baremetal-operator:latest
```
# run httpd
```
sudo podman run -d --net host --name httpd-infra --pod infra-pod \
-v /opt/ironic:/shared --entrypoint /bin/runhttpd \
127.0.0.1:5000/localimages/ironic:latest
```
### Run vbmc and sushy-tools
#### Create VirtualBMC directories
```
sudo mkdir /opt/ironic/virtualbmc
sudo mkdir /opt/ironic/virtualbmc/vbmc
sudo mkdir /opt/ironic/virtualbmc/vbmc/conf
sudo mkdir /opt/ironic/virtualbmc/vbmc/log
sudo mkdir /opt/ironic/virtualbmc/sushy-tools

cat <<EOF >> /opt/ironic/virtualbmc/vbmc/virtualbmc.conf
[default]
config_dir=/root/.vbmc/conf/
[log]
logfile=/root/.vbmc/log/virtualbmc.log
debug=True
[ipmi]
session_timout=20
EOF

# Create nodes configuration
# create a dir for each node
/opt/ironic/virtualbmc/vbmc/conf/node-1
/opt/ironic/virtualbmc/vbmc/conf/node-2
# add config for each

cat <<EOF >> /opt/ironic/virtualbmc/vbmc/conf/node-1/config
[VirtualBMC]
username = admin
password = password
domain_name = node-1
libvirt_uri = qemu+ssh://root@172.22.0.1/system?&keyfile=/root/ssh/id_rsa_virt_power&no_verify=1&no_tty=1
address = 172.22.0.1
active = True
port =  6230

cat <<EOF >> /opt/ironic/virtualbmc/vbmc/conf/node-2/config
[VirtualBMC]
username = admin
password = password
domain_name = node-2
libvirt_uri = qemu+ssh://root@172.22.0.1/system?&keyfile=/root/ssh/id_rsa_virt_power&no_verify=1&no_tty=1
address = 172.23.0.1
active = True
port =  6231

cat <<EOF >> /opt/ironic/virtualbmc/sushy-tools/conf.py
SUSHY_EMULATOR_LIBVIRT_URI = "qemu+ssh://root@172.22.0.1/system?&keyfile=/root/ssh/id_rsa_virt_power&no_verify=1&no_tty=1"
SUSHY_EMULATOR_IGNORE_BOOT_DEVICE = False
SUSHY_EMULATOR_VMEDIA_VERIFY_SSL = False
SUSHY_EMULATOR_AUTH_FILE = "/root/sushy/htpasswd"

 sudo ssh-keygen -f /root/.ssh/id_rsa_virt_power -P ""
 sudo cat /root/.ssh/id_rsa_virt_power.pub | sudo tee -a /root/.ssh/authorized_keys
```
```
sudo podman run -d --net host --name vbmc --pod infra-pod \
     -v /opt/ironic/virtualbmc/vbmc:/root/.vbmc -v "/root/.ssh":/root/ssh \
     127.0.0.1:5000/localimages/vbmc

#shellcheck disable=SC2086
sudo podman run -d --net host --name sushy-tools --pod infra-pod \
     -v /opt/ironic/virtualbmc/sushy-tools:/root/sushy -v "/root/.ssh":/root/ssh \
     127.0.0.1:5000/localimages/sushy-tools
```
### Check containers are running
```
$sudo podman ps
CONTAINER ID  IMAGE                                     COMMAND               CREATED            STATUS                PORTS                   NAMES
b512380a1e48  docker.io/library/registry:2.7.1          /etc/docker/regis...  23 hours ago       Up 23 hours ago       0.0.0.0:5000->5000/tcp  registry
ae873fc0e77b  localhost/podman-pause:4.1.0-1653464416                         21 hours ago       Up About an hour ago                          452124918e59-infra
88b76fffc3a8  127.0.0.1:5000/localimages/ironic:latest                        About an hour ago  Up About an hour ago                          httpd-infra
a16d5f464c2b  127.0.0.1:5000/localimages/vbmc:latest    /bin/sh -c /usr/b...  22 seconds ago     Up 22 seconds ago                             vbmc
```
### Add vbmc client
Create vbmc.sh
```
#!/bin/bash
sudo podman exec -ti vbmc vbmc "$@"
```
```
sudo chmod a+x vbmc.sh
sudo ln -sf [absolutepath]/vbmc.sh /usr/local/bin/vbmc
```
# Check that two vbmcs are running for the two nodes
```
[centos@mohammed-mutlidocs ~]$ vbmc list
+-------------+---------+------------+------+
| Domain name | Status  | Address    | Port |
+-------------+---------+------------+------+
| node-1      | running | 172.22.0.1 | 6230 |
| node-2      | running | 172.22.0.1 | 6231 |
+-------------+---------+------------+------+

```
# Play with vbmc and ipmitools

```
sudo yum install OpenIPMI ipmitool
ipmitool -I lanplus -U admin -P password -H 172.22.0.1 -p 6230 power on
# Check the domain is running on virsh
sudo virsh list --all

```
# Run management cluster
```
minikube start
# test create a deployment with image from the local registry 
kubectl create deployment hello-node --image=172.22.0.1:5000/localimages/ironic-client
# check the image was pulled successfully
sudo minikube ssh 
sudo brctl addbr ironicendpoint
sudo ip link set ironicendpoint up
sudo brctl addif  ironicendpoint eth2
sudo ip addr add 172.22.0.2/24 dev ironicendpoint

sudo brctl addbr ironicendpoint2
sudo ip link set ironicendpoint2 up
sudo brctl addif  ironicendpoint2 eth3
sudo ip addr add 172.23.0.2/24 dev ironicendpoint2
```
# Launch ironic
### Clone BMO repo
```
git clone https://github.com/metal3-io/baremetal-operator.git
cd baremetal-operator/
```
### Ironic bmo configmap
why we call ironic bmo at this point we would just to focus on ironic
```
cat << EOF | sudo tee "/opt/ironic/ironic_bmo_configmap.env"
HTTP_PORT=6180
PROVISIONING_IP=172.22.0.2
PROVISIONING_CIDR=24
PROVISIONING_INTERFACE=ironicendpoint
DHCP_RANGE=172.22.0.10,172.22.0.100
DEPLOY_KERNEL_URL=http://172.22.0.2:6180/images/ironic-python-agent.kernel
DEPLOY_RAMDISK_URL=http://172.22.0.2:6180/images/ironic-python-agent.initramfs
IRONIC_ENDPOINT=http://172.22.0.2:6385/v1/
IRONIC_INSPECTOR_ENDPOINT=http://172.22.0.2:5050/v1/
CACHEURL=http://172.22.0.1/images
IRONIC_FAST_TRACK=true
RESTART_CONTAINER_CERTIFICATE_UPDATED="false"
LISTEN_ALL_INTERFACES="false"
INSPECTOR_REVERSE_PROXY_SETUP=true
IRONIC_KERNEL_PARAMS=console=ttyS0
EOF
```
```
# Copy the generated configmap for ironic deployment
cp "/opt/ironic/ironic_bmo_configmap.env"  "[BMOPATH]/ironic-deployment/keepalived/ironic_bmo_configmap.env"
# Install go
sudo yum install wget
wget https://go.dev/dl/go1.18.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.3.linux-amd64.tar.gz
# Add this line to /etc/bashrc
export PATH=$PATH:/usr/local/go/bin
# Then source bashrc
source /etc/bashrc
# Deploy ironic
export IRONIC_HOST_IP=172.22.0.2
```
Work around:
Currently bmo deploy ironic does not parameterize the name space thus we can edit the deploy.sh script so we generate yaml first then we edit name space and apply after.

The generated file before chaging namespace [ironic.yaml](ironic.yaml)

from the previous we change namespace,image,readiness, and delete mariadb since not used [ironic-ns.yaml](ironic-ns.yaml)

### apply ironic in two different ns:
```
kubectl apply -f ironic-1.yaml -n baremetal-operator-system-test1
kubectl apply -f ironic-2.yaml -n baremetal-operator-system-test2
```
## Create Ironic client
Use the script [ironicclient.sh](ironicclient.sh)

## Create bmo
#### generate yaml:
edit line : `${KUSTOMIZE} build "${BMO_SCENARIO}" > bmo.yaml #| kubectl apply ${KUBECTL_ARGS} -f -`
generate bmo yaml [bmo.yaml](bmo.yaml)
add watch namespace [bmo-wns.yaml](bmo-wns.yaml)
edit RBAC

### apply BMO in two different ns:
```
kubectl apply -f bmo-1.yaml -n baremetal-operator-system-test1
kubectl apply -f bmo-2.yaml -n baremetal-operator-system-test2
```
## Creating bmhs:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: node-0-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=

---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: node-0
spec:
  online: true
  bootMACAddress: 00:5c:52:31:3a:9c
  bootMode: legacy
  bmc:
    address: ipmi://172.22.0.1:6230
    credentialsName: node-0-bmc-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: node-1-bmc-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=

---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: node-1
spec:
  online: true
  bootMACAddress: 00:5c:52:31:3a:ad
  bootMode: legacy
  bmc:
    address: ipmi://172.23.0.1:6231
    credentialsName: node-1-bmc-secret

```
# run capm3
## get clusterctl
```
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.1.5/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl
```
## init the cluster
```
clusterctl init --core cluster-api:v1.1.5 --bootstrap kubeadm:v1.1.5 --control-plane kubeadm:v1.1.5 --infrastructure=metal3:v1.1.2  -v5
```
# Ref
Ironic troubleshooting: https://opendev.org/openstack/ironic/src/commit/e5a1997df840080d53e3bc2a12ac9169c3f96990/doc/source/admin/troubleshooting.rst
https://github.com/metal3-io/metal3-docs/blob/main/design/use-ironic.md
