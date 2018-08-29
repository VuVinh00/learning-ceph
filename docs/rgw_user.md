# Sử dụng Radosgw

Có 2 loại user:

- User: Dùng để ám chỉ user với S3 interface
- Subuser: Dùng để ám chỉ user với swift interface. Subuser được liên kết với user.

Create user:

```sh
~# radosgw-admin user create --uid="User_radosgw" --display-name="User RadosGW"
{
    "user_id": "User_radosgw",
    "display_name": "User RadosGW",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "User_radosgw",
            "access_key": "P94E6VS0KT1B66X88G36",
            "secret_key": "hwplHXLiFDuSxB4GHY83APoMf91eNEx1WDmbBRxo"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

Tạo Subuser: subuser yêu cầu uid của user.

```sh
~# radosgw-admin subuser create --uid=User_radosgw --subuser=User_radosgw:swift --access=full
{
    "user_id": "User_radosgw",
    "display_name": "User RadosGW",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [
        {
            "id": "User_radosgw:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "User_radosgw",
            "access_key": "P94E6VS0KT1B66X88G36",
            "secret_key": "hwplHXLiFDuSxB4GHY83APoMf91eNEx1WDmbBRxo"
        }
    ],
    "swift_keys": [
        {
            "user": "User_radosgw:swift",
            "secret_key": "IpwTJYMmB1l5F1nCmSCPetmrgmfi3Bwdzl6uB79X"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

Note: access=full, full không chỉ read và write, mà bao gồm cả chính sách kiểm soát truy cập.

List users:

```sh
~# radosgw-admin user list
[
    "User_radosgw"
]
```

GET USER INFO:

```sh
~# radosgw-admin user info --uid=User_radosgw
{
    "user_id": "User_radosgw",
    "display_name": "User RadosGW",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [
        {
            "id": "User_radosgw:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [
        {
            "user": "User_radosgw",
            "access_key": "P94E6VS0KT1B66X88G36",
            "secret_key": "hwplHXLiFDuSxB4GHY83APoMf91eNEx1WDmbBRxo"
        }
    ],
    "swift_keys": [
        {
            "user": "User_radosgw:swift",
            "secret_key": "IpwTJYMmB1l5F1nCmSCPetmrgmfi3Bwdzl6uB79X"
        }
    ],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```

MODIFY USER INFO: thông tin user có thể thay đổi được, phải chỉ định `--uid=<uid user>`, các thông tin thường thay đổi là key, secret, display, email, và access level.

```sh
~# radosgw-admin user modify --uid=User_radosgw --display-name="New name of user_radosgw"
```

USER ENABLE/SUSPEND: Khi tạo mới user, user đó mặc định được enable. User có đang enable có thể suspend sau đó re-enable.

```sh
# Khi suspend user thì subuser cũng bị suspend.
~# radosgw-admin user suspend --uid=User_radosgw
```

```sh
~# radosgw-admin user enable --uid=User_radosgw
```

REMOVE A USER: Khi xóa một user thì subuser cũng bị xóa theo, tuy nhiên nếu muốn chỉ xóa bỏ subuser thì vẫn có thể.

```sh
# Remove a user
~# radosgw-admin user rm --uid=User_radosgw

# Remove a subuser
~# radosgw-admin subuser rm --uid=User_radosgw:swift
```

- Sử dụng option `--purge-data` để xóa tất cả các data liên quan đến user
- Sử dụng option `--purge-key` để xóa tất cả các key liên quan đến user.

ADD / REMOVE A KEY: cả user và subuser đều yêu cầu phải có key để truy cập vào s3 hoặc swift api. Để sử dụng s3, user cần cặp khóa `access key` và `secret key`