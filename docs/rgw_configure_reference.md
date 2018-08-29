# Object Gateway configuration reference

Trong Radosgw, Realms là namespace toàn cục có tên duy nhất chứa 1 hoặc nhiểu zonegroups, trong zonegroups chứa một hoặc nhiều zone, trong zone chứa các bucket, trong bucket chứa các object.

## Realms

Realms là namespace có tên duy nhất. Các thao tác với realms

List các realms sẵn có sau khi cài đặt radosgw

```sh
~# radosgw-admin realm list
{
    "default_info": "",
    "realms": []
}
```

Create a new realm:

```sh
~# radosgw-admin realm create --rgw-realm=movies          
{
    "id": "1fd32a5d-756d-45c8-b389-65c1a2130ecf",
    "name": "movies",
    "current_period": "5aa1d470-a168-49f8-99a8-b26b81b5cfb8",
    "epoch": 1
}
~# radosgw-admin realm list
{
    "default_info": "1fd32a5d-756d-45c8-b389-65c1a2130ecf",
    "realms": [
        "movies"
    ]
}
```

Trong danh sách realm, nên có 1 realm là realm mặc định. Nếu chỉ có một realm và nó không được chỉ định là default khi tạo, thì realms cũng nên được chỉ định là defualt.

```sh
~# radosgw-admin realm default --rgw-realm=movies
```

Delete realm

```sh
~# radosgw-admin realm delete --rgw-realm=movies
```

Get realm

```sh
~# radosgw-admin realm get --rgw-realm=<realm name>
```

- Nếu realm name không được chỉ định, thì lệnh trên sẽ get thông tin của realm default

