# TP CLUSTERIZATION CEPH

Setup ceph cluster with kvm

## Basic setup

### Install prerequisites

```
sudo dnf install epel-release centos-release-ceph-reef.noarch -y
sudo dnf update -y
sudo dnf config-manager --set-enabled crb highavailability resilientstorage

sudo dnf install wget vim bash-completion chrony libvirt libvirt-daemon libvirt-daemon-driver-storage-rbd libvirt-daemon-driver-network libvirt-daemon-config-network qemu-kvm virt-install bridge-utils pcs pacemaker ceph-2:18.2.2-1.el9s.x86_64 -y
```

### Create Pool storage directory
```
sudo virsh pool-define-as guest_images dir - - - - "/usr/share/libvirt/iso"
sudo virsh pool-build guest_images
sudo virsh pool-start guest_images
sudo virsh pool-autostart guest_images
sudo virsh pool-list --all
```

### Create vm with libvirt

* 1st method
```
sudo virt-install \
	--virt-type kvm \
	--memory 1024 \
    --vcpus 1 \
    --cdrom /usr/share/libvirt/iso/debian-12.5.0-amd64-netinst.iso \
	--boot hd,cdrom,menu=on \
    --os-variant debian12 \
    --disk size=10 \
	--network bridge=br0 \
	--graphics vnc,listen=0.0.0.0 \
	--noautoconsole \
    --name demo-guest-1
```

* 2nd method
```
sudo virt-install \
	--virt-type kvm \
	--memory 1024 \
	--vcpus 1 \
	--cpu host \
	--location /usr/share/libvirt/iso/debian-12.5.0-amd64-netinst.iso \
	--boot hd,cdrom,menu=on \
	--os-variant debian12 \
	--disk size=20 \
	--network bridge=br0 \
	--graphics vnc,listen=0.0.0.0 \
	--extra-args 'console=ttyS0,115200n8 --- console=ttyS0,115200n8' \
	--noautoconsole
```

### List domain
```
sudo virsh list
```

### Create snapshot
```
sudo virsh snapshot-create-as \
	--domain demo-guest-1 \
	--description "Before apache2 installation" \
	--name "apache"
```

### Revert snapshot
```
sudo virsh snapshot-revert \
	--domain demo-guest-1 \
	--description "Before apache2 installation" \
	--name "apache"
```

### Delete snapshot
```
sudo virsh snapshot-delete \
	--domain demo-guest-1 \
	--current
```

### Delete vm
```
sudo virsh undefine demo-guest-1 --remove-all-storage
```

## Setup kvm Cluster sur all nodes
```
sudo dnf install pcs pacemaker fence-agents-all
sudo systemctl enable --now pcsd.service
sudo firewall-cmd --permanent --add-service=high-availability
sudo firewall-cmd --reload
```

### Change passwd
```
echo rocky | sudo passwd --stdin hacluster
```

### Node cluster
```
sudo pcs host auth p-kvm1 p-kvm2 p-kvm3
sudo pcs cluster setup my_cluster --start p-kvm1 p-kvm2 p-kvm3 --force
```

### show cluster status
```
sudo pcs cluster status 
```

&nbsp;
&nbsp;


## Setup ceph cluster

### Bootstrap ceph cluster on p-kvm1
```
sudo cephadm bootstrap --mon-ip 192.168.2.111
```

### Enable telemetry
```
sudo ceph telemetry on --license sharing-1-0
```

### Enbale channels
```
sudo ceph telemetry enable channel perf
```

### Add osd devices
```
# sudo ceph orch apply osd --all-available-devices
sudo ceph orch daemon add osd p-kvm1:/dev/nvme0n2
sudo ceph orch daemon add osd p-kvm1:/dev/nvme0n3
sudo ceph orch daemon add osd p-kvm1:/dev/nvme0n4
```

### [Allow root ssh login on p-kvm2 and p-kvm3](https://www.ibm.com/docs/en/db2/11.1?topic=installation-enable-disable-remote-root-login)


### Get public key
```
sudo cephadm shell -- ceph cephadm get-pub-key > ~/ceph.pub
```

### Add ssh key for create ssh directory and copy ssh key
```
ssh-keygen -t rsa
ssh-copy-id -f -i ~/ceph.pub root@p-kvm2
ssh-copy-id -f -i ~/ceph.pub root@p-kvm3
```

### Join nodes
```
sudo ceph orch host add p-kvm2 192.168.2.112
sudo ceph orch host add p-kvm3 192.168.2.113
```

### Add osd devices on p-kvm2
```
sudo ceph orch daemon add osd p-kvm2:/dev/nvme0n2
sudo ceph orch daemon add osd p-kvm2:/dev/nvme0n3
sudo ceph orch daemon add osd p-kvm2:/dev/nvme0n4
```

### Add osd devices on p-kvm3
```
sudo ceph orch daemon add osd p-kvm3:/dev/nvme0n2
sudo ceph orch daemon add osd p-kvm3:/dev/nvme0n3
sudo ceph orch daemon add osd p-kvm3:/dev/nvme0n4
```

### Add manager on nodes
```
sudo ceph orch daemon add mgr p-kvm1
sudo ceph orch apply mgr --placement "p-kvm1 p-kvm2 p-kvm3"
sudo firewall-cmd --permanent --add-port=8443/tcp
```

### Configure rbd (follow this https://blog.modest-destiny.com/posts/kvm-libvirt-add-ceph-rbd-pool/)

```
export CEPH_PGS="128"
export CEPH_USER="mylibvirt"
export CEPH_POOL="newrbd"
export CEPH_RADOS_HOST="192.168.2.111"
export CEPH_RADOS_PORT="6789"
export VIRT_SCRT_UUID="$(uuidgen)"
export VIRT_SCRT_PATH="/tmp/libvirt-secret.xml"
export VIRT_POOL_PATH="/tmp/libvirt-rbd-pool.xml"

sudo ceph osd pool create ${CEPH_POOL} ${CEPH_PGS} ${CEPH_PGS}
sudo ceph auth get-or-create "client.${CEPH_USER}" mon "profile rbd" osd "profile rbd pool=${CEPH_POOL}"

cat > "${VIRT_SCRT_PATH}" <<EOF
<secret ephemeral='no' private='no'>
  <uuid>${VIRT_SCRT_UUID}</uuid>
  <usage type='ceph'>
    <name>client.${CEPH_USER} secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file "${VIRT_SCRT_PATH}"

sudo virsh secret-set-value --secret "${VIRT_SCRT_UUID}" --base64 "$(sudo ceph auth get-key client.${CEPH_USER})"

cat > "${VIRT_POOL_PATH}" <<EOF
<pool type="rbd">
  <name>${CEPH_POOL}</name>
  <source>
    <name>${CEPH_POOL}</name>
    <host name='${CEPH_RADOS_HOST}' port='${CEPH_RADOS_PORT}' />
    <auth username='${CEPH_USER}' type='ceph'>
      <secret uuid='${VIRT_SCRT_UUID}'/>
    </auth>
  </source>
</pool>
EOF

sudo virsh pool-define "${VIRT_POOL_PATH}"
sudo rm -f "${VIRT_POOL_PATH}"
sudo virsh pool-autostart "${CEPH_POOL}"
sudo virsh pool-start "${CEPH_POOL}"

export CEPH_POOL="newrbd"
export VM_NAME="rbd-test"
export VM_DEV="vda"
export VM_SZ="8G"
export VM_IMAGE="/usr/share/libvirt/iso/debian-12.5.0-amd64-netinst.iso"
export VM_VOLUME="debian12-server"
export VM_CPU=1
export VM_MEM=1024

# Create a volume for the VM
# Legacy qemu-img command:
# qemu-img create -f rbd "rbd:${CEPH_POOL}/${VM_VOLUME}" "${VM_SZ}"
sudo virsh vol-create-as "${CEPH_POOL}" "${VM_VOLUME}" --capacity "${VM_SZ}" --format raw


sudo virt-install \
	 --virt-type kvm \
	 --name "${VM_NAME}"                       \
	 --vcpus="${VM_CPU}" --memory "${VM_MEM}"  \
	 --disk "vol=${CEPH_POOL}/${VM_VOLUME}"    \
	 --cdrom "${VM_IMAGE}"
   	 --boot hd,cdrom,menu=on \
	 --os-variant debian12 \
     --network bridge=br0 \
	 --graphics vnc,listen=0.0.0.0 \
	 --extra-args 'console=ttyS0,115200n8 --- console=ttyS0,115200n8' \
	 --noautoconsole

```

#### Check if vm is communicating with ceph
```
sudo virsh qemu-monitor-command --hmp rbd-test 'info block'
```

#### Check details 
```
sudo virsh domblklist rbd-test --details
```

### Create ceph-fs
```
sudo ceph osd pool create cephfs_data
sudo ceph osd pool create cephfs_metadata
sudo ceph fs new mycephfs cephfs_metadata cephfs_data
sudo ceph osd pool set cephfs.mycephfs.meta target_size_ratio .1
```

### Mount client

#### [Create client user](https://access.redhat.com/documentation/fr-fr/red_hat_ceph_storage/3/html/ceph_file_system_guide/deploying-ceph-file-systems)
```
sudo ceph auth get-or-create client.1 mon 'allow r' mds 'allow rw' osd 'allow rw pool=mycephfs'
```

### Export keyring to a file
```
sudo ceph auth get client.1 -o ceph.client.1.keyring
```

### Copy key to monitor
```
sudo cp ceph.client.1.keyring /etc/ceph/ceph.client.1.keyring
```

### Set appropriate rights
```
sudo chmod 644 /etc/ceph/ceph.client.1.keyring
```

### Create mount point
```
sudo mkdir -p /mnt/cephfs
```

### Mount
```
sudo mount -t ceph p-kvm1:6789,p-kvm2:6789,p-kvm3:6789:/ /mnt/cephfs -o name=1,secretfile=/etc/ceph/ceph.client.1.keyring
```

### Test mount
```
sudo stat /sbin/mount.ceph
```


### Live migration
```
sudo virsh migrate --live rbd-test
```


## Commands for demo

### Show domains
```
sudo virsh list
```

### Show cluster status
```
sudo ceph cluster -w
sudo ceph orch ps
```

### List storage pools
```
sudo virsh pool-list --all
```

### List osd pools
```
sudo ceph osd pool ls
```
