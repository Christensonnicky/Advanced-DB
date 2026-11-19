# Redis — Điều phối đặt vé và khai thác cấu trúc dữ liệu

## 1. Mục tiêu và tài liệu kiểm chứng

Lab TicketRush mô phỏng nhiều khách cùng đặt vé để khảo sát thao tác nguyên tử, thời hạn giữ ghế, giới hạn tần suất và hàng chờ. Redis 8 được khai báo trong `compose.yaml`. Mã lab ở `ticketrush.py`, bài tập bổ sung ở `redis_exercises.py`; kết quả đã lưu lần lượt trong `ticketrush_output.txt` và `exercises_output.txt`.

Các con số về tranh chấp và thời gian dưới đây là số liệu của những lần chạy đã lưu, không phải giá trị cố định cho mọi lần chạy lại.

## 2. Giữ ghế và kiểm soát quyền xác nhận

### Bước 1 — So sánh hai cách giữ chỗ

Cho 50 thread cùng tranh ghế A1. Cách đọc `GET` rồi ghi `SET` có số người nhận kết quả thành công qua năm lần thử là **4, 18, 19, 19, 20**. Khi thay bằng `SET NX EX`, cả năm lần chỉ có **một** người thành công.

Nguyên nhân là khoảng thời gian giữa lệnh đọc và lệnh ghi cho phép nhiều thread cùng thấy key chưa tồn tại. `SET ... NX EX` gộp kiểm tra sự tồn tại, ghi người giữ và thiết lập TTL thành một thao tác nguyên tử, nên chỉ một yêu cầu tạo được key.

TTL của `hold:c1:A1` giảm từ 300 xuống 298 sau hai giây. Với lượt giữ chỉ có TTL hai giây, key không còn tồn tại khi kiểm tra sau 2.5 giây. Vì thế, nếu khách đóng trình duyệt, chỗ giữ sẽ hết hiệu lực theo TTL mà không cần tác vụ ứng dụng quét để giải phóng. Đây cũng là câu trả lời cho câu hỏi 1 và 2 của báo cáo.

### Bước 2 — Giữ, xác nhận và nhả ghế bằng Lua

Ba script thực hiện các quy tắc nghiệp vụ: không giữ ghế đã bán, chỉ người đang giữ được xác nhận, và chỉ người đang giữ được nhả ghế. Chuỗi thao tác mẫu trả về `True, False, False, True, False`; sơ đồ sau đó là `{'A1': 'sold', 'A2': 'free'}`.

Thử thêm với A2: Mai giữ thành công, Nam nhả bị từ chối, Mai nhả thành công. Các ghế đã bán nằm trong set `sold:c1`, thuận tiện cho việc lấy danh sách bằng `SMEMBERS`; cùng một ghế không bị lặp trong set khi gọi `SADD` nhiều lần. Điều này không đồng nghĩa toàn bộ hàm `confirm` có tính lặp lại trả cùng kết quả: sau lần thành công, key giữ ghế đã bị xóa.

**Câu hỏi 3 — Vì sao cần kiểm tra chủ sở hữu?** Nếu bỏ so sánh người giữ, người khác có thể xác nhận A1 của khách đang giữ hợp lệ. Tình huống tương tự xảy ra khi lượt giữ cũ hết hạn, khách mới giữ lại ghế nhưng yêu cầu xác nhận của khách cũ đến muộn. Kiểm tra trong Lua gắn việc xác nhận với trạng thái giữ hiện tại và tránh khoảng hở giữa đọc và cập nhật.

## 3. Phân phối vé và điều tiết lưu lượng

### Bước 3 — Flash sale 100 vé cho 500 người

Với cách `DECR` rồi kiểm tra giá trị trả về, cả ba lần chạy đều ghi nhận **100 vé bán được, tồn kho cuối bằng 0**. Nhánh thất bại dùng `INCR` để hoàn lại lượt giảm khi kết quả âm.

Cách `GET > 0` rồi mới `DECR` lần lượt bán **116, 110, 108** vé; tồn kho cuối là **-16, -10, -8**. Việc nhiều người cùng đọc một tồn kho dương dẫn tới bán vượt. Ở cách thứ nhất, `DECR` là nguyên tử nhưng cả chuỗi giảm, hoàn lại và ghi người mua vẫn gồm nhiều lệnh; thử nghiệm này không chứng minh tính an toàn khi tiến trình chết giữa các lệnh.

### Bước 4 — Giới hạn 5 yêu cầu trong 10 giây

Gửi tám yêu cầu liên tiếp cho kết quả giống nhau ở fixed window và sliding window: năm yêu cầu được chấp nhận, ba yêu cầu bị chặn. Khác biệt xuất hiện khi năm yêu cầu nằm sát cuối cửa sổ và năm yêu cầu tiếp theo đến sau 0.4 giây:

- Fixed window chấp nhận cả mười vì các yêu cầu thuộc hai cửa sổ khác nhau.
- Sliding window chỉ chấp nhận năm yêu cầu đầu vì chúng vẫn nằm trong khoảng thời gian đang xét.

Trong luồng đặt ghế, bảy lần `try_hold` liên tiếp cho kết quả `ok, taken, taken, taken, taken, rate_limited, rate_limited`. Những lần ghế đã bị giữ vẫn được tính vào giới hạn yêu cầu.

**Câu hỏi 4 — Chọn loại cửa sổ nào?** Fixed window phù hợp khi cần bộ đếm đơn giản và chấp nhận lưu lượng dồn ở ranh giới. Sliding window phù hợp khi cần hạn chế số lượt trong khoảng thời gian trượt, chẳng hạn chống bot lúc mở bán. Script mẫu cập nhật sorted set nguyên tử; độ chính xác theo thời gian còn phụ thuộc timestamp do client cung cấp.

### Bước 5 và 6 — Hàng chờ và kiểm thử tranh chấp

Sorted set xếp u1, u2, u3 ở vị trí 1, 2, 3. Nhờ `NX`, u1 vào lại không bị thay score nên vẫn giữ vị trí đầu. Lệnh `admit` lấy hai người trả về `['u1', 'u2']`, sau đó u3 đứng thứ nhất.

Stress test cho 100 người tranh 20 ghế được lặp mười lần. Cả mười lần đều xuất hiện bộ số `20 20 20`: số lượt mua thành công, số ghế phân biệt trong kết quả và số phần tử của set ghế bán. Các lần thử này không ghi nhận bán trùng ghế.

## 4. Câu hỏi 5 — Trạng thái tạm và dữ liệu cần lưu bền vững

Đơn hàng, người sở hữu vé và thông tin thanh toán cần được lưu trong cơ sở dữ liệu bền vững. Giữ chỗ, hàng chờ và bộ đếm có thể có vòng đời ngắn hơn, nhưng mất chúng vẫn ảnh hưởng đến quyền giữ chỗ, thứ tự phục vụ hoặc giới hạn truy cập.

Không thể suy ra Redis chỉ mất tối đa một giây dữ liệu từ cấu hình lab: file Compose chưa bật AOF hay khai báo volume riêng. Mức mất khoảng một giây thường được nhắc tới cho AOF với `appendfsync everysec`, không phải cam kết chung. Tham khảo [Redis persistence](https://redis.io/docs/latest/management/persistence/).

Một thiết kế mở rộng có thể lấy DB đơn hàng làm nguồn dữ liệu chính, dùng ràng buộc duy nhất trên `(event, seat)` và xử lý thanh toán theo mã yêu cầu để tránh ghi trùng. Sau khi giao dịch DB hoàn tất, trạng thái Redis được đồng bộ và có cơ chế thử lại/đối soát. Nếu giữ chỗ hết hạn hoặc `confirm` thất bại, hệ thống phải giải quyết trạng thái lệch; chỉ sắp xếp thứ tự hai lệnh ở hai hệ thống không tạo ra giao dịch nguyên tử chung. Khi cần khôi phục, `sold:{event}` được dựng từ các đơn đã xác nhận trong DB.

## 5. Bài tập bổ sung

### Bài 1 — Bảng xếp hạng

Sáu người chơi được đưa vào sorted set bằng `ZADD`. Sau khi Nam được cộng 250 điểm qua `ZINCRBY`, `ZREVRANGE board 0 2 WITHSCORES` trả về top ba: **Mai 1430, Tuan 1300, Nam 1200**. `ZREVRANK` trả vị trí bắt đầu từ 0 nên cần cộng 1 để hiển thị hạng. Nam và Linh cùng 1200 điểm; thứ tự khi bằng điểm còn phụ thuộc quy tắc sắp member của sorted set.

### Bài 2 — Hồ sơ và chỉ mục thành phố

Thông tin `name`, `city`, `age` của mỗi người nằm trong hash `user:{id}`. Set `city:{tên}` lưu các ID cùng thành phố để ứng dụng tra cứu trước, rồi lấy hồ sơ bằng `HGETALL`. Hanoi ban đầu trả về Linh, Mai và Tuan.

Khi Lan chuyển từ Ho Chi Minh sang Hanoi, chương trình cập nhật hash bằng `HSET` và chuyển ID 5 giữa hai set bằng `SMOVE`. Kết quả Hanoi có Linh, Mai, Lan, Tuan. Đây là chỉ mục do ứng dụng quản lý; hai lệnh hiện chạy riêng, nên nếu cần chống cập nhật dở dang phải bổ sung cơ chế cập nhật nhất quán.

### Bài 3 — Cache-aside

Hàm `slow_query` mô phỏng truy vấn một giây. Với 100 yêu cầu cho cùng sản phẩm, tổng thời gian không dùng cache là **100.06 giây**; có cache là **1.141 giây**, tương đương hệ số **87.7 lần** trong log.

Lần đầu mất 1005.5 ms do cache miss. 99 lượt tiếp theo có thời gian trung bình 1.371 ms nhờ cache hit. Quy trình là đọc Redis, truy vấn nguồn nếu thiếu, rồi ghi JSON vào cache với TTL 300 giây. Mức tăng tốc này áp dụng cho dữ liệu giả lập và chuỗi yêu cầu lặp đã đo; tỷ lệ hit, thời gian truy vấn nguồn và chính sách làm mới sẽ quyết định hiệu quả ở ứng dụng khác.
