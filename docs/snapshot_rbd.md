# Snapshot Block Device
Tham khảo cách sử dụng rbd để test snapshot [tại đây](./client_use_rbd.md)

## Tạo snapshot
Tạo snapshot

```
rbd snap create {pool-name}/{image-name}@{snap-name}
```

ví dụ:

```
rbd snap create rbd/disk-0@snap-0
```

List các snap của một image

```
rbd snap ls {pool-name}/{image-name}
```

ví dụ:

```
~# rbd snap list rbd/disk-0
SNAPID NAME     SIZE TIMESTAMP  
     8 snap-0 10 GiB Thu Jul 26 15:34:58 2018
```

Rollback

Lưu ý khi rollback: image đang được client sử dụng thì không thể thực hiện rollback. Tiến hành umount, unmap image trước khi thực hiện rollback

```
rbd snap rollback {pool-name}/{image-name}@{snap-name}
```

ví dụ:

```
rbd snap rollback rbd/disk-0@snap-0
```

- Thực hiện mount lại ở client để kiểm tra nội dung của image tại thời điểm tạo snap-0

Xóa snapshot

```
rbd snap rm {pool-name}/{image-name}@{snap-name}
```

Xóa toàn bộ snap của image

```
rbd snap purge {pool-name}/{image-name}
```

### Layering
Snapshot của iamge là read-only. Do đó, ta có thể clone snapshot ra một iamge mới để có thể ghi.

Trong quá trình clone, bản snapshot có thể vô tình bị xóa dẫn đến dữ liệu không được clone hoàn toàn. Do đó, trước khi clone một snapshot, hãy protect snapshot đó lại. Snapshot được protect sẽ không thể bị xóa.

PROTECTING A SNAPSHOT

```
rbd snap protect {pool-name}/{image-name}@{snapshot-name}
```

ví dụ:

```
rbd snap protect rbd/disk-0@snap-0
```

CLONING A SNAPSHOT

```
rbd clone {pool-name}/{image-name}@{snap-name} {pool-name}/{new-image-name}
```

ví dụ:

```
rbd clone rbd/disk-0@snap-0 new_pool/disk-1
```

- Chúng ta có thể liệt kê ra danh sách các bản clone của snapshot

```
rbd children {pool-name}/{image-name}@{snapshot-name}
```

ví dụ:

```
:~# rbd children pool_demo/disk-0@snap-0
pool_1/disk-0
pool_demo/disk-1
```

Để xóa được snapshot, phải unprotect. Lưu ý: không thể unprotect với các snapshot đang có bản clone. trước khi unprotect, hãy xóa hết các bản clone của snapshot đó.

```
rbd snap unprotect {pool-name}/{image-name}@{snapshot-name}
```
