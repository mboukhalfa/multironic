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

```
sudo minikube ssh sudo brctl addbr ironicendpoint
sudo brctl addbr ironicendpoint
sudo ip link set ironicendpoint up
sudo sudo brctl addif  ironicendpoint eth2
sudo ip addr add 172.22.0.2/24 dev ironicendpoint
```
