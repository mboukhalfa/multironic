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
sudo minikube ssh sudo brctl addbr ironicendpoint
sudo brctl addbr ironicendpoint
sudo ip link set ironicendpoint up
sudo sudo brctl addif  ironicendpoint eth2
sudo ip addr add 172.22.0.2/24 dev ironicendpoint
```
