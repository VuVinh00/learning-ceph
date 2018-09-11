# RBD MIRRORING

RBD mirroring là sự đồng bộ các bản sao (replication) của các rbd images giữa các cluster ceph với nhau.

  ![](../images/rbd_mirroring.png)

RBD mirroring có thể được cấu hình theo 2 cách `one way` và `two way`
- one way: dữ liệu sẽ được mirror từ primary site sang secondary site. RBD mirror daemon chỉ cần chạy trên secondary site.
- two way: Dữ liệu sẽ được mirror ở cả 2 site.

## Mục lục

[1. Cấu hình one way](#1)
- [mode pool](#2)
- [mode image](#3)

[2. Cấu hình two way](#4)

[Tham khảo](#5)

<a name=1></a>

# Cấu hình one way.

Thực hiện rbd mirroring với dạng active-passive: Dữ liệu sẽ được đồng bộ từ primary site sang secondary site.

Chúng ta phải có 2 cluster ceph khác nhau: ceph và backup cluster.

Trên backup cluster, thực hiện đổi tên file cấu hình `ceph.conf` thành `backup.conf` và file admin key `ceph.client.admin.keyring` thành `backup.client.admin.keyring`

```
~# mv /etc/ceph/ceph.client.admin.keyring /etc/ceph/backup.client.admin.keyring
~# mv /etc/ceph/ceph.conf /etc/ceph/backup.conf
```

Sau khi đổi tên file, thực hiện các cli ceph với cluster này đều phải thêm option `--cluster backup`

- 1. Tạo pool `data` trên cả 2 clusters (pool này sẽ được đồng bộ trên cả 2 cluster).

```
# Thực hiện trên ceph cluster
~# ceph osd pool create data 8 8 --cluster ceph

# Thực hiện trên backup cluster
~# ceph osd pool create data 8 8 --cluster backup
```

- 2. Tạo `client.primary` user trên `ceph` cluster có quyền truy cập vào `data` pool

```
# Thực hiện trên ceph cluster
~# ceph auth get-or-create client.primary mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=data' -o /etc/ceph/ceph.client.primary.keyring --cluster ceph
```

- 3. Tạo `client.secondary` user trên `backup` cluster.

```
# Thực hiện trên backup cluster
~# ceph auth get-or-create client.secondary mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=data' -o /etc/ceph/backup.client.secondary.keyring --cluster backup
```

- 4. Chuyển file cấu hình và file keyring từ cluster sang cluster khác.

```
# trên ceph cluster
~# cd /etc/ceph
/etc/ceph# ls
ceph.client.admin.keyring  ceph.client.primary.keyring  ceph.conf  rbdmap  tmpEN3FPb
/etc/ceph# scp ceph.conf ceph.client.primary.keyring root@<ip backup cluster>:/etc/ceph

# trên backup cluster
~# cd /etc/ceph
/etc/ceph# ls
backup.client.admin.keyring   backup.conf                ceph.conf  tmpDYx8uC 
backup.client.secondary.keyring  ceph.client.primary.keyring  rbdmap

/etc/ceph# scp backup.client.secondary.keyring backup.conf root@<ip ceph cluster>:/etc/ceph
```

#### Cấu hình pool 
RBD mirroring có thể áp dụng cho cả pool hoặc cho từng image cụ thể. Nếu enable mirror cả pool thì tất cả các image được tạo sẽ được mirroring. Nếu enable cho từng image thì chỉ có image nào được enable sẽ được mirroring.

- 1. Cài đặt `rbd-mirror` lên backup cluster.

```
sudo apt install -y rbd-mirror
```

<a name=2></a>

### Thực hiện mirroring cả pool
- 2. enable pool data để sử dụng tính năng mirror

```
# trên ceph cluster
~# rbd mirror pool enable data pool --cluster ceph
~# rbd mirror pool info data --cluster ceph
Mode: pool
Peers: none

# trên backup cluster
~# rbd mirror pool enable data pool --cluster backup 
~# rbd mirror pool info data --cluster backup
Mode: pool
Peers: none
```

Mode là pool có nghĩa là data pool được enable ở chế độ pool. Bất kỳ image nào được tạo trong pool này đều được mirroring

- 3. Thêm peer cluster cho data pool

```
# Thực hiện trên backup cluster
~# rbd mirror pool peer add data client.primary@ceph --cluster backup 
~# rbd mirror pool info data --cluster backup
Mode: pool
Peers: 
  UUID                                 NAME CLIENT
  34b52c2e-038c-49f5-bdfa-2686448652c3 ceph client.primary
```

- 4. Tạo rbd images để kiểm tra sự đồng bộ

```
# Thực hiện tạo các image trên ceph cluster
~# rbd create image-1 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd create image-2 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd create image-3 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd -p data ls --cluster ceph
image-1
image-2
image-3
```

- 5. Kiểm tra trên backup cluster

```
~# rbd mirror pool status data --cluster backup
health: OK
images: 0 total
```

Ở đây các rbd images từ ceph cluster chưa được đồng bộ sang backup cluster.

```
~# rbd-mirror -m 10.10.10.21 -d --cluster backup
2018-07-02 09:41:31.378 7fab8809f140  0 ceph version 13.2.0 (79a10589f1f80dfe21e8f9794365ed98143071c4) mimic (stable), process (unknown), pid 3115
2018-07-02 09:41:31.406 7fab8809f140  1 mgrc service_daemon_register rbd-mirror.84141 metadata {arch=x86_64,ceph_release=mimic,ceph_version=ceph version 13.2.0 (79a10589f1f80dfe21e8f9794365ed98143071c4) mimic (stable),ceph_version_short=13.2.0,cpu=Intel(R) Core(TM) i5-3320M CPU @ 2.60GHz,distro=ubuntu,distro_description=Ubuntu 16.04.4 LTS,distro_version=16.04,hostname=cluster-1,id=admin,instance_id=84141,kernel_description=#140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018,kernel_version=4.4.0-116-generic,mem_swap_kb=1003516,mem_total_kb=997652,os=Linux}
2018-07-02 09:41:31.474 7fab8809f140  0 rbd::mirror::PoolReplayer: 0x562060ebcfc0 init_rados: reverting global config option override: mon_host: 10.10.10.21 -> 10.10.10.11,10.10.10.12,10.10.10.13
```

rbd-mirror sẽ chạy forceground.

- Kiểm tra lại các images ở trong pool data trên backup cluster.

```
~# rbd -p data ls --cluster backup
image-1
image-2
image-3
```

- Kiểm tra lại thông tin của image ở trên 2 cluster.

```
# Thực hiện trên ceph cluster
~# rbd -p data info image-1 
rbd image 'image-1':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 117074b0dc51
        block_name_prefix: rbd_data.117074b0dc51
        format: 2
        features: exclusive-lock, journaling
        op_features: 
        flags: 
        create_timestamp: Wed Jul 11 23:42:41 2018
        journal: 117074b0dc51
        mirroring state: enabled
        mirroring global id: f07b714a-d69f-43f9-8c37-fef8f30cc280
        mirroring primary: true

# Thực hiện trên backup cluster
~# rbd -p data info image-1 --cluster backup
rbd image 'image-1':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 104d109cf92e
        block_name_prefix: rbd_data.104d109cf92e
        format: 2
        features: exclusive-lock, journaling
        op_features: 
        flags: 
        create_timestamp: Wed Jul 11 23:44:46 2018
        journal: 104d109cf92e
        mirroring state: enabled
        mirroring global id: f07b714a-d69f-43f9-8c37-fef8f30cc280
        mirroring primary: false
```

Ta thấy rằng, image ở trên ceph cluster là `mirroring primary: true` còn ở trên backup cluster là `mirroring primary: false` 

<a name=3></a>

### Thực hiện mirroring cho từng image cụ thể
Cùng với data pool ở trên, ta chuyển từ `Mode: pool` sang `Mode: image`

```
# Trên ceph cluster
~# rbd mirror pool enable data image --cluster ceph
note: changing mirroring mode from pool to image
~# rbd mirror pool info data --cluster ceph  
Mode: image
Peers: none

# Thực hiện trên backup cluster
~# rbd mirror pool enable data image --cluster backup
note: changing mirroring mode from pool to image
~# rbd mirror pool info data --cluster backup
Mode: image
Peers: 
  UUID                                 NAME CLIENT         
  8a3496cf-7bde-4b70-81a4-8083df10be2f ceph client.primary
```

Chúng ta sẽ tạo ra 3 image ở trên ceph cluster và chỉ định một image được mirroring.

```
~# rbd create disk-1 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd create disk-2 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd create disk-3 --size 1024 --pool data --image-feature exclusive-lock,journaling --cluster ceph
~# rbd -p data ls --cluster ceph
disk-1
disk-2
disk-3
```

Kiểm tra trên backup cluster sẽ không có image nào cả.

Thực hiện enable một disk-3 được mirroring

```
~# rbd mirror image enable data/disk-1 --cluster ceph
Mirroring enabled

~# rbd -p data info disk-1 --cluster ceph
rbd image 'disk-1':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 119474b0dc51
        block_name_prefix: rbd_data.119474b0dc51
        format: 2
        features: exclusive-lock, journaling
        op_features: 
        flags: 
        create_timestamp: Thu Jul 12 00:13:41 2018
        journal: 119474b0dc51
        mirroring state: enabled
        mirroring global id: 6ce49f8a-3228-450e-967c-944dd279bb5e
        mirroring primary: true
~# rbd -p data info disk-2 --cluster ceph                     
rbd image 'disk-2':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 119774b0dc51
        block_name_prefix: rbd_data.119774b0dc51
        format: 2
        features: exclusive-lock, journaling
        op_features: 
        flags: 
        create_timestamp: Thu Jul 12 00:13:54 2018
        journal: 119774b0dc51
        mirroring state: disabled
```

Thông tin của 2 image trong cùng một pool: với disk-1 được enable mirroring và disk-2 thì không.

Kiểm tra lại trên backup cluster

```
~# rbd -p data ls --cluster backup
disk-1
root@ducpx-ceph-cluster2:~# rbd -p data info disk-1 --cluster backup
rbd image 'disk-1':
        size 1 GiB in 256 objects
        order 22 (4 MiB objects)
        id: 104d721da317
        block_name_prefix: rbd_data.104d721da317
        format: 2
        features: exclusive-lock, journaling
        op_features: 
        flags: 
        create_timestamp: Thu Jul 12 00:17:04 2018
        journal: 104d721da317
        mirroring state: enabled
        mirroring global id: 6ce49f8a-3228-450e-967c-944dd279bb5e
        mirroring primary: false
```

<a name=4></a>

## Cấu hình two way.
Tạo ra pool mới để thực hiện cấu hình two way.

Two way yêu cầu `rbd mirror daemon` chạy trên cả 2 cluster.

Two way cũng có 2 chế độ: `Mode: pool` và `Mode: image`

Việc cấu hình rất giống với one way. Cho dễ hình dung, ta thực hiện cấu hình tương tự như one way theo các bước sau:
- Bước 1: cấu hình one way cho `twoway` pool với primary cluster là ceph và secondary cluster là backup (giả sử pool dùng để cấu hình two way có tên là twoway)
- Bước 2: ngược lại, cấu hình one way cho `twoway` pool với primary cluster là backup và secondary cluster là ceph

Chú ý: Với two way, image được tạo trên cluster nào thì image đó sẽ là primary, image được mirror sang cluster khác sẽ là non-primary 

<a name=5></a>

## tham khảo
http://www.cnblogs.com/sxwen/p/8042885.html

http://docs.ceph.com/docs/mimic/rbd/rbd-mirroring/
