# Cài đặt CEPH Rados Gateway

Install rados gateway

```sh
ceph-deploy install --rgw ceph-1 ceph-2 ceph-3
```

Deploy rados gateway

```sh
ceph-deploy create rgw ceph-1 ceph-2 ceph-3
```

Kiểm tra lại trạng thái của cluster

```sh
~# ceph -s
  cluster:
    id:     91bf5c0e-e038-40c8-be11-c053406e3879
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum ceph-1,ceph-2,ceph-3
    mgr: ceph-1(active), standbys: ceph-2, ceph-3
    osd: 6 osds: 6 up, 6 in
    rgw: 3 daemons active
 
  data:
    pools:   4 pools, 32 pgs
    objects: 189  objects, 2.2 KiB
    usage:   6.0 GiB used, 54 GiB / 60 GiB avail
    pgs:     32 active+clean
```

Ta thấy trong services có rgw, như vậy đã cài thành công.
