### Create Ceph Pool:
```
# ceph osd pool create volumes 128
# ceph osd pool create vms 128
```

### install ceph-common pacakge
```
apt install ceph-common

or 

yum install ceph-common python-rbd
```

### Copy ceph.conf to openstack node
```
# scp /etc/ceph/ceph.conf osp:/etc/ceph
```

### setup the client auth in ceph cluster
```
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'

# ceph auth get-or-create client.cinder | ssh {your-volume-server} sudo tee /etc/ceph/ceph.client.cinder.keyring
# ssh {your-cinder-volume-server} chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

# ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} tee /etc/ceph/ceph.client.cinder.keyring

# ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key
```

### Run in nova-compute node
```
# uuidgen > uuid-secret.txt

# cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>`cat uuid-secret.txt`</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

# virsh secret-define --file secret.xml
# virsh secret-set-value --secret $(cat uuid-secret.txt) --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
```

### Configure Cinder
```
[DEFAULT]
enabled_backends = ceph
glance_api_version = 2
...

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_cluster_name = ceph
rbd_pool = volumes
rbd_user = cinder
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_secret_uuid = 4b5fd580-360c-4f8c-abb5-c83bb9a3f964
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
```

### Configure in Compute node 
Edit the file /etc/ceph/ceph.conf
```
# vim /etc/ceph/ceph.conf
[client]
rbd cache = true
rbd cache writethrough until flush = true
rbd concurrent management ops = 20
admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
log file = /var/log/ceph/qemu-guest-$pid.log
```

### make directroty woth permission qemu.libvirt
```
mkdir -p /var/run/ceph/guests/ /var/log/ceph/
chown qemu:libvirt /var/run/ceph/guests /var/log/ceph/
```

### Configure /etc/nova/nova.conf
```
[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 4b5fd580-360c-4f8c-abb5-c83bb9a3f964
disk_cachemodes="network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
hw_disk_discard = unmap
```

Reference Link: https://docs.ceph.com/en/latest/rbd/rbd-openstack/
https://superuser.openstack.org/articles/ceph-as-storage-for-openstack/
