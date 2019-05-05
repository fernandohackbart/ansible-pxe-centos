This project aims to create a PXE guest to install other servers (for now CentOS7 en CoreOS)

The PXE server will contain the following services:
* Named
* TFTP
* DHCP
* Nginx

## Variables
```bash
export PXE_STAGE=/opt/stage/pxe
mkdir -p ${PXE_STAGE}
cd ${PXE_STAGE}
export CENTOS_ISO=CentOS-7-x86_64-DVD-1810.iso
```
## Downloads
```bash
curl -s http://mirrors.up.pt/pub/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-DVD-1810.iso -O ${PXE_STAGE}/CentOS-7-x86_64-DVD-1810.iso
```

## Create a network in the host for the private use of the PXE process
```bash
cat > ${PXE_STAGE}/pxe-net.xml <<EOF
<network>
  <name>pxe-net</name>
  <bridge name="pxe-br0" />
  <forward/>
  <ip address="192.168.40.1" netmask="255.255.255.0" />
</network>
EOF
```

```bash
sudo virsh net-define ${PXE_STAGE}/pxe-net.xml
sudo virsh net-autostart pxe-net
sudo virsh net-start pxe-net
```

## Create  the PXE guest
```bash
mkdir -p ${PXE_STAGE}/guests/pxe/

virt-install \
 -n pxe \
 -v \
 --description="PXE machine" \
 --os-type=Linux \
 --os-variant=generic \
 --ram=512 \
 --vcpus=1 \
 --graphics=vnc \
 --disk path=${PXE_STAGE}/guests/pxe/pxe.img,bus=virtio,size=10 \
 --cdrom=${PXE_STAGE}/${CENTOS_ISO} \
 --network=bridge:pxe-br0,model=virtio,mac=52:54:00:e2:87:5a
```
 
## Install  Linux Centos7
When the guest boots will present a graphical interface, the PXE server is installed with the minimum packages

To start de installation vncviewer should be used:
```bash
vncviewer :0
```

* IP: 192.168.40.10
* Gateway: 192.168.40.1
* DNS: 192.168.40.1
* Hostname: pxe.demobox.local
* Root password: Welcome1
* Keyboard: <choose yours>
* Timezone: <choose yours>

## Get host IP to run the playbook against the guest
```bash
for mac in `virsh domiflist pxe |grep -o -E "([0-9a-f]{2}:){5}([0-9a-f]{2})"` ; do  ip n  |grep $mac  |grep -o -P "^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" ; done
```

## Copy my ssh key to the host (have to make this better)
```bash
ssh-keygen -R 192.168.40.10 -f ~/.ssh/known_hosts
ssh-keyscan -t rsa 192.168.40.10 >> ~/.ssh/known_hosts
ssh-copy-id root@192.168.40.10
```

## Stop the guest to release the console
```bash
ssh root@192.168.40.10 shutdown -h now
```

## Attach the CDROM to the guest, is will be used to server the installation files 
```bash
cat > ${PXE_STAGE}/pxe-cdrom.xml <<EOF
<disk type='block' device='cdrom'>
  <driver name='qemu' type='raw'/>
  <source dev='${PXE_STAGE}/${CENTOS_ISO}'/>
  <target dev='hda' bus='ide'/>
  <readonly/>
</disk>
EOF

virsh update-device pxe ${PXE_STAGE}/pxe-cdrom.xml

virsh domblklist pxe
```

## Start again the server (will run in background)
```bash
virsh start pxe
```

# Execute Ansible playbook to configure the server

## Adjust the hosts file to the environment you plan to use
The hosts_names is a list of guest to be installed, the mac addresses should match with the parameter ` --network network=pxe-net,model=virtio,mac=52:54:00:e2:87:5c` in the `virt-install`

```
    pxeservers:
      hosts:
        192.168.122.219:
        
        hosts_names: ["pxe-boot","pxe-master1","pxe-agent1"]
        hosts_ips: ["192.168.30.100","192.168.30.101","192.168.30.102"]
        hosts_mac_addrs: ["52:54:00:e2:87:5c","52:54:00:e2:87:5d","52:54:00:e2:87:5e"]
        hosts_roles: ["boot","master","agent"]
```

## Run the playbook from the base of the project clone
```bash
ansible-playbook -i conf/hosts pxe.yml
```

## Stop the guest to apply the changes in the enterprise Linux
```bash
ssh root@192.168.40.10 reboot
```

# Cleanup

## If need to clean up the pxe server

```bash
virsh destroy pxe;virsh undefine pxe;rm -rf ${PXE_STAGE}/guests/pxe/
```

## If need to remove the network
```bash
virsh net-destroy pxe-net
virsh net-undefine pxe-net
```

Now the guests for the other applications can be created

# Some other commands

* Insert a cdrom in the guest drive: `virsh change-media pxe hda --insert /opt/stage/CentOS-7-x86_64-DVD-1708.iso`
* Eject the cdrom: `virsh attach-disk pxe " " hda --type cdrom --mode readonly`
* Start a guest: `virsh start pxe`
* List the block devices of a guest: `virsh domblklist pxe`
* List the network devices of a guest: `virsh domiflist pxe-boot`


