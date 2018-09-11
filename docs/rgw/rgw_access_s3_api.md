# Accessing the Ceph object storage using S3 API

Ceph cho phép truy cập object với s3 thông qua RESTful API.

S3 yêu cầu DNS server để gán cho bucket một url để user có thể truy cập.

## 1. Cài đặt DNS server

```sh
~# apt update
~# apt install bind9 bind9utils bind9-doc -y
```

Cấu hình cho dns bind với ipv4. `systemctl edit --full bind9`. Tìm ở dòng có `ExecStart` và sửa giống như sau:

```sh
[Service]
ExecStart=/usr/sbin/named -f -u bind -4
```

Reload daemon để có thể đọc file cấu hình mới

```sh
~# systemctl daemon-reload
```

Restart bind service

```sh
~# systemctl restart bind9
```

Cấu hình bind service. `vi /etc/bind/named.conf.options`

```sh
acl "trusted" {
        10.5.8.0/22;
};

options {
        directory "/var/cache/bind";

        recursion yes;
        allow-recursion { trusted; };
        listen-on { 10.5.8.185; };
        allow-transfer { none; };

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
};
```

Cấu hình Local file. `/etc/bind/named.conf.local`

```sh
zone "ceph-s3.com" {
    type master;
    file "/etc/bind/zones/db.ceph-s3.com"; 
    allow-update { none; };
};

zone "8.5.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.5.8";
};
```

Tạo thư mục chứa các zone:

```sh
~# mkdir /etc/bind/zones/
~# cp /etc/bind/db.local /etc/bind/zones/db.ceph-s3.com
```

Cấu hình zone: `vi /etc/bind/zones/db.ceph-s3.com`

```sh
$TTL    604800
@       IN      SOA     ceph-s3.com. root.ceph-s3.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ceph-s3.com.
@       IN      A       10.5.8.185
*       IN      CNAME   *
```

Cấu hình phân giải ngược:

```sh
~# cp /etc/bind/db.127 /etc/bind/zones/db.10.5.8
~# vi /etc/bind/zones/db.10.5.8
```

nội dung file db.10.5.8 như sau:

```sh
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ceph-s3.com. root.ceph-s3.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ceph-s3.com.
185    IN      PTR      ceph-s3.com.
```

Kiểm tra cấu hình:

```sh
~# named-checkconf
~# named-checkzone ceph-s3.com /etc/bind/zones/db.ceph-s3.com 
zone ceph-s3.com/IN: loaded serial 2
OK
~# named-checkzone 185.in-addr.arpa /etc/bind/zones/db.10.5.8 
zone 185.in-addr.arpa/IN: loaded serial 1
OK
```

Restart BIND:

```sh
~# systemctl restart bind9
```

## Cài đặt trên client

Sửa file: `vi /etc/resolv.conf`, thêm vào đầu file

```sh
nameserver 10.5.8.185
search ceph-s3.com
```

Cài đặt s3cmd cho client:

```sh
~# apt install -y s3cmd
```

Cấu hình s3cmd bằng cách cung cấp access key và secret key của user.

với user có keys như sau:

```sh
"keys": [
    {
        "user": "User_radosgw",
        "access_key": "P94E6VS0KT1B66X88G36",
        "secret_key": "hwplHXLiFDuSxB4GHY83APoMf91eNEx1WDmbBRxo"
    }
]
```

cấu hình key bằng lệnh: `s3cmd --configure`. Thực hiện cung cấp các thông tin được yêu cầu, sau khi nhập xong thông tin thực như sau:

```sh
New settings:
  Access Key: QESJ12VYWK7EA2Q4ZO1T
  Secret Key: AWRPCyLhR3njC89xAAgXTEN8uu0E7HIdhAdmDlXV
  Default Region: US
  Encryption password: Welcome123
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: False
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n] n

Save settings? [y/N] y
Configuration saved to '/root/.s3cfg'
```

Các thông tin cấu hình được s3cmd lưu trong `/root/.s3cfg`. Chúng ta cần sửa một số thông tin ở trong này: `vi /root/.s3cfg`

```sh
host_base = ceph-s3.com
host_bucket = %(bucket)s.ceph-s3.com
```

Tạo bucket:

```sh
~# s3cmd mb s3://FIRST
~# s3cmd ls
s3cmd mb s3://FIRST
```

Đẩy objects vào trong bucket

```sh
~# s3cmd put /etc/hosts s3://FIRST
upload: '/etc/hosts' -> 's3://FIRST/hosts'  [1 of 1]
 97 of 97   100% in    1s    66.50 B/s  done
 ~# s3cmd ls s3://FIRST
2018-08-23 02:12        97   s3://FIRST/hosts
```
