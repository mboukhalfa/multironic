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
echo -e "DEVICE=provisioning\nTYPE=Bridge\nONBOOT=yes\nBOOTPROTO=static\nIPADDR=172.22.0.1\nNETMASK=255.255.255.1" | sudo dd of=/etc/sysconfig/network-scripts/ifcfg-provisioning-1
echo -e "DEVICE=provisioning\nTYPE=Bridge\nONBOOT=yes\nBOOTPROTO=static\nIPADDR=172.23.0.1\nNETMASK=255.255.255.1" | sudo dd of=/etc/sysconfig/network-scripts/ifcfg-provisioning-2
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
sudo virsh attach-interface --domain minikube --model virtio --source provisioning-1 --type network --config
sudo virsh attach-interface --domain minikube --model virtio --source provisioning-2 --type network --config
minikube start
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
```
# Tag images

```
sudo podman tag quay.io/metal3-io/vbmc 127.0.0.1:5000/localimages/vbmc
sudo podman tag quay.io/metal3-io/sushy-tools 127.0.0.1:5000/localimages/sushy-tools
sudo podman tag quay.io/metal3-io/ironic-ipa-downloader 127.0.0.1:5000/localimages/ironic-ipa-downloader
sudo podman tag quay.io/metal3-io/ironic-client 127.0.0.1:5000/localimages/ironic-client
sudo podman tag quay.io/metal3-io/keepalived 127.0.0.1:5000/localimages/keepalived
sudo podman tag quay.io/metal3-io/ironic:latest 127.0.0.1:5000/localimages/ironic:latest
```
# Push images

```
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/keepalived </pre>
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic-client
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic:latest
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/ironic-ipa-downloader
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/sushy-tools
sudo podman  push --tls-verify=false 127.0.0.1:5000/localimages/vbmc
```
# run httpd
```
sudo podman run -d --net host --name httpd-infra --pod infra-pod \
-v /opt/ironic:/shared --entrypoint /bin/runhttpd \
127.0.0.1:5000/localimages/ironic:latest
```
### Run vbmc and sushy-tools
```
sudo podman run -d --net host --name vbmc --pod infra-pod \
     -v /opt/ironic/virtualbmc/vbmc:/root/.vbmc -v "/root/.ssh":/root/ssh \
     127.0.0.1:5000/localimages/vbmc

#shellcheck disable=SC2086
sudo podman run -d --net host --name sushy-tools --pod infra-pod \
     -v /opt/ironic/virtualbmc/sushy-tools:/root/sushy -v "/root/.ssh":/root/ssh \
     127.0.0.1:5000/localimages/sushy-tools
```
```
sudo minikube ssh sudo brctl addbr ironicendpoint
sudo brctl addbr ironicendpoint
sudo ip link set ironicendpoint up
sudo sudo brctl addif  ironicendpoint eth2
sudo ip addr add 172.22.0.2/24 dev ironicendpoint
```

# Ref
Ironic troubleshooting: https://opendev.org/openstack/ironic/src/commit/e5a1997df840080d53e3bc2a12ac9169c3f96990/doc/source/admin/troubleshooting.rst
