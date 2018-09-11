## Client sử dụng RBD Block Device
Client sử dụng ubuntu 16.04

## Cài đặt cho client

Cài ceph-common

```
~# wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
~# echo deb https://download.ceph.com/debian-mimic/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
~# apt update
~# apt install -y ceph-common
```


## 1. Client sử dụng RBD Block Device 
<Update sau>


## 2. Client sử dụng RBD Block Device được cấu hình mirroring.

Tham khảo cấu hình RBD mirroring [tại đây](./rbd_mirroring.md)

Cài đặt `rbd-nbd` cho client

```
~# apt install -y rbd-nbd
```

Thực hiện map device để sử dụng

```
~# rbd device map -t nbd data/disk-0 --name client.ubuntu --cluster ceph
/dev/nbd0
```

Lists các device

```
~# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    253:0    0  24G  0 disk 
`-vda1 253:1    0  24G  0 part /
nbd0    43:0    0   1G  0 disk
```

Định dạng filesystem cho /dev/nbd0 để mount và sử dụng

```
~# mkfs.xfs /dev/nbd0
~# mount /dev/nbd0 /mnt
~# df -h | grep nbd
/dev/nbd0      1014M   33M  982M   4% /mnt
```