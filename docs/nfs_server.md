# Install nfs - server

Install

```sh
apt update
apt install nfs-kernel-server -y
```

Create share directories

chúng ta sẽ export 2 thư mục share với 2 cấu hình khác nhau nhằm minh họa cho 2 cách nfs mount với quyền superuser.

```sh
# Thư mục nfs share
~# mkdir /var/nfs/general -p

# Thư mục /home, thư mục có sẵn nên không phải tạo
```

Cấu hình NFS export. `vi /etc/exports`

```sh
# Cấu hình share: directory_to_share    client(share_option1,...,share_optionN)
/var/nfs/general    *(rw, sync, no_subtree_check)
/home               *(rw, sync, no_root_squash, no_subtree_check)
```

Restart service

```sh
~# systemctl restart nfs-kernel-server
```
## Install in client

Install nfs-common

```sh
apt update
apt -y install nfs-common
```

Mount:

```sh
mount IP:/var/nfs/general /mnt
```
