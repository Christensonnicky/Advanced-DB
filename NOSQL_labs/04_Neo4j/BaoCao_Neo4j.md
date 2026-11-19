# Neo4j — Mô hình quan hệ học tập và các ứng dụng đồ thị

## 1. Phạm vi và minh chứng

CampusGraph biểu diễn môn học, sinh viên, quan hệ tiên quyết, kết quả học và tình bạn bằng đồ thị. Môi trường dùng Neo4j 5 theo `compose.yaml` cùng driver Python `neo4j`.

- `campus.py` triển khai bảy task; `check_output.txt` lưu kết quả kiểm tra.
- `report_queries.py` và `report_output.txt` chứa các thử nghiệm cho phần câu hỏi.
- `store_exercises.py` và `exercises_output.txt` dành cho bài tập cửa hàng, metro và lịch sử công việc.
- `screenshots/check_pass.png` là ảnh chụp kết quả kiểm tra đã có trong bộ tài liệu.

Sau khi nạp 13 môn và 10 sinh viên, log ghi nhận đủ tám mục kiểm tra đạt, trong đó task 7 có hai tình huống từ chối và ghi thành công:

```
loaded: {'Course': 13, 'Student': 10}
PASS  1 passed_courses
PASS  2 prerequisite_chain
PASS  3 eligible_courses
PASS  4 suggest_friends
PASS  5 connection
PASS  6 bottleneck_courses
PASS  7a record_pass refuses
PASS  7b record_pass writes
```

Các kết quả dưới đây được diễn giải từ mã và minh chứng đã lưu; phần biên tập không tạo ra một lần chạy cơ sở dữ liệu mới.

## 2. Từ yêu cầu nghiệp vụ đến truy vấn CampusGraph

### Tra cứu kết quả, điều kiện học và mức phụ thuộc

Task 1 đọc quan hệ `PASSED`, sắp điểm giảm dần rồi theo mã môn. Task 2 duyệt một hoặc nhiều cạnh `REQUIRES` để tìm cả tiên quyết trực tiếp và gián tiếp. Task 3 chọn môn chưa qua và không có tiên quyết trực tiếp nào còn thiếu. Task 6 đếm số môn phân biệt phụ thuộc vào từng môn.

```cypher
// Task 2
MATCH path = (:Course {code: $code})-[:REQUIRES*1..]->(p:Course)
RETURN p.code AS code, min(length(path)) AS depth
ORDER BY depth, code

// Task 3
MATCH (s:Student {id: $sid}), (c:Course)
WHERE NOT (s)-[:PASSED]->(c)
  AND NOT EXISTS {
      MATCH (c)-[:REQUIRES]->(p:Course)
      WHERE NOT (s)-[:PASSED]->(p)
  }
RETURN c.code AS code ORDER BY code

// Task 6
MATCH (d:Course)-[:REQUIRES*1..]->(c:Course)
RETURN c.code AS code, count(DISTINCT d) AS dependents
ORDER BY dependents DESC, code LIMIT $k
```

Với dữ liệu kiểm tra, `eligible_courses('S04')` trả **CS102, MATH102, STAT101**. Trong năm môn có nhiều môn khác phụ thuộc nhất, kết quả là **CS101: 9; CS102: 7; MATH101: 7; STAT101: 5; MATH102: 4**.

### Kết nối và gợi ý bạn học

Task 4 xếp ứng viên chưa là bạn theo số môn đã qua chung, giảm dần theo số môn rồi tăng dần theo tên. Với S02, kết quả là **Minh 4, Phuong 4, Chi 3**; với S08 là **An 1, Binh 1, Chi 1**.

Task 5 tìm đường ngắn nhất theo số cạnh tình bạn, không xét hướng lưu. Đường từ S03 tới S05 trong log là **Chi → Minh → Phuong → Khoa → Giang**; từ S01 tới S08 là **An → Binh → Dung → Lan**.

### Ghi kết quả học

Task 7 dùng một `execute_write`, trong đó ba lần `tx.run` lần lượt kiểm tra sinh viên/môn tồn tại, xác định tiên quyết còn thiếu và ghi điểm bằng `MERGE ... SET`. Đầu vào sinh viên S99 hoặc môn XX999 gây `ValueError`. Khi S08 được ghi CS102 hai lần với điểm 7.0 rồi 7.5, kiểm tra nhận một quan hệ với điểm cuối 7.5. Kết quả này xác nhận hành vi cập nhật trong chuỗi gọi đã thử.

## 3. Giải thích thiết kế và kết quả khảo sát

### Câu hỏi 1 — Biểu diễn cùng bài toán bằng SQL

Mô hình quan hệ dùng bảng thực thể `course`, `student` cùng các bảng nối `requires`, `passed`, `friend`. Việc duyệt tiên quyết nhiều cấp được viết bằng CTE đệ quy:

```sql
CREATE TABLE course  (code TEXT PRIMARY KEY, title TEXT, credits INT);
CREATE TABLE student (id TEXT PRIMARY KEY, name TEXT, year INT);
CREATE TABLE requires (
    course_code TEXT REFERENCES course(code),
    prereq_code TEXT REFERENCES course(code),
    PRIMARY KEY (course_code, prereq_code));
CREATE TABLE passed (
    student_id  TEXT REFERENCES student(id),
    course_code TEXT REFERENCES course(code),
    grade REAL,
    PRIMARY KEY (student_id, course_code));
CREATE TABLE friend (
    a_id TEXT REFERENCES student(id),
    b_id TEXT REFERENCES student(id),
    PRIMARY KEY (a_id, b_id));

WITH RECURSIVE chain(code, depth) AS (
    SELECT prereq_code, 1 FROM requires WHERE course_code = :code
    UNION ALL
    SELECT r.prereq_code, c.depth + 1
    FROM chain c JOIN requires r ON r.course_code = c.code
)
SELECT code, MIN(depth) AS depth
FROM chain
GROUP BY code
ORDER BY depth, code;
```

Phần neo lấy tiên quyết trực tiếp; phần đệ quy tiếp tục nối tới các tiên quyết kế tiếp. Cuối cùng nhóm theo mã môn và lấy độ sâu nhỏ nhất. `report_queries.py` thực hiện truy vấn trên SQLite cho DE301 và CAP401, thu được cùng kết quả với Cypher.

Cypher diễn tả đường đi trực tiếp bằng mẫu quan hệ, còn SQL thể hiện quá trình duyệt qua phép nối lặp. Mẫu CTE này phù hợp đồ thị tiên quyết không có chu trình của bài; nếu dữ liệu cho phép chu trình, cần thêm điều kiện ngăn đệ quy lặp vô hạn.

### Câu hỏi 2 — Nhiều đường tới cùng một môn

`MATCH path` sinh kết quả theo từng đường đi. Trong `RETURN p.code, min(length(path))`, mã môn đóng vai trò khóa nhóm nên mỗi môn chỉ còn một hàng và nhận độ sâu nhỏ nhất. Với CAP401, 11 đường được gom thành tám môn.

Hai đường từ CAP401 đến CS101 là:

```
CAP401 -> ML201 -> CS102 -> CS101              (3 bước)
CAP401 -> DB301 -> DB201 -> CS102 -> CS101     (4 bước)
```

Vì hai độ dài là 3 và 4, kết quả của CS101 có `depth = 3`. Toàn bộ danh sách cho CAP401 là DB301 và ML201 ở độ sâu 1; CS102, DB201, MATH102, STAT101 ở độ sâu 2; CS101 và MATH101 ở độ sâu 3.

### Câu hỏi 3 — Tác động của constraint trên Student.id

PROFILE task 3 với S01 ghi nhận **194 db hits** khi có constraint `student_id`, sử dụng `NodeUniqueIndexSeek` để tìm sinh viên. Khi bỏ constraint, kế hoạch dùng `NodeByLabelScan` rồi `Filter`, tổng cộng **213 db hits**.

Chênh lệch 19 db hits xuất hiện trên tập chỉ có mười sinh viên. Constraint tạo chỉ mục hỗ trợ tìm theo ID nên tránh quét toàn bộ nhãn ở bước này. Tổng chi phí còn phụ thuộc số môn, quan hệ và lựa chọn kế hoạch; không thể suy ra chính xác db hits cho 100.000 sinh viên khi chưa đo trên bộ dữ liệu đó.

### Câu hỏi 4 — Lưu một hay hai chiều FRIEND

Một cạnh có hướng lưu trữ đã đủ biểu diễn một cặp bạn nếu quy ước tình bạn đối xứng và luôn truy vấn bằng `-[:FRIEND]-`. Cách này giảm dữ liệu lặp và tránh phải đồng bộ hai cạnh ngược chiều.

Thử thêm cạnh ngược cho 11 cặp làm tổng số cạnh thành 22. Hai hàm hiện tại vẫn cho kết quả đã nêu: task 4 kiểm tra tồn tại tình bạn và đếm môn chung bằng `count(DISTINCT c)`; task 5 tìm đường ngắn nhất. Tuy vậy, truy vấn liệt kê bạn không hướng của An trả **Binh, Giang, Binh, Giang**.

Nếu dùng đường đi qua bạn chung rồi `count(*)`, cạnh lặp có thể nhân số hàng. Mức nhân tùy số bước và cấu trúc cạnh, không mặc định chỉ gấp đôi. `allShortestPaths` cũng có thể liệt kê nhiều đường khác nhau về cạnh nhưng cùng chuỗi người. Lưu hai chiều làm tăng dữ liệu cần xử lý, song không đủ cơ sở kết luận mọi truy vấn sẽ tốn đúng gấp đôi chi phí.

### Câu hỏi 5 — Mô hình hóa lần học lại

Quan hệ `PASSED` hiện tại phù hợp khi chỉ cần một kết quả tổng hợp cho mỗi cặp sinh viên–môn. Khi mỗi lần học cần có định danh và liên kết riêng, có thể thêm thực thể Enrollment:

```
(:Student)-[:TOOK]->(:Enrollment {semester, grade, status})-[:OF]->(:Course)
```

Mỗi node lưu học kỳ, điểm và trạng thái; nếu cần, nối tiếp tới giảng viên hoặc lớp. Lưu thuộc tính học kỳ trên nhiều quan hệ cũng là một phương án, nhưng phải thay cách định danh và `MERGE` hiện tại. Ưu điểm của node trung gian là có thể trực tiếp tham gia các quan hệ khác.

Để lấy điểm cao nhất trong những lần đã đạt:

```cypher
MATCH (:Student {id: $sid})-[:TOOK]->(e:Enrollment {status: 'passed'})-[:OF]->(c:Course)
WITH c.code AS code, max(e.grade) AS grade
RETURN code, grade
ORDER BY grade DESC, code
```

Thử nghiệm bổ sung cho S04 hai lần học CS102: **3.5, failed, 2025-1** và **7.0, passed, 2025-2**. Kết quả tổng hợp là **MATH101 7.5, CS102 7.0, CS101 6.5**, đồng thời vẫn truy xuất được lịch sử hai lần học. Khi đổi mô hình, task 3 và 7 cần kiểm tra đường đi qua Enrollment có `status: 'passed'` thay cho cạnh PASSED trực tiếp.

### Câu hỏi 6 — Kiểm tra và ghi khi có cập nhật đồng thời

Nếu kiểm tra và ghi nằm trong hai `execute_query` riêng, có thể xảy ra: lần kiểm tra thấy S08 đã qua CS101; một transaction khác xóa kết quả CS101; sau đó lệnh ghi vẫn thêm kết quả CS102. Điều kiện nghiệp vụ đã thay đổi giữa hai thao tác.

Một `execute_write` gom các thao tác vào cùng transaction để thành công hoặc rollback cùng nhau. Tuy nhiên, Neo4j mặc định dùng read committed; chỉ gộp transaction, hoặc chỉ gộp thành một câu Cypher, không tự bảo đảm ngăn mọi thay đổi cạnh tranh trên dữ liệu đã đọc. Cần thiết kế giao thức khóa chung cho các luồng sửa liên quan và kiểm tra lại điều kiện sau khi lấy khóa. Nội dung này dựa trên [tài liệu truy cập đồng thời Neo4j](https://neo4j.com/docs/operations-manual/current/database-internals/concurrent-data-access/).

Mã lab chưa thực hiện kiểm thử xóa đồng thời, nên tám mục PASS không phải bằng chứng về an toàn trong tình huống đó. Ngoài ra, nếu yêu cầu dữ liệu luôn giữ tiên quyết sau khi ghi, luồng xóa cũng phải kiểm tra các kết quả phụ thuộc.

## 4. Bài tập ứng dụng

### Bài 1 — Tìm bạn của bạn tại Hanoi

Đồ thị cửa hàng bắt đầu với Linh, Nam, Mai, Hung và ba cuốn sách. Chương trình bổ sung Tuan, Vy, Quang, Thao, Duc cùng sáu cạnh FRIEND. Bạn trực tiếp của Linh là Nam và Tuan.

```cypher
MATCH (me:Person {name: "Linh"})-[:FRIEND]-(:Person)-[:FRIEND]-(fof:Person)
WHERE fof <> me AND NOT (me)-[:FRIEND]-(fof) AND fof.city = "Hanoi"
RETURN DISTINCT fof.name
```

Kết quả là **Duc, Mai, Quang**. Điều kiện loại chính Linh tránh đường quay lại điểm xuất phát; điều kiện không có cạnh trực tiếp loại người đã là bạn. `DISTINCT` hợp nhất người xuất hiện qua nhiều bạn chung.

### Bài 2 — Gợi ý sách và thống kê chi tiêu

Gợi ý từ sách bạn bè đã mua nhưng Mai chưa mua trả **SQL in Practice**, do Nam đã mua. Cách gợi ý dựa trên những người mua chung sách cũng trả cùng tựa, với hai người liên quan là Linh và Nam.

Tổng tiền của Nam là **285.000**, Linh **270.000**, Mai **255.000**; mỗi người mua hai cuốn. Sáu người còn lại có số sách và số tiền bằng 0. `OPTIONAL MATCH` giữ các người chưa mua trong kết quả, thay vì loại họ khi không có cạnh BOUGHT.

### Bài 3 — Đường đi trên mạng metro mô phỏng

Mô hình gồm 15 ga và 15 cạnh NEXT, có thuộc tính `minutes`, `line`. Tên ga lấy cảm hứng từ Hà Nội; các tuyến, kết nối và thời gian trong dữ liệu phục vụ mô phỏng, không dùng như sơ đồ vận hành thực tế. Ba tuyến trong bài là 2A, 3 và tuyến giả định X qua Cầu Giấy – Nguyễn Trãi – Văn Quán.

```cypher
MATCH (a:Station {name: $a}), (b:Station {name: $b}),
      p = shortestPath((a)-[:NEXT*..30]-(b))
RETURN [s IN nodes(p) | s.name] AS route, length(p) AS hops,
       reduce(t = 0, r IN relationships(p) | t + r.minutes) AS minutes
```

Kết quả đã lưu:

- **Cát Linh → Yên Nghĩa:** 8 đoạn, 31 phút; qua La Thành, Thái Hà, Láng, Cầu Giấy, Nguyễn Trãi, Văn Quán, Hà Đông.
- **Nhổn → Hà Đông:** 4 đoạn, 25 phút; qua Cầu Giấy, Nguyễn Trãi, Văn Quán.
- **Ga Hà Nội → Văn Quán:** 5 đoạn, 24 phút; qua Kim Mã, Láng, Cầu Giấy, Nguyễn Trãi.

Truy vấn bỏ hướng vì giả định mỗi đoạn đi được hai chiều. `shortestPath` tối thiểu hóa **số cạnh**, không tối thiểu hóa tổng `minutes`. Bởi vậy đường Cát Linh – Yên Nghĩa được chọn có tám đoạn nhưng 31 phút, trong khi đi theo chuỗi tuyến 2A có chín đoạn và 24 phút. Nếu mục tiêu là thời gian, cần thuật toán đường đi có trọng số, chẳng hạn Dijkstra.

### Bài 4 — Hai cách lưu lịch sử công việc

Phương án thứ nhất gắn `title`, `start`, `end` vào cạnh WORKED_AT nối Person với Company. Cách này phục vụ trực tiếp câu hỏi đang làm ở đâu và làm trong thời gian nào. Truy vấn việc hiện tại ở Viettel trả Mai và Linh; đối chiếu khoảng thời gian ở FPT tìm được Nam từng làm cùng lúc với Linh.

Phương án thứ hai dùng đường `Person → HELD → Job → AT → Company`. Job trở thành thực thể để nối thêm PROMOTED_TO hoặc REPORTS_TO. Trong dữ liệu, Linh chuyển từ Intern sang Data Analyst tại FPT; công việc Developer của Nam nối tới người quản lý Hung.

Neo4j không cho một relationship nối trực tiếp vào relationship khác, nên node Job thuận tiện khi cần mô hình hóa các liên kết của chính công việc. Cả hai cách đều có thể lưu nhiều vị trí ở cùng công ty; điểm phân biệt chính là khả năng gắn quan hệ vào từng vị trí, không phải việc có lưu được lịch sử hay không.

### Bài 5 — Chỉ mục duy nhất cho Person.name

Truy vấn khảo sát là:

```
MATCH (p:Person {name: "Linh"}) RETURN p
```

Trước constraint, kế hoạch đi qua `NodeByLabelScan` với chín node, `Filter`, rồi `ProduceResults`; tổng **23 db hits**. Sau khi tạo uniqueness constraint, kế hoạch dùng `NodeUniqueIndexSeek` lấy một node rồi trả kết quả, tổng **5 db hits**.

Thử tạo thêm Person tên Linh nhận `ConstraintError`. Constraint vừa hỗ trợ truy vấn qua chỉ mục vừa thực thi yêu cầu tên duy nhất trong bài; nếu triển khai dữ liệu người thật, cần cân nhắc định danh riêng vì nhiều người có thể trùng tên.
