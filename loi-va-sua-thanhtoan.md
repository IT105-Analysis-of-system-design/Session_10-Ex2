# Tìm lỗi và sửa Sequence Diagram — Chức năng Thanh toán RikkeiBank

## Lỗi 1 — Bước 2: Kiểm tra định dạng thẻ

**Thực tập sinh vẽ:** thông điệp **Async** gửi sang một Lifeline mới được dựng thêm chỉ để xử lý việc kiểm tra định dạng thẻ.

**Vì sao sai:**
- Kịch bản chỉ có 4 đối tượng cố định (Khách hàng, Cổng Thanh Toán, Ngân Hàng Lõi, EmailServer). Việc "tự kiểm tra định dạng thẻ" là xử lý **nội bộ** của Cổng Thanh Toán, không có object nào khác tham gia.
- Dựng thêm 1 lifeline mới cho việc này là bịa ra một thành phần không tồn tại trong bài toán, khiến Dev hiểu nhầm cần xây thêm một service/class riêng để validate định dạng thẻ → sai kiến trúc hệ thống, thừa thành phần không cần thiết.

**Loại đúng phải dùng:** **Self** — mũi tên vòng cung quay lại chính Lifeline "Cổng Thanh Toán".

## Lỗi 2 — Bước 6: Gửi email hóa đơn

**Thực tập sinh vẽ:** thông điệp **Sync** (kèm mũi tên Return đi ngay sau) từ Cổng Thanh Toán sang EmailServer.

**Vì sao sai:**
- Đề bài quy định rõ: gửi hóa đơn "không được chờ EmailServer phản hồi mới đi tiếp".
- Sync nghĩa là activation bar của Cổng Thanh Toán bị **khóa** cho đến khi EmailServer xử lý và trả lời xong.
- Nếu dịch vụ email phản hồi chậm (mạng lag, server quá tải...), toàn bộ luồng thanh toán — vốn đã hoàn tất giao dịch ở các bước trước — sẽ bị **treo/chờ vô ích** chỉ vì một tác vụ phụ không ảnh hưởng đến kết quả giao dịch chính. Đây chính là lỗi có thể khiến luồng xử lý bị chờ khi dịch vụ email phản hồi chậm.

**Loại đúng phải dùng:** **Async** (mũi tên nét liền, đầu hở), **không kèm Return** — Cổng Thanh Toán gửi xong là đi tiếp ngay lập tức.

## Các bước giữ nguyên vì đã đúng

| Bước | Thông điệp | Loại | Vì sao đúng |
|---|---|---|---|
| 3 | `processPayment()`: Cổng Thanh Toán → Ngân Hàng Lõi | Sync | Cần chờ Ngân Hàng Lõi xử lý xong mới biết kết quả giao dịch |
| 4 | Kết quả giao dịch: Ngân Hàng Lõi → Cổng Thanh Toán | Return | Khép lại activation bar của lời gọi Sync ở bước 3 |
| 5 | Hiển thị thành công: Cổng Thanh Toán → Khách hàng | Return | Khép lại activation bar của lời gọi Sync ban đầu từ Khách hàng ở bước 1 |

## Sơ đồ đã sửa (tóm tắt luồng đúng)

```
Khách hàng --Sync(nhập thẻ tín dụng)--> Cổng Thanh Toán
                                            |◄─┐ Self: kiểm tra định dạng thẻ()
                                            |──┘
Cổng Thanh Toán --Sync(processPayment)--> Ngân Hàng Lõi
Ngân Hàng Lõi --Return(kết quả giao dịch)--> Cổng Thanh Toán
Cổng Thanh Toán --Async(sendReceiptEmail, không chờ)--> EmailServer
Cổng Thanh Toán --Return(hiển thị thành công)--> Khách hàng
```

Xem file `thanhtoan-sequence-fixed.drawio` để có bản vẽ đầy đủ.
