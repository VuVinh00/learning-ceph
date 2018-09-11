# Một số cấu hình cho Radosgw

Thêm vào file cấu hình ceph các dòng cấu hình sau cho radosgw. Ví dụ:

```sh
[client.rgw.ducpx-cephaio]
host = <ducpx-cephaio>
keyring = /var/lib/ceph/radosgw/ceph-rgw.ducpx-cephaio/keyring
rgw socket path = /var/run/ceph/ceph-client.rgw.ducpx-cephaio.1333.94016615440384.asok
log file = /var/log/ceph/ceph-client.rgw.ducpx-cephaio.log
rgw_frontends = civetweb port=80
```

mặc định radosgw sử dụng port 7480, thay đổi port bằng cặp `port=<numberport>`, lưu ý không có khoảng trắng giữa port và giá trị port.

