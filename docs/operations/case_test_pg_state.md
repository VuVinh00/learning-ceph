# Test một số trường hợp quan trọng khi vận hành ceph.

với cluster 3 node:

- sập 1 node, 2 node
- thay osd mới
- rút osd ra khỏi server
- osd đầy trên 85%
- đầy data 
- osd bị up/down liên tục
- mấy link của a hoàn gửi



## 1. Khi các osd gần đầy

Trường hợp test: Các VMs đẩy data xuống, các osd đạt đến mức trên 85%. Xảy ra sự cố 1 osd bị down, cluster sẽ thực hiện recovery các object ở osd bị down đó. Sẽ có trường hợp 1 osd nào đó chứa nhiều PG hơn các osd khác, do đó có một số PG không thể thực hiện recovery. Khi đó PG sẽ có trạng thái `recovery_toofull` có nghĩa là quá trình recovery đang chờ (bị dừng) do osd đích (destination osd) để recovery đã đầy, không thể ghi data vào nữa.

Cách giải quyết:

- Tìm osd chứa các PG bị `recovery_toofull` bằng cách.
    - `ceph osd df`: để xem lượng data đã lưu trên từng osd.
    - `ceph pg dump_stuck unclean`: để xem PG nào đang bị `recovery_toofull` và PG đó đang nằm trên những osd nào.
    - Trong trường hợp này thường Pg nào bị recovery_toofull sẽ nằm trên osd có lượng data gần đầy.

- Sau khi xác định được osd rồi, thực hiện reweight. Cụ thể giảm weight của osd `ceph osd crush reweight osd.<id> <weight>
=> Dẫn đến khi sửa chữa được osd thì cluster cũng không khôi phục lại được trạng thái như ban đầu. Các requests sẽ bị block.

Khi dung lượng lưu trữ của hệ thống đã lên đến 70% nên mở rộng 