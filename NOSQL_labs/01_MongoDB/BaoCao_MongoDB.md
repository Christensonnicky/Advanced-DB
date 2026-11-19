# MongoDB — Khảo sát dữ liệu đơn hàng và hiệu quả truy vấn

## 1. Phạm vi thực hành

Bài thực hành sử dụng MongoDB 8 trong Docker, cấu hình tại `compose.yaml`, và thư viện Python `pymongo`. Ba nội dung được khảo sát là tính linh hoạt của document, tổng hợp doanh thu và tác động của chỉ mục ghép. Phần cài đặt nằm trong `mongodb_exercises.py`; các số liệu dưới đây được đối chiếu với lần chạy lưu ở `output.txt`.

## 2. Bài tập 1: mở rộng dữ liệu và tìm đơn theo sản phẩm

### Thiết kế dữ liệu

Collection `orders` gồm ba đơn ban đầu, mã 1001–1003, cùng năm đơn bổ sung. Các trường mở rộng được chọn theo từng tình huống nghiệp vụ:

- Đơn 1004 mô tả giao hàng bằng `shipping.address` nhiều cấp và trường `note`.
- Đơn 1005 chứa ebook với `format`, `download_limit`; mảng `payments` lưu hai lần thanh toán.
- Đơn 1006 bổ sung email, số điện thoại của khách và đánh giá trong `rating`.
- Đơn 1007 lưu trạng thái hủy và lý do trong `cancel`, không có mảng `items`.
- Đơn 1008 dành cho khách doanh nghiệp, có `tax_id`, yêu cầu hóa đơn, dòng hàng giảm giá và dòng hàng dạng combo.

Cả tám document được lưu chung một collection. Trong cấu hình bài này, không có bộ kiểm tra schema buộc mọi document phải sở hữu cùng tập trường, nên việc thêm thuộc tính không đòi hỏi thay đổi định nghĩa bảng.

### Truy vấn và diễn giải

```js
db.orders.find({ "items.sku": "BK-01" })
```

Đường dẫn `items.sku` kiểm tra giá trị SKU trong các phần tử của mảng. Với `BK-01`, tập mã đơn trả về là **1001, 1003, 1004, 1008**; với `BK-03`, kết quả là **1002, 1006**. Document 1007 không có trường tương ứng nên không khớp điều kiện, thay vì gây lỗi truy vấn.

## 3. Bài tập 2: tổng hợp doanh thu theo thành phố

### Quy tắc tính

Chỉ đơn ở trạng thái `paid` hoặc `shipped` được đưa vào thống kê. Giá trị từng dòng hàng bằng số lượng nhân đơn giá, sau đó áp dụng tỷ lệ giảm giá nếu có. Trường `discount_pct` vắng mặt được hiểu là 0; phí giao hàng và mã coupon không được tự động cộng hoặc trừ trong công thức này.

```js
db.orders.aggregate([
  { $match: { status: { $in: ["paid", "shipped"] } } },
  { $unwind: "$items" },
  { $group: {
      _id: "$customer.city",
      orders: { $addToSet: "$_id" },
      revenue: { $sum: { $multiply: [
        "$items.qty", "$items.price",
        { $subtract: [1, { $divide: [{ $ifNull: ["$items.discount_pct", 0] }, 100] }] }
      ] } }
  } },
  { $project: { _id: 0, city: "$_id", num_orders: { $size: "$orders" },
                revenue: { $round: ["$revenue", 2] } } },
  { $sort: { revenue: -1 } }
])
```

Pipeline lần lượt lọc đơn, tách dòng hàng, nhóm theo thành phố, định dạng kết quả và sắp xếp doanh thu giảm dần. Sau `$unwind`, một đơn có thể xuất hiện nhiều lần; vì vậy `$addToSet` thu thập mã đơn duy nhất, còn `$size` mới cho ra số đơn thực tế.

### Kết quả đối chiếu

- **Hanoi:** 3 đơn, doanh thu 455.5.
- **Da Nang:** 2 đơn, doanh thu 80.0.
- **Hue:** 1 đơn, doanh thu 12.5.
- **Ho Chi Minh:** 1 đơn, doanh thu 9.9.

Có thể kiểm tra riêng Hanoi bằng ba phép tính: đơn 1001 là `2 × 12.5 + 18 = 43`; đơn 1003 là `12.5`; đơn 1008 là `20 × 12.5 × 0.9 + 5 × 35 = 400`. Cộng lại được **455.5**, trùng với kết quả tổng hợp.

## 4. Bài tập 3: so sánh kế hoạch trước và sau chỉ mục

Chương trình sinh 10.000 đơn trong `orders_big` với seed 42. Truy vấn khảo sát là `{ "customer.city": "Hanoi", status: "paid" }`; chỉ mục được thêm trên `{ "customer.city": 1, status: 1 }`.

**Trước khi tạo chỉ mục:** kế hoạch dùng `COLLSCAN`, đọc 10.000 document, không đọc khóa chỉ mục, trả về 253 document và ghi nhận thời gian 5 ms.

**Sau khi tạo chỉ mục:** `IXSCAN` tìm khóa phù hợp rồi `FETCH` lấy document. Số khóa đọc và số document đọc đều bằng 253; số kết quả vẫn là 253, thời gian ghi nhận 1 ms.

Kết quả đáng chú ý là lượng document cần đọc giảm khoảng **39.5 lần**, trong khi tập kết quả không thay đổi. Chênh lệch 5 ms và 1 ms chỉ phản ánh lần chạy đã lưu; không nên dùng nó để khẳng định hệ số tăng tốc cố định cho mọi kích thước dữ liệu hoặc môi trường.

## 5. Nhận xét

Document linh hoạt giúp biểu diễn nhiều loại đơn hàng trong một collection. Tuy nhiên, phần tổng hợp vẫn cần quy ước rõ về trạng thái đơn, trường thiếu và giảm giá. Chỉ mục ghép hỗ trợ đúng bộ điều kiện truy vấn đã thử, và hiệu quả được thể hiện trực tiếp qua số document phải đọc.
