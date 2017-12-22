I am creating the PXE server on a OpenSuSE tumbleweed desktop, so everything is Linux... Will use this to setup a DC/OS cluster with this...



The PXE server will contain the following services:
* Named
* TFTP
* DHCP
* Nginx


## Downloads
```
mkdir -p /opt/stage
cd /opt/stage
curl http://mirror.isec.pt/CentOS/7/isos/x86_64/CentOS-7-x86_64-DVD-1708.iso -O CentOS-7-x86_64-DVD-1708.iso
```

## Create a network in the host for the private use of the PXE process
```
cat > /tmp/dcos-net.xml <<EOF
<network>
  <name>dcos-net</name>
  <bridge name="dcos-br0" />
  <forward/>
  <ip address="192.168.40.1" netmask="255.255.255.0" />
</network>
EOF
```

```
virsh net-define /tmp/dcos-net.xml
virsh net-autostart dcos-net
virsh net-start dcos-net
```


## Create  the PXE guest
```
mkdir -p /opt/dcos/guests/dcos-pxe/

virt-install \
 -n dcos-pxe \
 -v \
 --description="DCOS PXE machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=512 \
 --vcpus=1 \
 --graphics=vnc \
 --disk path=/opt/dcos/guests/dcos-pxe/dcos-pxe.img,bus=virtio,size=10 \
 --cdrom=/opt/stage/CentOS-7-x86_64-DVD-1708.iso \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:5b
```
 
## Install  Linux Centos7
When the guest boots will present a graphical interface, the PXE server is installed with the minimum packages

* IP: 192.168.40.10
* Gateway: 192.168.40.1
* DNS: 192.168.40.1
* Hostname: dcos-pxe.prototype.local
* Root password: Welcome1
* Keyboard: <choose yours>
* Timezone: <choose yours>

## Get host IP to run the playbook against the guest
```
for mac in `virsh domiflist dcos-pxe |grep -o -E "([0-9a-f]{2}:){5}([0-9a-f]{2})"` ; do  ip n  |grep $mac  |grep -o -P "^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" ; done
```

## Copy my ssh key to the host (have to make this better)
```
ssh-copy-id root@192.168.40.10
```

## Stop the guest to release the console
```
ssh root@192.168.40.10 shutdown -h now
```

## Attach the CDROM to the guest, is will be used to server the installation files 
```
cat > /tmp/dcos-pxe-cdrom.xml <<EOF
<disk type='block' device='cdrom'>
  <driver name='qemu' type='raw'/>
  <source dev='/opt/stage/CentOS-7-x86_64-DVD-1708.iso'/>
  <target dev='hda' bus='ide'/>
  <readonly/>
</disk>
EOF

virsh update-device dcos-pxe /tmp/dcos-pxe-cdrom.xml
```

## Start again the server (will run in background)
```
virsh start dcos-pxe
```


## Clone the ansible playbook
```
git clone https://github.com/fernandohackbart/ansible-pxe-centos.git
```
## Adjust the hosts file to the environment you plan to use
The hosts_names is a list of guest to be installed, the mac addresses should match with the parameter ` --network network=dcos-pxe-net,model=virtio,mac=52:54:00:e2:87:5c` in the `virt-install`
```
    pxeservers:
      hosts:
        192.168.122.219:
        
        hosts_names: ["dcos-boot","dcos-master1","dcos-agent1"]
        hosts_ips: ["192.168.30.100","192.168.30.101","192.168.30.102"]
        hosts_mac_addrs: ["52:54:00:e2:87:5c","52:54:00:e2:87:5d","52:54:00:e2:87:5e"]
        hosts_roles: ["boot","master","agent"]
```

## Run the playbook from the base of the project clone
```
ansible-playbook -i hosts dcos-pxe.yml
```

## Stop the guest to apply the changes in the enterprise Linux
```
ssh root@192.168.40.10 reboot
```

## If need to clean up the dcos-pxe server!!!!
```
virsh destroy dcos-pxe;virsh undefine dcos-pxe;rm -rf /opt/dcos/guests/dcos-pxe/
```

## Now you can create other guests 
### Create a boot guest

```
virsh destroy dcos-boot;virsh undefine dcos-boot;rm -rf /opt/dcos/guests/dcos-boot/
```

```
mkdir -p /opt/dcos/guests/dcos-boot/

virt-install \
 -n dcos-boot \
 --description="DCOS BOOT machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1200 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-boot/dcos-boot.img,bus=virtio,size=10 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:5c
```
 
### Create a master guest
```
virsh destroy dcos-master1;virsh undefine dcos-master1;rm -rf /opt/dcos/guests/dcos-master1/
```

```
mkdir -p /opt/dcos/guests/dcos-master1/

virt-install \
 -n dcos-master1 \
 --description="DCOS Master 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=7000 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-master1/dcos-master1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:5d
```
 
### Create a agent guest
```
virsh destroy dcos-agent1;virsh undefine dcos-agent1;rm -rf /opt/dcos/guests/dcos-agent1/
```
 
```
mkdir -p /opt/dcos/guests/dcos-agent1/

virt-install \
 -n dcos-agent1 \
 --description="DCOS Agent 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=7000 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-agent1/dcos-agent1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:5e
 ```

### Create a NFS guest
```
virsh destroy dcos-nfs1;virsh undefine dcos-nfs1;rm -rf /opt/dcos/guests/dcos-nfs1/
```
 
```
mkdir -p /opt/dcos/guests/dcos-nfs1/

virt-install \
 -n dcos-nfs1 \
 --description="DCOS NFS 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-nfs1/dcos-nfs1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:5f
 ```

### Create a Gluster server guest
```
virsh destroy dcos-gluster1;virsh undefine dcos-gluster1;rm -rf /opt/dcos/guests/dcos-gluster1
virsh destroy dcos-gluster2;virsh undefine dcos-gluster2;rm -rf /opt/dcos/guests/dcos-gluster2
```
 
```
mkdir -p /opt/dcos/guests/dcos-gluster1
mkdir -p /opt/dcos/guests/dcos-gluster2

virt-install \
 -n dcos-gluster1 \
 --description="DCOS Gluster 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-gluster1/dcos-gluster1.img,bus=virtio,size=7 \
 --disk path=/opt/dcos/guests/dcos-gluster1/dcos-gluster1-data1.img,bus=virtio,size=10 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:62


virt-install \
 -n dcos-gluster2 \
 --description="DCOS Gluster 2 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/dcos-gluster2/dcos-gluster2.img,bus=virtio,size=7 \
 --disk path=/opt/dcos/guests/dcos-gluster2/dcos-gluster2-data1.img,bus=virtio,size=10 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:63
 ```


### Create a Kubernetes master guest
```
virsh destroy k8s-master1;virsh undefine k8s-master1;rm -rf /opt/dcos/guests/k8s-master1/
```
 
```
mkdir -p /opt/dcos/guests/k8s-master1

virt-install \
 -n k8s-master1 \
 --description="DCOS Kubernetes master 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=1500 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/k8s-master1/k8s-master1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:60
 ``` 

### Create a Kubernetes node guest
```
virsh destroy k8s-node1;virsh undefine k8s-node1;rm -rf /opt/dcos/guests/k8s-node1/
```
 
```
mkdir -p /opt/dcos/guests/k8s-node1

virt-install \
 -n k8s-node1 \
 --description="DCOS Kubernetes node 1 machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=6000 \
 --vcpus=1 \
 --graphics=vnc \
 --noautoconsole \
 --disk path=/opt/dcos/guests/k8s-node1/k8s-node1.img,bus=virtio,size=7 \
 --pxe \
 --network=bridge:dcos-br0,model=virtio,mac=52:54:00:e2:87:61
 ``` 

 
## Some other commands

* Insert a cdrom in the guest drive: `virsh change-media dcos-pxe hda --insert /opt/stage/CentOS-7-x86_64-DVD-1708.iso`
* Eject the cdrom: `virsh attach-disk dcos-pxe " " hda --type cdrom --mode readonly`
* Start a guest: `virsh start dcos-pxe`
* List the block devices of a guest: `virsh domblklist dcos-pxe`
* List the network devices of a guest: `virsh domiflist dcos-boot`
* Get the IP of a guest
```
for mac in `virsh domiflist dcos-pxe |grep -o -E "([0-9a-f]{2}:){5}([0-9a-f]{2})"` ; do  ip n  |grep $mac  |grep -o -P "^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" ; done
```

