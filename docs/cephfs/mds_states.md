# Trạng thái của metadata server

Trong quá trình hoạt động của Mds server sẽ nhận một số trạng thái khác nhau cho biết trạng thái hoạt động hiện tại của nó như thế nào.

```sh
~# ceph -s 
...
    mds: cephfs-2/2/2 up  {0=ducpx-ceph-3=up:active,1=ducpx-ceph-2=up:active}, 1 up:standby
```

Các trạng thái hoạt động bình thường: `up:active` và `up:standby`

```sh
~# ceph fs status cephfs
cephfs - 1 clients
======
+------+---------+--------------+----------+-------+-------+
| Rank |  State  |     MDS      | Activity |  dns  |  inos |
+------+---------+--------------+----------+-------+-------+
|  0   | resolve | ducpx-ceph-1 |          |    9  |   12  |
|  1   |  failed |              |          |       |       |
+------+---------+--------------+----------+-------+-------+
+-----------------+----------+-------+-------+
|       Pool      |   type   |  used | avail |
+-----------------+----------+-------+-------+
| cephfs-metadata | metadata | 33.5k | 42.8G |
|   cephfs-data   |   data   |    0  | 42.8G |
+-----------------+----------+-------+-------+
+-------------+
| Standby MDS |
+-------------+
+-------------+
```