# Hướng dẫn tích hợp ceph với cinder - ứng dụng tính năng rbd mirroring của ceph và cinder replication

Trước jhi thực hiện theo hướng dẫn, bạn phải biết cách tích hợp ceph với cinder, tham khảo [tại đây](https://github.com/pxduc96/ghichep-CEPH/blob/master/docs/ceph_luminous_integrated_ops_pike.md)

và cấu hình rbd mirroring [tại đây](../rbd_mirroring.md)

Môi trường để thực hiện bao gồm như sau:

- Hệ thống openstack đã được tích hợp cinder với ceph
- Gồm 2 cụm ceph.

## Cấu hình ở 2 cluster ceph

rbd-mirror daemon chạy trên cả 2 cluster. Để cài đặt

```
apt install ceph-mirror -y
```

Ở trên cả 2 cụm ceph yêu cầu đều có pool là `volumes` và user `cinder` được phép truy cập vào pool này.

```
# Trên ceph cluster
~# ceph osd pool ls --cluster ceph
images
volumes
vms

~# ceph auth list --cluster ceph
...
client.cinder
        key: AQDGKlJbK+eWChAAtPcoMhMLRlm69BxP5KvWUA==
        caps: [mon] allow r
        caps: [osd] allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=images

~# rbd mirror pool info volumes --cluster ceph
Mode: image
Peers:
  UUID                                 NAME   CLIENT
  37b75d90-8a78-4bef-b79f-d8b3c8b70630 backup client.cinder
```

```
# Trên backup cluster
~# ceph osd pool ls --cluster backup
images
volumes
vms

~# ceph auth list --cluster backup
...
client.cinder
        key: AQDtKlJbij/GHBAAqZX8nrvcrf7yVA0PcMYGyw==
        caps: [mon] allow r
        caps: [osd] allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rwx pool=images

~# rbd mirror pool info volumes --cluster backup
Mode: image
Peers:
  UUID                                 NAME CLIENT
  645f59c1-1f8d-4fb2-95bd-e0800a79c1ed ceph client.cinder
```

## Cấu hình cho cinder

Trong section [ceph_hdd] trong file cấu hình của cinder như sau:

```
[ceph_hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph_hdd
rbd_pool = volumes 
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = true
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rrados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = 6f31b1c4-4c12-4ebc-a43d-d91a81cfc0b4
report_discard_supported = true
replication_device = backend_id:backup, conf:/etc/ceph/backup.conf, user:cinder, pool:volumes
```

Trong thư mục `/etc/ceph` phải có các file key và cấu hình cho cinder:

```
~# ls /etc/ceph/
backup.client.cinder.keyring  backup.conf  ceph.client.cinder.keyring  ceph.client.glance.keyring  ceph.client.nova.keyring  ceph.conf  rbdmap
```

Restart các service của cinder

```
service cinder-volume restart
service cinder-scheduler restart
```

Kiểm tra service của cinder

```
~# openstack volume service list
+------------------+-----------------------------+------+---------+-------+----------------------------+
| Binary           | Host                        | Zone | Status  | State | Updated At                 |
+------------------+-----------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller                  | nova | enabled | up    | 2018-07-28T10:07:50.000000 |
| cinder-volume    | controller@lvm              | nova | enabled | up    | 2018-07-28T10:07:54.000000 |
| cinder-volume    | controller@ceph_hdd         | nova | enabled | up    | 2018-07-23T10:14:00.000000 |
+------------------+-----------------------------+------+---------+-------+----------------------------+
```

Tạo type cho cinder

```
~# cinder type-create replicated
~# cinder type-create ceph_hdd
~# cinder type-key ceph_hdd set volume_backend_name=ceph_hdd
~# cinder type-key replicated set volume_backend_name=ceph_hdd
~# cinder type-key replicated set replication_enabled='<is> True'

~# cinder type-list
+--------------------------------------+------------+-------------+-----------+
| ID                                   | Name       | Description | Is_Public |
+--------------------------------------+------------+-------------+-----------+
| 2abcfa31-d336-4278-8289-fa695092e944 | lvm        | -           | True      |
| ba462b1e-9a95-46c6-9047-7b92f555819a | replicated | -           | True      |
| ef9ba46c-98da-45a2-9903-d5048d84ebf4 | ceph_hdd   | -           | True      |
+--------------------------------------+------------+-------------+-----------+

~# cinder type-show replicated
+---------------------------------+--------------------------------------+
| Property                        | Value                                |
+---------------------------------+--------------------------------------+
| description                     | None                                 |
| extra_specs                     | replication_enabled : <is> True      |
|                                 | volume_backend_name : ceph_hdd       |
| id                              | ba462b1e-9a95-46c6-9047-7b92f555819a |
| is_public                       | True                                 |
| name                            | replicated                           |
| os-volume-type-access:is_public | True                                 |
| qos_specs_id                    | None                                 |
+---------------------------------+--------------------------------------+

~# cinder type-show ceph_hdd  
+---------------------------------+--------------------------------------+
| Property                        | Value                                |
+---------------------------------+--------------------------------------+
| description                     | None                                 |
| extra_specs                     | volume_backend_name : ceph_hdd       |
| id                              | ef9ba46c-98da-45a2-9903-d5048d84ebf4 |
| is_public                       | True                                 |
| name                            | ceph_hdd                             |
| os-volume-type-access:is_public | True                                 |
| qos_specs_id                    | None                                 |
+---------------------------------+--------------------------------------+
```

Thực hiện test

```
~# cinder create --volume-type ceph_hdd --name normal-ceph 1
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | false                                |
| consistencygroup_id            | None                                 |
| created_at                     | 2018-07-28T10:33:31.000000           |
| description                    | None                                 |
| encrypted                      | False                                |
| id                             | 331f80dd-cde1-421d-ac9f-587768af9b0b |
| metadata                       | {}                                   |
| migration_status               | None                                 |
| multiattach                    | False                                |
| name                           | normal-ceph                          |
| os-vol-host-attr:host          | None                                 |
| os-vol-mig-status-attr:migstat | None                                 |
| os-vol-mig-status-attr:name_id | None                                 |
| os-vol-tenant-attr:tenant_id   | 741f06395b4d4b9c8d852ca71012ed7c     |
| replication_status             | None                                 |
| size                           | 1                                    |
| snapshot_id                    | None                                 |
| source_volid                   | None                                 |
| status                         | creating                             |
| updated_at                     | 2018-07-28T10:33:31.000000           |
| user_id                        | edbe58627e0d49d5b4117ce21c310f49     |
| volume_type                    | ceph_hdd                             |
+--------------------------------+--------------------------------------+

~# openstack volume list
+--------------------------------------+------------------+-----------+------+-------------+
| ID                                   | Name             | Status    | Size | Attached to |
+--------------------------------------+------------------+-----------+------+-------------+
| 331f80dd-cde1-421d-ac9f-587768af9b0b | normal-ceph      | available |    1 |             |
+--------------------------------------+------------------+-----------+------+-------------+
```

Kiểm tra trên các cluster ceph

```
# ceph cluster
~# rbd -p volumes ls --cluster ceph
volume-331f80dd-cde1-421d-ac9f-587768af9b0b

~# rbd -p volumes ls --cluster backup

```

- Volume vừa tạo không được replication nên chỉ có ở trên cụm ceph cluster mà không có trên cụm backup cluster

Xóa volume này:

```
~# openstack volume delete normal-ceph

~# rbd -p volumes ls --cluster ceph

```

- volume đã được xóa bình thường

### Kiểm tra với volume được replication

Tạo volume

```
~# cinder create --volume-type replicated --name replicated-ceph 1
+--------------------------------+----------------------------------------------+
| Property                       | Value                                        |
+--------------------------------+----------------------------------------------+
| attachments                    | []                                           |
| availability_zone              | nova                                         |
| bootable                       | false                                        |
| consistencygroup_id            | None                                         |
| created_at                     | 2018-07-28T10:46:17.000000                   |
| description                    | None                                         |
| encrypted                      | False                                        |
| id                             | 4ea135c0-1bf2-4a54-8271-6a0fbc242fd3         |
| metadata                       | {}                                           |
| migration_status               | None                                         |
| multiattach                    | False                                        |
| name                           | replicated-ceph                              |
| os-vol-host-attr:host          | controller@ceph_replication#ceph_replication |
| os-vol-mig-status-attr:migstat | None                                         |
| os-vol-mig-status-attr:name_id | None                                         |
| os-vol-tenant-attr:tenant_id   | 741f06395b4d4b9c8d852ca71012ed7c             |
| replication_status             | None                                         |
| size                           | 1                                            |
| snapshot_id                    | None                                         |
| source_volid                   | None                                         |
| status                         | creating                                     |
| updated_at                     | 2018-07-28T10:46:17.000000                   |
| user_id                        | edbe58627e0d49d5b4117ce21c310f49             |
| volume_type                    | replicated                                   |
+--------------------------------+----------------------------------------------+
~# openstack volume list
+--------------------------------------+------------------+-----------+------+-------------+
| ID                                   | Name             | Status    | Size | Attached to |
+--------------------------------------+------------------+-----------+------+-------------+
| 4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 | replicated-ceph  | available |    1 |             |
+--------------------------------------+------------------+-----------+------+-------------+

~# openstack volume show replicated-ceph
+--------------------------------+----------------------------------------------+
| Field                          | Value                                        |
+--------------------------------+----------------------------------------------+
| attachments                    | []                                           |
| availability_zone              | nova                                         |
| bootable                       | false                                        |
| consistencygroup_id            | None                                         |
| created_at                     | 2018-07-28T10:46:17.000000                   |
| description                    | None                                         |
| encrypted                      | False                                        |
| id                             | 4ea135c0-1bf2-4a54-8271-6a0fbc242fd3         |
| migration_status               | None                                         |
| multiattach                    | False                                        |
| name                           | replicated-ceph                              |
| os-vol-host-attr:host          | controller@ceph_replication#ceph_replication |
| os-vol-mig-status-attr:migstat | None                                         |
| os-vol-mig-status-attr:name_id | None                                         |
| os-vol-tenant-attr:tenant_id   | 741f06395b4d4b9c8d852ca71012ed7c             |
| properties                     |                                              |
| replication_status             | enabled                                      |
| size                           | 1                                            |
| snapshot_id                    | None                                         |
| source_volid                   | None                                         |
| status                         | available                                    |
| type                           | replicated                                   |
| updated_at                     | 2018-07-28T10:46:17.000000                   |
| user_id                        | edbe58627e0d49d5b4117ce21c310f49             |
+--------------------------------+----------------------------------------------+
```

- Trạng thái replication_status được enabled

Kiểm tra trên 2 cụm ceph

```
~# rbd -p volumes ls --cluster ceph
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3
~# rbd mirror image status volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster ceph
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3:
  global_id:   42ce8755-d7ef-44a7-abba-11dd2ad7b142
  state:       up+stopped
  description: local image is primary
  last_update: 2018-07-28 17:52:52
~# rbd info volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster ceph
rbd image 'volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 36df4247fc4ae
        block_name_prefix: rbd_data.36df4247fc4ae
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, journaling
        op_features: 
        flags: 
        create_timestamp: Sat Jul 28 17:46:17 2018
        journal: 36df4247fc4ae
        mirroring state: enabled
        mirroring global id: 42ce8755-d7ef-44a7-abba-11dd2ad7b142
        mirroring primary: true

~# rbd -p volumes ls --cluster backup
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3
~# rbd mirror image status volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster backup
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3:
  global_id:   42ce8755-d7ef-44a7-abba-11dd2ad7b142
  state:       up+replaying
  description: replaying, master_position=[object_number=3, tag_tid=1, entry_tid=3], mirror_position=[object_number=3, tag_tid=1, entry_tid=3], entries_behind_master=0
  last_update: 2018-07-28 17:52:22
~# rbd info volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster backup
rbd image 'volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 10642ee75cda
        block_name_prefix: rbd_data.10642ee75cda
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, journaling
        op_features: 
        flags: 
        create_timestamp: Sat Jul 28 17:46:19 2018
        journal: 10642ee75cda
        mirroring state: enabled
        mirroring global id: 42ce8755-d7ef-44a7-abba-11dd2ad7b142
        mirroring primary: false
```

- volume vừa tạo đã được replicated sang cả 2 cluster.

### Kiểm tra hoạt động của cinder-volume khi xảy ra failover

Tạo volume mới không được replicated

```
~# cinder create --volume-type ceph_hdd --name normal-ceph 1

~# openstack volume list
+--------------------------------------+------------------+-----------+------+-------------+
| ID                                   | Name             | Status    | Size | Attached to |
+--------------------------------------+------------------+-----------+------+-------------+
| 41af27e8-8358-4343-b0a0-1b7446f95800 | normal-ceph      | available |    1 |             |
| 4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 | replicated-ceph  | available |    1 |             |
+--------------------------------------+------------------+-----------+------+-------------+

~# cinder service-list --binary cinder-volume
+---------------+-----------------------------+------+---------+-------+----------------------------+-----------------+
| Binary        | Host                        | Zone | Status  | State | Updated_at                 | Disabled Reason |
+---------------+-----------------------------+------+---------+-------+----------------------------+-----------------+
| cinder-volume | controller@ceph_hdd         | nova | enabled | up    | 2018-07-28T11:46:47.000000 | -               |
| cinder-volume | controller@lvm              | nova | enabled | up    | 2018-07-28T11:46:44.000000 | -               |
+---------------+-----------------------------+------+---------+-------+----------------------------+-----------------+

# Thực hiện đánh dấu cinder-volume failover


~# cinder failover-host controller@ceph_hdd
~# cinder service-list --binary cinder-volume
+---------------+-----------------------------+------+----------+-------+----------------------------+-----------------+
| Binary        | Host                        | Zone | Status   | State | Updated_at                 | Disabled Reason |
+---------------+-----------------------------+------+----------+-------+----------------------------+-----------------+
| cinder-volume | controller@ceph_hdd         | nova | disabled | up    | 2018-07-28T11:50:17.000000 | failed-over     |
| cinder-volume | controller@lvm              | nova | enabled  | up    | 2018-07-28T11:50:14.000000 | -               |
+---------------+-----------------------------+------+----------+-------+----------------------------+-----------------+

~# openstack volume list
+--------------------------------------+------------------+-----------+------+-------------+
| ID                                   | Name             | Status    | Size | Attached to |
+--------------------------------------+------------------+-----------+------+-------------+
| 41af27e8-8358-4343-b0a0-1b7446f95800 | normal-ceph      | error     |    1 |             |
| 4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 | replicated-ceph  | available |    1 |             |
+--------------------------------------+------------------+-----------+------+-------------+
```

- Kiểm tra volume trên các cụm ceph

```
~# rbd -p volumes ls --cluster ceph
volume-41af27e8-8358-4343-b0a0-1b7446f95800
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3
~# rbd info volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster ceph
rbd image 'volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 36df4247fc4ae
        block_name_prefix: rbd_data.36df4247fc4ae
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, journaling
        op_features: 
        flags: 
        create_timestamp: Sat Jul 28 17:46:17 2018
        journal: 36df4247fc4ae
        mirroring state: enabled
        mirroring global id: 42ce8755-d7ef-44a7-abba-11dd2ad7b142
        mirroring primary: false

~# rbd -p volumes ls --cluster backup
volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3
~# rbd info volumes/volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3 --cluster backup
rbd image 'volume-4ea135c0-1bf2-4a54-8271-6a0fbc242fd3':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 10642ee75cda
        block_name_prefix: rbd_data.10642ee75cda
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten, journaling
        op_features: 
        flags: 
        create_timestamp: Sat Jul 28 17:46:19 2018
        journal: 10642ee75cda
        mirroring state: enabled
        mirroring global id: 42ce8755-d7ef-44a7-abba-11dd2ad7b142
        mirroring primary: true
```

Ta thấy rằng, volume được replicated ở trên cluster backup đã chuyển từ `mirroring primary: false` sang `mirroring primary: true`

Chú ý: 

- Với volume được được replicated, trước khi thực hiện failover nếu được attach vào VM thì vẫn có khả năng được ghi dữ liệu bình thường.
- Sau khi thực hiện failover, nếu tạo mới một volume với type `replicated` thì volume đó sẽ có trạng thái `available` nhưng không thể attach vào vm (chưa rõ lý do).
- Cơ chế failback hiện nay chưa được implementation.
