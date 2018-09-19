# Hướng dẫn export RGW ra NFS-ganesha trên ubuntu 16.04

Chuẩn bị 1 cluster ceph cấu hình rgw, 1 node nfs-ganesha.

## Thực hiện cài đặt trên node nfs-ganesha

Cài đặt ceph common.

```s
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-mimic/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
apt update
apt install ceph-common -y
```

Lấy file cấu hình và admin.keyring từ cluster ceph

```s
scp root@ducpx-ceph-1:/etc/ceph/ceph.conf /etc/ceph
scp root@ducpx-ceph-1:/etc/ceph/ceph.client.admin.keyring /etc/ceph
```

Cài đặt các gói bổ sung cho nfs-ganesha.

```s
apt install nfs-common -y
apt install libwbclient0 -y
apt install libjemalloc1 -y
```

Download các gói để cài đặt nfs-ganesha

```sh
wget https://chacra.ceph.com/r/nfs-ganesha-stable/V2.6-stable/b9685b89acb5a21db2a44a6b5bf5d85709e98b7d/ubuntu/xenial/flavors/ceph_mimic/pool/main/libn/libntirpc/libntirpc1_1.6.2-0-g2be0f80-1xenial_amd64.deb

wget https://chacra.ceph.com/r/nfs-ganesha-stable/V2.6-stable/b9685b89acb5a21db2a44a6b5bf5d85709e98b7d/ubuntu/xenial/flavors/ceph_mimic/pool/main/n/nfs-ganesha/nfs-ganesha_2.6.2-0-gb9685b8-1xenial_amd64.deb
wget https://chacra.ceph.com/r/nfs-ganesha-stable/V2.6-stable/b9685b89acb5a21db2a44a6b5bf5d85709e98b7d/ubuntu/xenial/flavors/ceph_mimic/pool/main/n/nfs-ganesha/nfs-ganesha-rgw_2.6.2-0-gb9685b8-1xenial_amd64.deb
```

Kiểm tra lại các file vừa tải về

```s
~# ls
libntirpc1_1.6.2-0-g2be0f80-1xenial_amd64.deb  nfs-ganesha-rgw_2.6.2-0-gb9685b8-1xenial_amd64.deb  nfs-ganesha_2.6.2-0-gb9685b8-1xenial_amd64.deb
```

Cài đặt:

```s
dpkg -i libntirpc1_1.6.2-0-g2be0f80-1xenial_amd64.deb
dpkg -i nfs-ganesha_2.6.2-0-gb9685b8-1xenial_amd64.deb
dpkg -i nfs-ganesha-rgw_2.6.2-0-gb9685b8-1xenial_amd64.deb
```

Cấu hình nfs-ganesha

```s
EXPORT
{
        Export_ID=101;

        Path = "/";

        Pseudo = "/";

        Access_Type = RW;

        Protocols = 4;

        Transports = TCP;

        Squash = No_Root_Squash;

        FSAL {  
                Name = RGW;
                User_Id = "admin";
                Access_Key_Id ="GTXK73S5O581T90DEC29";
                Secret_Access_Key = "eL35J6MhfnCOra8BgHecCpBFrXpggPbJAPgZjAJg";
        }
}

RGW {
        ceph_conf = "/etc/ceph/ceph.conf";
        name = "client.admin";
        cluster = "ceph";
        init_args = "--keyring=/etc/ceph/ceph.client.admin.keyring";
}
```

Copy script cấu hình:

```s
mkdir /usr/lib/ganesha/
cp /usr/lib/nfs-ganesha-config.sh /usr/lib/ganesha/
chmod +x /usr/lib/ganesha/nfs-ganesha-config.sh 
```

Start nfs-ganesha

```s
service nfs-ganesha start
```

Kiểm tra lại trạng thái running của nfs-ganesha service

```s
~# service nfs-ganesha status
● nfs-ganesha.service - NFS-Ganesha file server
   Loaded: loaded (/lib/systemd/system/nfs-ganesha.service; disabled; vendor preset: enabled)
   Active: active (running) since Wed 2018-09-19 23:53:46 +07; 1min 50s ago
     Docs: http://github.com/nfs-ganesha/nfs-ganesha/wiki
  Process: 5988 ExecStartPost=/bin/bash -c /bin/sleep 2 && /usr/bin/dbus-send --system   --dest=org.ganesha.nfsd --type=method_call 
  Process: 5982 ExecStartPost=/bin/bash -c prlimit --pid $MAINPID --nofile=$NOFILE:$NOFILE (code=exited, status=0/SUCCESS)
  Process: 5976 ExecStart=/bin/bash -c ${NUMACTL} ${NUMAOPTS} /usr/bin/ganesha.nfsd ${OPTIONS} ${EPOCH} (code=exited, status=0/SUCCE
 Main PID: 5980 (ganesha.nfsd)
   CGroup: /system.slice/nfs-ganesha.service
           └─5980 /usr/bin/ganesha.nfsd -L /var/log/ganesha/ganesha.log -f /etc/ganesha/ganesha.conf -N NIV_EVENT

Sep 19 23:53:44 ducpx-nfs-ganesha systemd[1]: Starting NFS-Ganesha file server...
Sep 19 23:53:46 ducpx-nfs-ganesha systemd[1]: Started NFS-Ganesha file server.
```

## Thực hiện mount ở client.

Cài đặt nfs-common

```s
apt install nfs-common -y
```

Thực hiện mount

```s
mount -t nfs4 <ip-nfs-ganesha-node>:/ /mnt
ví dụ trong bài lab này: mount -t nfs4 ducpx-nfs-ganesha:/ /mnt
```

Kiểm tra lại

```s
~# df -h
Filesystem           Size  Used Avail Use% Mounted on
...
ducpx-nfs-ganesha:/   96G  6.1G   90G   7% /mnt
```

Sau khi thực hiện mount thành công thì vẫn chưa có khả năng ghi dữ liệu vào thư mục /mnt này. User dùng để export trong file cấu hình ở trên phải tạo bucket trước.
Ví dụ: 

ở trên cluster ceph, kiểm tra các bucket của user

```s
~# s3cmd ls
2018-09-19 08:40  s3://FIRST
```

Kiểm các file trong thư mục /mnt ở client

```s
~# ls /mnt
FIRST
```

Thử tạo file, lưu ý chỉ tạo file ở trong bucket có sẵn.

```s
~# touch /mnt/FIRST/file
~# ls /mnt/FIRST/
file
```

Nếu tạo file không nằm trong bucket nào sẽ báo lỗi

```s
~# ~# touch /mnt/file
touch: setting times of '/mnt/file': No such file or directory
```

Có thể dùng lệnh `mkdir` để tạo bucket trong thư mục /mnt

```s
~# mkdir /mnt/bucket
~# ls /mnt
FIRST  bucket 
```

Kiểm tra lại các bucket ở trên cluster ceph

```s
~# s3cmd ls
2018-09-19 08:40  s3://FIRST
2018-09-19 17:16  s3://bucket
```

---

Chúc bạn thành công.