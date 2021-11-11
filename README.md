### Prepare host
Install RHEL 8   
Create bridge, set IP

### Prepare service-vm
```angular2html
dnf install libguestfs-tools  virt-manager virt-install libguestfs-xfs

cd /var/lib/libvirt/images
qemu-img create -f qcow2 service-vm.qcow2 200G
virt-resize --expand /dev/sda3 /root/rhel-8.4-x86_64-kvm.qcow2 service-vm.qcow2 
virt-customize -a service-vm.qcow2  --run-command 'yum remove cloud-init* -y' --root-password password:redhat

virt-customize -a service-vm.qcow2 --run-command 'cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
ONBOOT="yes"
TYPE="Ethernet"
PEERDNS="yes"
IPV6INIT="no"
IPADDR=10.1.198.130
PREFIX=28
GATEWAY=10.1.198.142
DNS1=10.1.196.1
EOF'

virt-install --ram 32000 --vcpus 8 \
--os-variant rhel8.4 \
--disk path=/var/lib/libvirt/images/service-vm.qcow2,device=disk,bus=scsi,format=qcow2 \
--import --noautoconsole --vnc  \
--bridge  br-ex \
--name service-vm

ssh root@service-vm.pool5.roskosb.info 'hostnamectl set-hostname service-vm.pool5.roskosb.info'
```

### Prepare bastion
```
dnf install ansible python3-netaddr skopeo
git clone https://github.com/redhat-partner-solutions/crucible
cd crucible
rm -rf ~/.ansible/collections/  # optional, but recommended as there is no version information specificed in requirements.yml
ansible-galaxy collection install -r requirements.yml
ssh-copy-id root@infra.pool5.roskosb.info
ssh-copy-id root@service-vm.pool5.roskosb.info
```

### prepare service-vm
```
ssh root@service-vm.pool5.roskosb.info
- register service-vm to cdn or satellite
dnf install firewalld httpd podman -y
systemctl enable --now firewalld
```

### after running crucible playbooks
```
ssh root@infra.pool5.roskosb.info

qemu-img create -f qcow2 /var/lib/libvirt/images/crucible/data1.qcow2 100G
qemu-img create -f qcow2 /var/lib/libvirt/images/crucible/data2.qcow2 100G
qemu-img create -f qcow2 /var/lib/libvirt/images/crucible/data3.qcow2 100G
chown qemu.qemu /var/lib/libvirt/images/data*
virsh attach-disk super1 /var/lib/libvirt/images/crucible/data1.qcow2 --target vdb --persistent --subdriver qcow2
virsh attach-disk super2 /var/lib/libvirt/images/crucible/data2.qcow2 --target vdb --persistent --subdriver qcow2
virsh attach-disk super3 /var/lib/libvirt/images/crucible/data3.qcow2 --target vdb --persistent --subdriver qcow2
```
