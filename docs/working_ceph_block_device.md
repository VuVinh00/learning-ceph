# Working ceph block device

## Content
- Configuring Ceph client

## 1. Configuring Ceph client
- Phần này sẽ cấp phát block device cho client sử dụng. Client có thể là Ubuntu hoặc là centos.

### 1.1 Client là ubuntu
- Trước tiên thực hiện cài đặt gói ceph-common trên client.

```
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-mimic/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
apt update
apt install -y ceph-common
```

### Thực hiện trên ceph cluster.
- 1. Tạo pool cho client.

```
~# ceph osd pool create ubuntu-rbd 64 64
~# ceph osd pool application enable ubuntu-rbd rbd
```

- 2. Tạo user

```
~# ceph auth get-or-create client.ubuntu mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=ubuntu-rbd'
```

- 3. Chuyển file cấu hình ceph cho client

```
ssh root@<ip client> sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
```

- 4. Chuyển key của user ubuntu cho client

```
ceph auth get-or-create client.ubuntu | ssh root@<ip client> sudo tee /etc/ceph/ceph.client.ubuntu.keyring
```

### Thực hiện trên client
- Tạo 1 block deivce

```
rbd -p ubuntu-rbd create disk-1 --size 1024 --name client.ubuntu
```

- Kiểm tra lại rbd image vừa tạo.

```
~# rbd ls -p ubuntu-rbd --name client.ubuntu
disk-1

~# rbd -p ubuntu-rbd --image disk-1 info --name client.ubuntu
rbd image 'disk-1':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	id: 2333674b0dc51
	block_name_prefix: rbd_data.2333674b0dc51
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Tue Jun 26 10:40:37 2018
```

- client phải map rbd image mới có thể thấy nó như một ổ đĩa.

```
~# rbd -p ubuntu-rbd map disk-1 --name client.ubuntu
rbd: sysfs write failed
RBD image feature set mismatch. Try disabling features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
```

Lúc này chúng ta sẽ thấy map bị failed do rbd đặt tính năng `mismatch`. Chúng ta cần phải disable các tính năng để sử dụng map rbd image.

```
~# rbd -p ubuntu-rbd feature disable disk-1 exclusive-lock object-map deep-flatten fast-diff --name client.ubuntu
```

- Tiến hành map lại

```
~# rbd -p ubuntu-rbd map test --name client.ubuntu
/dev/rbd0
```

- Kiểm tra lại các block device đã map

```
~# rbd showmapped --name client.ubuntu
id pool       image snap device    
0  ubuntu-rbd test  -    /dev/rbd0
```

- Sử dụng `lsblk` ta sẽ thấy rbd0 hiện ra như một ổ đĩa được gắn vào client

```
~# lsblk
NAME                  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                     8:0    0   20G  0 disk 
|-sda1                  8:1    0  487M  0 part /boot
|-sda2                  8:2    0    1K  0 part 
`-sda5                  8:5    0 19.5G  0 part 
  |-ubuntu--vg-root   252:0    0 18.6G  0 lvm  /
  `-ubuntu--vg-swap_1 252:1    0  980M  0 lvm  [SWAP]
sr0                    11:0    1  848M  0 rom  
rbd0                  251:0    0    1G  0 disk
```

Bây giờ chúng ta đã có thể tiến hành format, mount và sử dụng block device như một ổ đĩa bình thường.
 
```
~# fdisk -l /dev/rbd0
~# mkfs.xfs /dev/rbd0
~# mkdir /mnt/ceph-disk1
~# mount /dev/rbd0 /mnt/ceph-disk1
~# df -h /mnt/ceph-disk1
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0      1014M   33M  982M   4% /mnt/ceph-disk1
```

- Để client có thể tự động mount block device tự động khi reboot, ta cần phải cấu hình như sau:
- Sử file `vi /etc/ceph/rbdmap`

```
ubuntu-rbd/disk-1      id=ubuntu,keyring=/etc/ceph/ceph.client.ubuntu.keyring
```

và file `/etc/fstab`

```
/dev/rbd0   /mnt  xfs defaults,noatime,_netdev        0       0
```

### Với client là centos.
- Các bước tương tự như với ubuntu.
- Với centos, bước cài đặt sẽ khác. Chúng ta sẽ thử sử dụng lại block đã cấp phát cho ubuntu để xem các file mà ubuntu tạo ra trên block có được thấy ở centos không?

#### Tiến hành cài đặt.
- install packages

```sh
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ 
sudo yum install --nogpgcheck -y epel-release 
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 
sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
```


```sh
sudo rpm --import 'https://download.ceph.com/keys/release.asc'

vi /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-mimic/el7/$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-mimic/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-mimic/el7/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

yum update -y
yum install ceph-common -y
```

- Sau đó thực hiện các bước chuyển file cấu hình, chuyển key của client.ubuntu sang cho centos.
- Thực hiện map, mount và liệt kê ta thấy được các file ở do ubuntu tạo ra.





















































