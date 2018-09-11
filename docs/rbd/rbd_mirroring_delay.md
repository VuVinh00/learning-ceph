## Cấu hình RBD mirroring delay

Tham khảo cách cấu hình rbd mirroring cơ bản [tại đây](./rbd_mirroring.md)

Quá trình đồng bộ dữ liệu giữ 2 cluster có thể cấu hình cho phép delay. Thực hiện cấu hình delay với tham số `rbd mirroring replay delay = <minimun delay in seconds>` trong file cấu hình. Giá trị mặc định 0

Quá trình mirroring là đồng bộ từ journaling của image ở primary site sang secondary site. Do đó, các tham số cấu hình của journal cũng ảnh hưởng đến quá trình delay. Các tham số của journal ảnh hưởng đến mirroring:

```
/**
 * RBD Mirror options
 */
OPTION(rbd_mirror_journal_commit_age, OPT_DOUBLE, 5) // commit time interval, seconds
OPTION(rbd_mirror_journal_poll_age, OPT_DOUBLE, 5) // maximum age (in seconds) between successive journal polls
OPTION(rbd_mirror_journal_max_fetch_bytes, OPT_U32, 32768) // maximum bytes to read from each journal data object per fetch
OPTION(rbd_mirror_sync_point_update_age, OPT_DOUBLE, 30) // number of seconds between each update of the image sync point object number, tham số này chưa rõ
OPTION(rbd_mirror_concurrent_image_syncs, OPT_U32, 5) // maximum number of image syncs in parallel
```

### Kiểm tra băng thông của mạng khi mirroring

Tạo 10 image ở primary site

```
~# for ((i = 0; i < 10; i++)) do 
rbd create disk-$i --size 3720 --pool data --image-feature exclusive-lock,journaling --name client.ubuntu --cluster ceph
done
~# rbd -p data list
disk-0
disk-1
disk-2
disk-3
disk-4
disk-5
disk-6
disk-7
disk-8
disk-9
```

Kiểm tra bên secondary site

```
~# rbd -p data list --cluster backup
disk-0
disk-1
disk-2
disk-3
disk-4
disk-5
disk-6
disk-7
disk-8
disk-9
```

Client map và sử dụng đồng thời 10 image

Map image

```
for ((i = 0; i < 10; i++)) do  
rbd device map -t nbd data/disk-$i --name client.ubuntu --cluster ceph 
done
```

Create filesystem

```
~# for ((i = 0; i < 10; i++)) do
mkfs.xfs /dev/nbd$i
done
```

Tạo thư mục để mount

```
~# for ((i = 0; i < 10; i++)) do 
> mkdir /mnt/disk-$i
> done
```

Mount

```
for ((i = 0; i < 10; i++)) do 
mount /dev/nbd$i /mnt/disk-$i
done
```

Kiểm tra lại

```
~# df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/nbd0       3.7G   33M  3.6G   1% /mnt/disk-0
/dev/nbd1       3.7G   33M  3.6G   1% /mnt/disk-1
/dev/nbd2       3.7G   33M  3.6G   1% /mnt/disk-2
/dev/nbd3       3.7G   33M  3.6G   1% /mnt/disk-3
/dev/nbd4       3.7G   33M  3.6G   1% /mnt/disk-4
/dev/nbd5       3.7G   33M  3.6G   1% /mnt/disk-5
/dev/nbd6       3.7G   33M  3.6G   1% /mnt/disk-6
/dev/nbd7       3.7G   33M  3.6G   1% /mnt/disk-7
/dev/nbd8       3.7G   33M  3.6G   1% /mnt/disk-8
/dev/nbd9       3.7G   33M  3.6G   1% /mnt/disk-9
```

Tạo ra các file trong các thư mục đã mount ở trên

```
for ((i = 0; i < 10; i++)) do 
dd if=/dev/zero of=/mnt/disk-$i/file count=20480 bs=1024
done
```

Unmap

```
for ((i = 2; i < 10; i++)) do 
umount /dev/nbd$i
rbd-nbd unmap /dev/nbd$i
done
```

delete

for ((i = 2; i < 10; i++)) do 
rbd -p data remove disk-$i --cluster ceph
done