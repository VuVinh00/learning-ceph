# Resizing ceph rbd
Ceph có hỗ trợ tính năng thin provisioning cho block device. Ceph block device có thể dễ dàng tăng, giảm kích thước. Hệ thống filesystem của block nên hỗ trợ resizing.

Tham khảo cách tạo mới và sử dụng một block device [tại đây](./working_ceph_block_device.md)

## Tăng kích thước của image.
- Kiểm tra thông tin của image trước khi resizing

```
~# rbd -p client info disk-0 --name client.ubuntu
rbd image 'disk-0':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	id: 6039074b0dc51
	block_name_prefix: rbd_data.6039074b0dc51
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Wed Jul 11 08:49:46 2018
```

Kích thước hiện tại của image là `1 GiB`

- Thực hiện tăng kích thước image này lên 5 GiB

```
~# rbd resize -p client --image disk-0 --size 5120 --name client.ubuntu
Resizing image: 100% complete...done.
```

- Kiểm tra lại thông tin

```
~# rbd -p client info disk-0 --name client.ubunturbd image 'disk-0':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	id: 6039074b0dc51
	block_name_prefix: rbd_data.6039074b0dc51
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Wed Jul 11 08:49:46 2018
```

- Kiểm tra hệ thống filesystem.

```
~# df -h
Filesystem                   Size  Used Avail Use% Mounted on
udev                         468M     0  468M   0% /dev
tmpfs                         98M  4.6M   93M   5% /run
/dev/mapper/ubuntu--vg-root   19G  1.6G   16G  10% /
tmpfs                        488M     0  488M   0% /dev/shm
tmpfs                        5.0M     0  5.0M   0% /run/lock
tmpfs                        488M     0  488M   0% /sys/fs/cgroup
/dev/sda1                    472M   58M  390M  13% /boot
tmpfs                         98M     0   98M   0% /run/user/0
/dev/rbd0                   1014M  133M  882M  14% /mnt/ceph-disk1

~# lsblk
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                     8:0    0   20G  0 disk 
|-sda1                  8:1    0  487M  0 part /boot
|-sda2                  8:2    0    1K  0 part 
`-sda5                  8:5    0 19.5G  0 part 
  |-ubuntu--vg-root   252:0    0 18.6G  0 lvm  /
  `-ubuntu--vg-swap_1 252:1    0  980M  0 lvm  [SWAP]
sr0                    11:0    1  848M  0 rom  
rbd0                  251:0    0    5G  0 disk /mnt/ceph-disk1
```

Chúng ta thấy rằng, mặc dù image đã được resizing nhưng filesystem vẫn không thay đổi.

- Kiểm tra thông tin hệ thống cho biết image đã được thay đổi size

```
~# dmesg | grep -i capacity
[ 2189.311511] rbd0: detected capacity change from 1073741824 to 5368709120
```

- Thực hiện lệnh sau để hệ thống filesystem nhận kích thước mới của image

```
~# xfs_growfs -d /mnt/ceph-disk1
meta-data=/dev/rbd0              isize=512    agcount=9, agsize=31744 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1 spinodes=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 262144 to 1310720

~# df -h
Filesystem                   Size  Used Avail Use% Mounted on
udev                         468M     0  468M   0% /dev
tmpfs                         98M  4.6M   93M   5% /run
/dev/mapper/ubuntu--vg-root   19G  1.6G   16G  10% /
tmpfs                        488M     0  488M   0% /dev/shm
tmpfs                        5.0M     0  5.0M   0% /run/lock
tmpfs                        488M     0  488M   0% /sys/fs/cgroup
/dev/sda1                    472M   58M  390M  13% /boot
tmpfs                         98M     0   98M   0% /run/user/0
/dev/rbd0                    5.0G  134M  4.9G   3% /mnt/ceph-disk1
```

/dev/rbd0 giờ đã lên 5G.

- Việc tăng kích thước cho một image rất đơn giản. Ở trên đây, image sử dụng xfs làm filesystem có hỗ trợ resizing online. Để thực hiện việc resizing được an toàn, hãy tham khảo thêm tài liệu về các loại filesystem hỗ trợ resizing.

## Giảm kích thước của image
- Giảm kích thước của image cũng sử dụng lệnh tương tự như tăng kích thước.
- Lưu ý khi giảm: Sau khi thực hiện giảm kích thước của image, iamge sẽ mất toàn bộ dữ liệu. Image đấy sẽ trở thành một image trống chưa được định dạng. Để sử dụng, chúng ta cần phải thực hiện lại như một image mới hoàn toàn.
