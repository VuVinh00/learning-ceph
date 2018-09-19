# Ghi chép về một số vấn đề gặp phải đối với PG

## STUCK PLACEMENT GROUPS

Các PG sẽ rơi vào trạng thái `degraded` hoặc là `peering` khi xảy ra lỗi. PGs sẽ ở trong trạng thái này khi cluster đang thực hiện recovery. Tuy nhiên, nếu PGs ở trong trạng thái này quá lâu thì có thể xảy ra vấn đề khác cần tìm hiểu. Do đó, monitor sẽ cảnh báo các trạng thái của PG không tối ưu. Một số trạng thái bị stuck mà ta có thể kiểm tra:

- inactive: PG không được active quá lâu, lúc này PG không thể phụ vụ yêu cầu read/write
- unclean: PG không được clean quá lâu do không thể hoàn thành recover từ một sự cố nào đó.
- stale: Xảy ra khi monitor không nhận được thông tin cập nhật từ osd chứa primary của pg hoặc các peer osd báo cáo osd primary bị down.

Các PG có trạng thái này có thể được liệt kê ra bằng các lệnh:

```sh
ceph pg dump_stuck stale
ceph pg dump_stuck inactive
ceph pg dump_stuck unclean
```

ví dụ:

```sh
~# ceph pg dump_stuck inactive
ok
PG_STAT STATE                            UP  UP_PRIMARY ACTING ACTING_PRIMARY 
12.0    stale+undersized+degraded+peered [1]          1    [1]              1 
12.1    stale+undersized+degraded+peered [0]          0    [0]              0 
12.2          undersized+degraded+peered [4]          4    [4]              4 
12.5          undersized+degraded+peered [4]          4    [4]              4 
12.3          undersized+degraded+peered [4]          4    [4]              4 
12.4          undersized+degraded+peered [5]          5    [5]              5 
```



inactive|unclean|stale|undersized|degraded