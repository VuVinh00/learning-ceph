# Tích hợp cephfs với manila openstack để cung cấp share file system.

## Thực hiện trên ceph cluster

Kiểm tra các pool cho cephfs

```sh
~# ceph fs ls
name: cephfs, metadata pool: cephfs-metadata, data pools: [cephfs-data ]
```

Pool cho metadata: cephfs-metadata.
Pool cho data: cephfs-data.

Tạo client.manila sử dụng cephfs

```sh
~# ceph auth add client.manila mds 'allow *' \
    mon 'allow r, allow command="auth del", allow command="auth caps", allow command="auth get", allow command="auth get-or-create"' \
    osd 'allow class-read object_prefix rbd_children, allow rwx pool=cephfs-data, allow rwx pool=cephfs-metadata' \
~# ceph auth get-or-create client.manila -o /etc/ceph/ceph.client.manila.keyring 
~# ls /etc/ceph/
ceph.client.admin.keyring   ceph.conf  tmpJGmsyU
ceph.client.manila.keyring  rbdmap
~# cat /etc/ceph/ceph.client.manila.keyring
[client.manila]
	key = AQBH/XBbDp8pNxAApjH/MiAnpaCTPuSgJXE6MQ==
~# ceph auth list | grep -A4 manila
installed auth entries:

client.manila
	key: AQBH/XBbDp8pNxAApjH/MiAnpaCTPuSgJXE6MQ==
	caps: [mds] allow *
	caps: [mon] allow r, allow command="auth del", allow command="auth caps", allow command="auth get", allow command="auth get-or-create"
	caps: [osd] allow class-read object_prefix rbd_children, allow rwx pool=cephfs-data, allow rwx pool=cephfs-metadata
```

Cấu hình client.manila trong file cấu hình `/etc/ceph/ceph.conf`

```sh
[client.manila]
client mount uid = 0
client mount gid = 0
log file = /var/log/manila/ceph-client.manila.log
keyring = /etc/ceph/manila.keyring
```

Enable CephFS snapshots.

```sh
~# ceph mds set allow_new_snaps true --yes-i-really-mean-it
```

## Cài đặt trên node manila-share

Thêm dòng cấu hình vào `/etc/hosts`

```sh
172.16.69.2 ceph-1
```

Cài đặt ceph-common

```sh
~# wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
~# echo deb https://download.ceph.com/debian-mimic/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
~# apt update
~# apt install -y ceph-common
:~# dpkg -l | egrep -i "ceph|rados|rbd"
ii  ceph-common                         13.2.1-1xenial                              amd64        common utilities to mount and interact with a ceph storage cluster
ii  libcephfs2                          13.2.1-1xenial                              amd64        Ceph distributed file system client library
ii  librados2                           13.2.1-1xenial                              amd64        RADOS distributed object store client library
ii  libradosstriper1                    13.2.1-1xenial                              amd64        RADOS striping interface
ii  librbd1                             13.2.1-1xenial                              amd64        RADOS block device client library
ii  librgw2                             13.2.1-1xenial                              amd64        RADOS Gateway client library
ii  libvirt-daemon-driver-storage-rbd   4.0.0-1ubuntu8.3~cloud0                     amd64        Virtualization daemon RBD storage driver
ii  python-cephfs                       13.2.1-1xenial                              amd64        Python 2 libraries for the Ceph libcephfs library
ii  python-rados                        13.2.1-1xenial                              amd64        Python 2 libraries for the Ceph librados library
ii  python-rbd                          13.2.1-1xenial                              amd64        Python 2 libraries for the Ceph librbd library
ii  python-rgw                          13.2.1-1xenial                              amd64        Python 2 libraries for the Ceph librgw library
```


Lấy file cấu hình của ceph và keyring của client.manila

```sh
~# scp root@ceph-1:/etc/ceph/ceph.conf /etc/ceph/
~# scp root@ceph-1:/etc/ceph/ceph.client.manila.keyring /etc/ceph/
```

Cấu hình manila-share.

Cài đặt thêm công cụ cấu hình crudini

```sh
apt install -y crudini
```

Cách sử dụng crudini

```sh
# Thêm cấu hình
crudini --set config_file section [param] [value]
    ví du:  thêm dòng cấu hình debug = true trong section DEFAULT vào file /etc/manila/manila.conf ta dùng lệnh sau
            
            ~# crudini --set /etc/manila/manila.conf DEFAULT debut true

# Xóa dòng cấu hình
crudini --del config_file section [param]
    Ví dụ:  Xóa dòng cấu hình debug = true trong section DEFAULT vào file /etc/manila/manila.conf ta dùng lệnh sau

            ~# crudini --del /etc/manila/manila.conf DEFAULT debut
```

Thực hiện các lệnh sau để cấu hình manila với ceph

```sh
crudini --set /etc/manila/manila.conf DEFAULT enabled_share_protocols NFS,CIFS,CEPHFS
crudini --set /etc/manila/manila.conf DEFAULT enabled_share_backends lvm,generic,cephfs

crudini --set /etc/manila/manila.conf cephfs driver_handles_share_servers False
crudini --set /etc/manila/manila.conf cephfs share_backend_name cephfs
crudini --set /etc/manila/manila.conf cephfs share_driver manila.share.drivers.cephfs.cephfs_native.CephFSNativeDriver
crudini --set /etc/manila/manila.conf cephfs cephfs_conf_path /etc/ceph/ceph.conf
crudini --set /etc/manila/manila.conf cephfs cephfs_auth_id manila
crudini --set /etc/manila/manila.conf cephfs cephfs_cluster_name ceph
crudini --set /etc/manila/manila.conf cephfs cephfs_enable_snapshots True
```

Restart các manila-share

```sh
~# service manila-share restart

# Kiểm tra trên node controller
~# manila service-list
+----+------------------+--------------------+------+---------+-------+----------------------------+
| Id | Binary           | Host               | Zone | Status  | State | Updated_at                 |
+----+------------------+--------------------+------+---------+-------+----------------------------+
| 1  | manila-scheduler | controller         | nova | enabled | up    | 2018-08-13T04:32:48.000000 |
| 2  | manila-share     | controller@generic | nova | enabled | up    | 2018-08-13T04:32:47.000000 |
| 3  | manila-share     | controller@lvm     | nova | enabled | up    | 2018-08-13T04:32:49.000000 |
| 4  | manila-share     | controller@cephfs  | nova | enabled | up    | 2018-08-13T04:32:48.000000 |
+----+------------------+--------------------+------+---------+-------+----------------------------+
```

### Sử dụng back end cephfs

Tạo share-type

```sh
~# manila type-create cephfs false
+----------------------+--------------------------------------+
| Property             | Value                                |
+----------------------+--------------------------------------+
| required_extra_specs | driver_handles_share_servers : False |
| Name                 | cephfs                               |
| Visibility           | public                               |
| is_default           | -                                    |
| ID                   | 168bd540-3f15-4a54-b069-51b8ef674fcc |
| optional_extra_specs |                                      |
| Description          | None                                 |
+----------------------+--------------------------------------+
```

Chỉ định backend cho type-share

```sh
~# manila type-key cephfs set share_backend_name=cephfs
~# manila extra-specs-list
+--------------------------------------+--------------------+--------------------------------------+
| ID                                   | Name               | all_extra_specs                      |
+--------------------------------------+--------------------+--------------------------------------+
| 168bd540-3f15-4a54-b069-51b8ef674fcc | cephfs             | share_backend_name : cephfs          |
|                                      |                    | driver_handles_share_servers : False |
| 57ff97a1-5a17-4575-b68f-d9d19acfbdad | lvm                | driver_handles_share_servers : False |
| a237075b-0855-4dd1-af69-7af055abbbfb | default_share_type | driver_handles_share_servers : True  |
+--------------------------------------+--------------------+--------------------------------------+
~# manila type-list
+--------------------------------------+--------------------+------------+------------+--------------------------------------+-----------------------------+-------------+
| ID                                   | Name               | visibility | is_default | required_extra_specs                 | optional_extra_specs        | Description |
+--------------------------------------+--------------------+------------+------------+--------------------------------------+-----------------------------+-------------+
| 168bd540-3f15-4a54-b069-51b8ef674fcc | cephfs             | public     | -          | driver_handles_share_servers : False | share_backend_name : cephfs | None        |
| 57ff97a1-5a17-4575-b68f-d9d19acfbdad | lvm                | public     | -          | driver_handles_share_servers : False |                             | None        |
| a237075b-0855-4dd1-af69-7af055abbbfb | default_share_type | public     | YES        | driver_handles_share_servers : True  |                             | None        |
+--------------------------------------+--------------------+------------+------------+--------------------------------------+-----------------------------+-------------+
```

Tạo share mới

```sh
manila create nfs 1 --name share-cephfs-1 --share-type cephfs
``` 
