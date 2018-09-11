# RBD import and export

Ví dụ sử dụng rbd import và rbd export để tạo một rbd image.

## RBD import

Tạo block image

```sh
~# dd if=/dev/zero of=/tmp/disk.img bs=1M count=1
~# mkfs.ext4 /tmp/disk.img
```

Import image vừa tạo vào rbd pool

```sh
~# rbd import /tmp/disk.img disk --pool pool_import
~# rbd -p pool_import ls
disk
```

Map rbd image này để sử dụng

```sh
~# rbd map -t nbd pool_import/disk
/dev/nbd0

~# rbd-nbd list-mapped
id      pool        image snap device
4044956 pool_import disk  -    /dev/nbd0
```

Mount để sử dụng

```sh
~# mount /dev/nbd0 /mnt
~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            982M     0  982M   0% /dev
...
/dev/nbd0      1003K   17K  915K   2% /mnt

~# touch /mnt/file-{1..9}
~# ls /mnt/
file-1  file-2  file-3  file-4  file-5  file-6  file-7  file-8  file-9  lost+found
```

## RBD export

Chúng ta sẽ export image pool_import/disk ở trên.

```sh
~# rbd export disk --pool pool_import /tmp/export.img
Exporting image: 100% complete...done.
```

Mount image vừa export.

```sh
~# mount -o loop /tmp/export.img disk-0/
~# df -h
Filesystem      Size  Used Avail Use% Mounted on
...
/dev/nbd0      1003K   17K  915K   2% /mnt
/dev/loop0     1003K   17K  915K   2% /root/disk-0

~# ls /root/disk-0/
file-1  file-2  file-3  file-4  file-5  file-6  file-7  file-8  file-9  lost+found
```

Ta thấy các file trong image được giữ nguyên.

## Script backup

Chúng ta có thể sử dụng hai lệnh này để thực hiện backup và recovery các image trong 1 pool rbd.

Script để backup

```sh
#!/bin/bash

BACKUP_ROOT=/home/ceph/backups
TIME_START=$(date --rfc-3339=seconds)
TODAY=$(date --rfc-3339=date)

BACKUP_DIR=${BACKUP_ROOT}/${TODAY}

echo "Backup start ${TIME_START}"

POOLS=($(rados lspools))

for POOL in ${POOLS[@]}
do
    POOL_BACKUP_DIR=${BACKUP_DIR}/${POOL}
    mkdir -p ${POOL_BACKUP_DIR}

    IMAGES=($(rbd -p "${POOL}" ls))

    for IMAGE in "${IMAGES[@]}"
    do
        rbd export ${IMAGE} --pool ${POOL} ${POOL_BACKUP_DIR}/${IMAGE}
    done
done

END_TIME=$(date --rfc-3339=seconds)
echo "CEPH Backup completed ${END_TIME}"
```

[Tham khảo](https://nicksabine.com/post/ceph-backup/)
