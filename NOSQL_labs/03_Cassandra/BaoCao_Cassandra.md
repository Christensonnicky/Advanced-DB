# Cassandra — Phân vùng, sao chép và thiết kế bảng cho SensorRing

## 1. Môi trường và cách đọc kết quả

Lab dùng cụm Cassandra 5 gồm ba container trong `docker-compose.yml`. Mã `sensorring.py` tạo dữ liệu cảm biến và thử consistency level; `cassandra_exercises.py` triển khai bài tập. Những quan sát sau được đối chiếu với `lab_output.txt` và `exercises_output.txt`.

Ở lần chạy đã lưu, cass1, cass2, cass3 lần lượt có IP **172.22.0.2, 172.22.0.3, 172.22.0.4**. Bước khởi tạo ghi nhận ba node `UN` và 3.000 số đo thuộc 30 cảm biến trong keyspace `iot`, có replication factor (RF) bằng 3. Địa chỉ IP, phân bố token và thời gian đo là đặc điểm của môi trường này.

## 2. Phân bố partition trên vòng token

### Bước 1 — Kết quả quan sát

Partition key `hn-01` có token `-6927399588689370405`. Với `iot` và RF = 3, `getendpoints` trả cả ba địa chỉ; `nodetool status iot` ghi mỗi node sở hữu hiệu dụng 100% vì dữ liệu có bản sao trên cả ba node.

Khi dùng `ring1` với RF = 1, hn-01, hn-04, hn-06 được ánh xạ tới 172.22.0.4; hn-02, hn-03, hn-05 tới 172.22.0.3. Tỷ lệ sở hữu của ba node là **32.7%, 31.6%, 35.7%**, cộng lại bằng 100%.

Log `nodetool ring` chứa 48 token, mỗi node có 16 token. Với hn-01, token đầu tiên không nhỏ hơn token của partition là `-6648083467482496530`, thuộc 172.22.0.4. Với hn-02, từ token `1779598930076072019` đi tới `2068553595866611698`, thuộc 172.22.0.3. Hai phép đối chiếu đều phù hợp với kết quả `getendpoints` của RF = 1.

### Câu hỏi 1 — Token giống nhau, node sở hữu khác nhau

Token được tính từ partition key theo partitioner. Khi cùng dùng Murmur3 và cùng cách biểu diễn khóa như trong lab, hn-01 cho cùng token. Node phụ trách lại phụ thuộc phân bố vnode và cấu hình cụm; IP còn do mạng Docker cấp. Vì vậy không cần IP giống ví dụ để kiểm chứng quy tắc tìm token kế tiếp. Không nên suy rộng thành khẳng định mọi cấu hình Cassandra đều bắt buộc dùng một partitioner duy nhất.

### Câu hỏi 2 — Tác động khi thêm cass4

Nếu node mới tiếp tục dùng cấu hình 16 vnode, nó sẽ nhận các khoảng token và bootstrap dữ liệu tương ứng từ các node hiện hữu. Các partition ngoài khoảng được chuyển giao không cần đổi chủ chỉ vì số node tăng. Trong phân bố cân bằng, phần dữ liệu node mới nhận xấp xỉ một phần tư; đây là ước lượng, không phải tỷ lệ cố định.

Sau khi bootstrap thành công, các node cũ có thể chạy `nodetool cleanup` để dọn dữ liệu không còn thuộc phạm vi sở hữu. Cách phân phối theo vòng token hạn chế việc ánh xạ lại hàng loạt như khi thay số chia trong `hash(key) % số_node`.

## 3. Truy vấn và thiết kế bảng theo nhu cầu đọc

### Bước 2 — Đọc theo partition và lọc toàn bảng

Điều kiện `temperature > 31.5` bị từ chối nếu không có `ALLOW FILTERING`. Tracing của lần chạy cho thấy:

- Truy vấn theo `sensor_id = 'hn-01'` thực hiện single-partition query, đọc và trả về 3 hàng; mốc hoàn tất là **13.167 µs** (13,167 microsecond).
- Truy vấn lọc nhiệt độ gửi range requests theo các đợt 49 và 16 range; tổng số hàng đọc là **2.208 + 792 = 3.000**. Kết quả đếm là **131** số đo phù hợp; mốc hoàn tất là **72.739 µs** (72,739 microsecond).

Trong log, truy vấn thứ hai dùng `count(*)`, nên phản hồi cuối là một hàng chứa giá trị 131, không phải 131 hàng dữ liệu. Khoảng **95.6%** số đo đã đọc không thỏa điều kiện. Số liệu minh họa chi phí quét trên bộ dữ liệu này, không xác định thời gian chung cho mọi cụm.

### Câu hỏi 3 — Thiết kế cho số đo vượt 31.5°C trong một ngày

Đặt ngày làm partition key và nhiệt độ làm clustering column đầu tiên giúp điều kiện ngày xác định partition, còn ngưỡng nhiệt độ giới hạn miền cần đọc:

```sql
CREATE TABLE iot.readings_by_day_temp (
    day         date,
    temperature double,
    ts          timestamp,
    sensor_id   text,
    humidity    double,
    PRIMARY KEY ((day), temperature, ts, sensor_id)
) WITH CLUSTERING ORDER BY (temperature DESC, ts ASC, sensor_id ASC);

SELECT * FROM iot.readings_by_day_temp
WHERE day = '2026-10-22' AND temperature > 31.5;
```

Các thành phần `ts` và `sensor_id` phân biệt số đo có cùng nhiệt độ. Đây là bảng đề xuất trong báo cáo, không phải bảng đã được `sensorring.py` tự động tạo. Ứng dụng cần cập nhật cả bảng theo cảm biến lẫn bảng theo ngày. Nếu lưu lượng trong ngày quá lớn, có thể chia nhỏ partition theo giờ, đồng thời chấp nhận đọc nhiều bucket cho truy vấn cả ngày.

## 4. Khả năng hoạt động khi node ngừng chạy

### Bước 3 và câu hỏi 4 — Số bản sao cần phản hồi

Với RF = 3, lần lượt thử ghi và đọc ở ONE, QUORUM, ALL:

- **Ba node sống:** cả ba mức đều thành công.
- **Tắt cass3, còn hai node:** ONE và QUORUM thành công; ALL báo `Unavailable`, cần 3 replica nhưng chỉ có 2.
- **Tắt tiếp cass2, còn một node:** ONE thành công; QUORUM cần 2 replica và ALL cần 3 nên đều báo `Unavailable`.

Các lệnh được chạy sau khi trạng thái node đã thành `DN`, vì vậy lỗi thiếu replica được trả ngay trong thử nghiệm. Công thức quorum là `floor(RF / 2) + 1`. Với RF = 5, cần ba replica phản hồi; có thể mất hai replica của partition mà vẫn đáp ứng quorum, nếu các replica còn lại liên lạc được.

### Bước 4 — Tái hiện bản sao thiếu dữ liệu

Quy trình trong log gồm các giai đoạn sau:

1. Cho cass1 và cass2 hoạt động, tắt hinted handoff trên hai node này, giữ cass3 ở trạng thái dừng.
2. Ghi số đo hn-99 ở QUORUM, sau đó bật cass3 và dừng cass1, cass2.
3. Đọc ONE từ cass3 nhận `(0 rows)`: node này chưa có lần ghi đã bỏ lỡ.
4. Khi cụm đủ replica trở lại, đọc ALL trả một hàng hn-99 với nhiệt độ **41** và độ ẩm **30**.
5. Chạy `nodetool repair iot` nhận thông báo hoàn tất thành công. Sau các lần khởi động lại, `statushandoff` báo `running`.

Việc đọc từ nhiều replica cho phép đối chiếu phiên bản; read repair có thể sửa dữ liệu liên quan theo cấu hình bảng. Log này cho thấy kết quả đọc ALL và repair thành công, nhưng không có phép đọc cô lập cass3 sau đó để xác nhận riêng trạng thái của node. Phạm vi và hành vi của read repair được mô tả trong [tài liệu Cassandra](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/read_repair.html).

### Câu hỏi 5 — Nếu cả đọc và ghi đều QUORUM

Trong kịch bản ghi đã thành công và không có ghi/xóa cạnh tranh, W = 2 và R = 2 tạo giao nhau vì `R + W = 4 > RF = 3`. Lượt đọc có thể đối chiếu với ít nhất một replica đã xác nhận lần ghi. Nếu chỉ cass3 sống, QUORUM báo thiếu replica thay vì trả kết quả rỗng như ONE. Lập luận này dựa trên giao nhau của tập replica, không có nghĩa QUORUM cung cấp mọi tính chất của giao dịch tuần tự hóa. Xem [kiến trúc sao chép Cassandra](https://cassandra.apache.org/doc/stable/cassandra/architecture/dynamo).

### Câu hỏi 6 — Ưu tiên tiếp nhận dữ liệu IoT

ONE là lựa chọn phù hợp với giả thiết chấp nhận mất một phần số đo để ưu tiên tính sẵn sàng: chỉ cần một replica xác nhận. Đây thường là lựa chọn có ít yêu cầu phản hồi hơn QUORUM hoặc ALL; không thể kết luận luôn có độ trễ thấp nhất trong mọi điều kiện.

ANY còn cho phép xác nhận ghi bằng hint tại coordinator khi replica không khả dụng, nhưng dữ liệu chưa đọc được cho tới khi được chuyển tới replica. Các cơ chế hinted handoff, read repair và repair hỗ trợ hội tụ bản sao; chúng không bảo đảm khôi phục được một lần ghi nếu mọi bản chứa dữ liệu và hint tương ứng đã mất.

## 5. Bài tập thiết kế bảng

### Bài 1 — Đơn hàng theo khách và theo ngày

Hai hướng truy vấn được phục vụ bằng hai bảng riêng:

```sql
CREATE TABLE shop.orders_by_customer (
    customer_id text,
    order_ts    timestamp,
    order_id    int,
    city        text,
    status      text,
    total       double,
    PRIMARY KEY ((customer_id), order_ts, order_id)
) WITH CLUSTERING ORDER BY (order_ts DESC, order_id ASC);

CREATE TABLE shop.orders_by_day (
    day         date,
    order_ts    timestamp,
    order_id    int,
    customer_id text,
    status      text,
    total       double,
    PRIMARY KEY ((day), order_ts, order_id)
) WITH CLUSTERING ORDER BY (order_ts ASC, order_id ASC);
```

Chương trình sinh 40 đơn của ba khách và ghi vào cả hai bảng. Với `orders_by_customer`, truy vấn theo khách Linh kèm `LIMIT 10` lấy mười đơn mới nhất theo thứ tự clustering giảm dần. Hai đơn 1008 và 1035 cùng thời điểm 2026-11-11 08:00 vẫn được giữ riêng nhờ thành phần `order_id` trong khóa.

Với `orders_by_day`, điều kiện ngày 2026-11-03 trả bốn đơn theo thời gian tăng dần. Việc nhân bản dữ liệu phục vụ hai kiểu đọc làm phát sinh trách nhiệm cập nhật nhất quán giữa hai bảng ở phía ứng dụng.

### Bài 2 — Vai trò của partition key

Bộ dữ liệu bổ sung gồm 60 số đo của dn-01, dn-02 và hue-01. Truy vấn xác định `sensor_id = 'dn-01'` lấy dữ liệu bình thường. Hai câu chỉ lọc `temperature > 30` hoặc `ts > '2026-11-10 02:00:00'` đều nhận `InvalidRequest` với thông báo cần `ALLOW FILTERING` do chi phí lọc khó dự đoán.

`ts` là clustering column nhưng thứ tự của nó chỉ hữu ích trong partition tương ứng. Không chỉ định partition key khiến truy vấn không thu hẹp được về một partition như thiết kế ban đầu. Điều này không có nghĩa Cassandra không biết vị trí dữ liệu; vấn đề là điều kiện truy vấn chưa đủ để định tuyến đọc theo khóa.

### Bài 3 — Giới hạn partition bằng bucket tháng

```sql
CREATE TABLE shop.readings_by_sensor_month (
    sensor_id   text,
    month       text,
    ts          timestamp,
    temperature double,
    humidity    double,
    PRIMARY KEY ((sensor_id, month), ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

Với khóa ghép `(sensor_id, month)`, 364 số đo của dn-01, cách nhau sáu giờ, được chia thành ba partition: **tháng 9 có 120, tháng 10 có 124, tháng 11 có 120** hàng.

Nếu chỉ partition theo cảm biến, toàn bộ lịch sử của cảm biến tiếp tục tích lũy trong một partition; ở tần suất một lần mỗi giây, một năm đã có khoảng 31 triệu hàng. Bucket tháng giới hạn phạm vi thời gian, nhưng kích thước thực tế vẫn phụ thuộc tốc độ ghi và độ lớn mỗi hàng. Hệ thống có tần suất cao có thể cần bucket ngày hoặc giờ.

Đổi lại, truy vấn phải cung cấp cả cảm biến và tháng. Truy vấn thiếu `month` bị từ chối trong lần thử. Để đọc từ 2026-10-31 tới trước 2026-11-01 12:00, chương trình dùng hai bucket `2026-10`, `2026-11` và nhận sáu hàng. Không nên suy ra thứ tự toàn cục giữa nhiều partition chỉ từ thứ tự clustering bên trong từng partition.
