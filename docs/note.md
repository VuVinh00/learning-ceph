# Note

Khi thay đổi cấu hình: nên thực hiện trong thư mục đã deploy, sau đó dùng ceph-deploy để push sang tất cả các note.

```sh
ceph-deploy --overwrite-conf config push <all ceph node>
```

