# Cài đặt CephFS

## Cài đặt metadata server

Sử dụng ceph-deploy để cài đặt cephfs

```sh
~# ceph-deploy mds create <node>
```

ví dụ: `ceph-deploy mds create ducpx-ceph-1 ducpx-ceph-2 ducpx-ceph-3`

Sau khi hoàn thành thêm mds server, kiểm tra trạng thái của các mds server.

```sh
~# ceph mds stat
, 3 up:standby

~# ceph fs status
+--------------+
| Standby MDS  |
+--------------+
| ducpx-ceph-1 |
| ducpx-ceph-2 |
| ducpx-ceph-3 |
+--------------+
MDS version: ceph version 13.2.0 (79a10589f1f80dfe21e8f9794365ed98143071c4) mimic (stable)
```

- Nếu kiểm tra không có kết quả tương tự, hãy reboot server.

## Cấu hình

Tạo pool

```sh
~# ceph osd pool create cephfs-data 128 128
~# ceph osd pool create cephfs-metadata 128 128
```

Tạo mới ceph fs

```sh
~# ceph fs new cephfs cephfs-metadata cephfs-data
new fs with metadata pool 11 and data pool 12
```

- 11 là id của pool cephfs-metadata, 12 là id của pool cephfs-data.

Set số lượng metadata active

```sh
~# ceph fs set max_mds cephfs 2
```

Trạng thái của cephfs

```sh
~# ceph fs status cephfs
cephfs - 0 clients
======
+------+--------+--------------+---------------+-------+-------+
| Rank | State  |     MDS      |    Activity   |  dns  |  inos |
+------+--------+--------------+---------------+-------+-------+
|  0   | active | ducpx-ceph-3 | Reqs:    0 /s |   10  |   13  |
|  1   | active | ducpx-ceph-2 | Reqs:    0 /s |   10  |   13  |
+------+--------+--------------+---------------+-------+-------+
+-----------------+----------+-------+-------+
|       Pool      |   type   |  used | avail |
+-----------------+----------+-------+-------+
| cephfs-metadata | metadata | 4926  | 42.8G |
|   cephfs-data   |   data   |    0  | 42.8G |
+-----------------+----------+-------+-------+
+--------------+
| Standby MDS  |
+--------------+
| ducpx-ceph-1 |
+--------------+
MDS version: ceph version 13.2.0 (79a10589f1f80dfe21e8f9794365ed98143071c4) mimic (stable)
```

Trạng thái của cluster

```sh
~# ceph -s
  cluster:
    id:     a10525ec-19d2-4210-970a-8e12f8bce0cd
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ducpx-ceph-3,ducpx-ceph-2,ducpx-ceph-1
    mgr: ducpx-ceph-1(active), standbys: ducpx-ceph-3, ducpx-ceph-2
    mds: cephfs-2/2/2 up  {0=ducpx-ceph-3=up:active,1=ducpx-ceph-2=up:active}, 1 up:standby
    osd: 6 osds: 6 up, 6 in

  data:
    pools:   7 pools, 352 pgs
    objects: 884  objects, 1.1 GiB
    usage:   7.8 GiB used, 136 GiB / 144 GiB avail
    pgs:     352 active+clean

  io:
    client:   2.0 KiB/s rd, 2 op/s rd, 0 op/s wr
```
