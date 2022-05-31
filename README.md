# multironic
# Requirements:
CentOS9-20220330
4c / 16gb / 100gb

# Install libvirt and prepare nodes 


```
sudo minikube ssh sudo brctl addbr ironicendpoint
sudo brctl addbr ironicendpoint
sudo ip link set ironicendpoint up
sudo sudo brctl addif  ironicendpoint eth2
sudo ip addr add 172.22.0.2/24 dev ironicendpoint
```
