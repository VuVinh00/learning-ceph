# Placement groups

Kiểm tra PG khi tạo pool với 1 PGs, replicate = 3

```sh
~# ceph pg dump
dumped all
version 338
stamp 2018-09-10 03:21:14.681186
last_osdmap_epoch 0
last_pg_scan 0
PG_STAT OBJECTS MISSING_ON_PRIMARY DEGRADED MISPLACED UNFOUND BYTES LOG DISK_LOG STATE        STATE_STAMP                VERSION REPORTED UP      UP_PRIMARY ACTING  ACTING_PRIMARY LAST_SCRUB SCRUB_STAMP                LAST_DEEP_SCRUB DEEP_SCRUB_STAMP           SNAPTRIMQ_LEN 
1.0           0                  0        0         0       0     0   0        0 active+clean 2018-09-10 11:22:14.165425     0'0     27:9 [3,5,0]          3 [3,5,0]              3        0'0 2018-09-10 11:22:13.099742             0'0 2018-09-10 11:22:13.099742             0 
                  
1 0 0 0 0 0 0 0 0 
                    
sum 0 0 0 0 0 0 0 0 
OSD_STAT USED    AVAIL  TOTAL  HB_PEERS    PG_SUM PRIMARY_PG_SUM 
5        1.0 GiB 15 GiB 16 GiB [0,1,2,3,4]      1              0 
4        1.0 GiB 15 GiB 16 GiB [0,1,2,3,5]      0              0 
3        1.0 GiB 15 GiB 16 GiB [0,1,2,4,5]      1              1 
2        1.0 GiB 15 GiB 16 GiB [0,1,3,4,5]      0              0 
0        1.0 GiB 15 GiB 16 GiB [1,2,3,4,5]      1              0 
1        1.0 GiB 15 GiB 16 GiB [0,2,3,4,5]      0              0     
```

Hiểu một số thông tin trong kết quả trả về ở trên:

- `1.0`: đây là tên PG.
- `[3,5,0]`: là id của các osd mà chứa PG `1.0`. Ở đây osd 0, 3, 5 đang giữ osd `1.0`
- `active+clean`: là trạng thái hiện thời của PG

Ở cột `PG_SUM` cho biết số lượng PG mà osd đang giữ. Như ví dụ trên, với 1 pool có số PG là 1 và replicate là 3 thì tổng số PG trên cluster là 1x3=3, osd 0, 3, 5 đang giữ 1 PG.

Xem thêm ví dụ 1 pool có số lượng PG là 2, replicate = 3

```sh
~# ceph pg dump
dumped all
version 338
stamp 2018-09-10 03:21:14.681186
last_osdmap_epoch 0
last_pg_scan 0
PG_STAT OBJECTS MISSING_ON_PRIMARY DEGRADED MISPLACED UNFOUND BYTES LOG DISK_LOG STATE            STATE_STAMP                VERSION REPORTED UP      UP_PRIMARY ACTING  ACTING_PRIMARY LAST_SCRUB SCRUB_STAMP                LAST_DEEP_SCRUB DEEP_SCRUB_STAMP           SNAPTRIMQ_LEN 
2.1           0                  0        0         0       0     0   0        0 creating+peering 2018-09-10 11:28:59.285812     0'0      0:0 [1,5,3]          1 [1,5,3]              1        0'0 2018-09-10 11:28:58.751900             0'0 2018-09-10 11:28:58.751900             0 
2.0           0                  0        0         0       0     0   0        0 creating+peering 2018-09-10 11:28:58.861004     0'0      0:0 [3,5,0]          3 [3,5,0]              3        0'0 2018-09-10 11:28:58.751900             0'0 2018-09-10 11:28:58.751900             0 
                  
2 0 0 0 0 0 0 0 0 
                    
sum 0 0 0 0 0 0 0 0 
OSD_STAT USED    AVAIL  TOTAL  HB_PEERS    PG_SUM PRIMARY_PG_SUM 
5        1.0 GiB 15 GiB 16 GiB [0,1,2,3,4]      2              0 
4        1.0 GiB 15 GiB 16 GiB [0,1,2,3,5]      0              0 
3        1.0 GiB 15 GiB 16 GiB [0,1,2,4,5]      2              1 
2        1.0 GiB 15 GiB 16 GiB [0,1,3,4,5]      0              0 
0        1.0 GiB 15 GiB 16 GiB [1,2,3,4,5]      1              0 
1        1.0 GiB 15 GiB 16 GiB [0,2,3,4,5]      1              1 
sum      6.0 GiB 90 GiB 96 GiB
```

Như vậy tổng số Pg là 2x3=6. Xem cột PG_SUM: 2 + 0 + 2 + 0 + 1 + 1 = 6.

## Tìm kiếm vị trí của object được lưu trên cluster

Tìm vị trí lưu của của object `disk` trong `data` pool

```sh
~# ceph osd map data disk
osdmap e35 pool 'data' (2) object 'disk' -> pg 2.eec5673a (2.0) -> up ([3,5,0], p3) acting ([3,5,0], p3)
```

- `pg 2.eec5673a (2.0)`: đây là PG để xác định vị trí lưu trữ của object. 
- `[3,5,0], p3`: thông tin về osd đang lưu object. Object này đang được lưu trên các osd 3, 5, 0. p3 cho biết osd 3 đang giữ bản primary của object.

## Tìm vị trí của osd.

Sau khi xác định vị trí của object trên osd nào rồi, chúng ta muốn biết osd đó nằm trên server nào thì có thể sử dụng lệnh sau:

```sh
~# ceph osd find 3
{
    "osd": 3,
    "ip": "10.5.8.171:6804/17967",
    "crush_location": {
        "host": "ducpx-ceph-2",
        "root": "default"
    }
}
```

## Các trạng thái của PGs

Tại một thời điểm, PG sẽ có nhiều trạng thái khác nhau. Trạng thái tối ưu nhất của pg là `active + clean`

*creating*: PG đang được tạo. Trạng thái này xuất hiện khi tạo mới các pool, khi đó ceph đang tạo mới các PG cho pool.

```sh
~# ceph -s
  cluster:
    id:     13dc2110-7e2a-4ac9-9e16-ef56d54eb7f0
    health: HEALTH_WARN
            application not enabled on 1 pool(s)
 
  services:
    mon: 3 daemons, quorum ducpx-ceph-2,ducpx-ceph-1,ducpx-ceph-3
    mgr: ducpx-ceph-1(active), standbys: ducpx-ceph-2, ducpx-ceph-3
    osd: 6 osds: 6 up, 6 in
 
  data:
    pools:   5 pools, 258 pgs
    objects: 5  objects, 133 B
    usage:   6.0 GiB used, 90 GiB / 96 GiB avail
    pgs:     24.806% pgs not active
             194 active+clean
             64  creating+peering
```

*stale*: Monitor không xác định được trạng thái của PG, PG đang thay đổi.

cluster đang hoạt động bình thường thì bị down 2 osd:

```sh
~# ceph -s
  cluster:
    id:     13dc2110-7e2a-4ac9-9e16-ef56d54eb7f0
    health: HEALTH_WARN
            2 osds down
            1 host (2 osds) down
            application not enabled on 1 pool(s)
 
  services:
    mon: 3 daemons, quorum ducpx-ceph-2,ducpx-ceph-1,ducpx-ceph-3
    mgr: ducpx-ceph-1(active), standbys: ducpx-ceph-2, ducpx-ceph-3
    osd: 6 osds: 4 up, 6 in
 
  data:
    pools:   6 pools, 322 pgs
    objects: 302  objects, 11 KiB
    usage:   6.1 GiB used, 90 GiB / 96 GiB avail
    pgs:     211 active+clean
             111 stale+active+clean
```

*undersized*: PG đang có số bản sao ít hơn so với cấu hình của pool.

```sh
~# ceph -s
  cluster:
    id:     13dc2110-7e2a-4ac9-9e16-ef56d54eb7f0
    health: HEALTH_WARN
            2 osds down
            1 host (2 osds) down
            Degraded data redundancy: 302/906 objects degraded (33.333%), 2 pgs degraded
            application not enabled on 1 pool(s)
 
  services:
    mon: 3 daemons, quorum ducpx-ceph-2,ducpx-ceph-1,ducpx-ceph-3
    mgr: ducpx-ceph-1(active), standbys: ducpx-ceph-2, ducpx-ceph-3
    osd: 6 osds: 4 up, 6 in
 
  data:
    pools:   6 pools, 322 pgs
    objects: 302  objects, 11 KiB
    usage:   6.1 GiB used, 90 GiB / 96 GiB avail
    pgs:     302/906 objects degraded (33.333%)
             320 active+undersized
             2   active+undersized+degraded
```

*degraded*: Một số object trong PG đã không được sao chép đúng số lượng bản sao.

*inconsistent*: Xảy ra khi giữa các bản sao của object không khớp với nhau, ví dụ kích thước sai hoặc bị mất object sau khi recovery.


---

# Tham khảo

http://docs.ceph.com/docs/mimic/rados/operations/pg-states/
