# Báo cáo Day 13 — Tạ Xuân Anh Vũ, 2A202603173

## 1. Kết luận trước
Mã số: 2A202603173, task `D13-2A202603173`, danh sách 10 frame: `s01f03`, `s01f42`, `s02f01`, `s02f28`, `s03f10`, `s03f34`, `s04f31`, `s04f47`, `s05f00`, `s05f43`.
Hệ điều hành: Windows, trình duyệt: Chrome, phiên bản CVAT: 2.74.1. Bạn ngồi cạnh đổi kiểm: 2A202600123.
Tổng số cuboid vẽ được là 66 cuboid; frame ít nhất là `s02f01` và `s03f34` (mỗi frame 5 cuboid) do cảnh có mật độ phương tiện thưa hơn và nhiều vùng bị che chắn; công cụ `selfcheck` ghi nhận 2/66 cuboid còn cờ cảnh báo. Bản thân tự tin nhất ở phần xác định class, căn chỉnh đáy cuboid tiếp xúc mặt đường và yaw của các phương tiện trong phạm vi 30 m; phần còn cần lưu ý thêm là ước lượng chiều dài các xe ở xa trên 40 m do mật độ điểm bị thưa.

Cách dùng CVAT:
- Đổi góc nhìn: Kéo chuột trái để xoay 3D (xoay nhìn thẳng từ trên xuống để quan sát BEV), kéo chuột phải để tịnh tiến/dời khung nhìn, lăn con lăn chuột để phóng to thu nhỏ; nháy đúp vùng trống để reset góc nhìn mặc định.
- Vẽ và chỉnh cuboid: Chọn class ở thanh công cụ bên trái, bấm Shape rồi nháy đúp chuột vào tâm cụm điểm để đặt hộp; dùng khung Top để dịch tâm, chỉnh dài/rộng và xoay góc yaw; dùng khung Side và Front kéo cạnh đáy chạm mặt đường và chỉnh chiều cao.
- Bật mũi tên hướng và xem trước: Vào menu Appearance bật "Cuboid orientation" để hiện vạch đỏ ngắn (+X của cuboid) chỉ về đầu xe; sau khi lưu dùng lệnh `python scripts/lab.py preview` để kiểm tra độ khớp của cuboid trên 7 ảnh camera.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ. Ảnh: `t1-s01f03-3d`, `t1-s01f03-top`, `t1-s01f42-3d`, `t1-s01f42-top`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| s01f03 | Phía trước (−y) ở cạnh trên; +x sang trái, +y xuống dưới | Mặt đường phẳng, thấp hơn LiDAR 1.62 m | 49 m | Xe con, xe tải, xe hai bánh, biển báo, cây cối, dải phân cách | Xe con phía sau (cam09), người đi xe máy bên trái (cam05) | Cụm điểm thưa ở rìa xa bên trái |
| s01f42 | Phía trước (−y) ở cạnh trên; +x sang trái, +y xuống dưới | Mặt đường phẳng, thấp hơn LiDAR 1.75 m | 56 m | Xe con, xe hai bánh, cột đèn, cây cối, công trình | Xe con đi phía sau, xe máy bên sườn phải | Cụm điểm lẫn vào tán cây ở xa |

3 điều chỉ camera cho biết:
- Màu sắc ngoại thất của phương tiện và chi tiết biển báo, vạch kẻ đường.
- Trạng thái đèn tín hiệu giao thông (xanh/đỏ/vàng) và đèn phanh/xi-nhan của xe.
- Chi tiết hình thái người đi bộ/trang phục hoặc loại hàng hóa chuyên dụng chở trên xe.

3 điều chỉ LiDAR cho biết:
- Tọa độ 3D và khoảng cách hình học chính xác tính bằng mét đến tâm vật thể.
- Kích thước thật (dài × rộng × cao) tính theo mét độc lập với phối cảnh xa gần.
- Phát hiện vật thể ở góc mù giữa các camera hoặc trong điều kiện ánh sáng chói/ngược sáng.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| s02f01 | Cạnh trên | Xe buýt lớn che khuất góc nhìn bên trái |
| s02f28 | Cạnh trên | Không có |
| s03f10 | Cạnh trên | Mật độ xe con đông ở làn đối diện |
| s03f34 | Cạnh trên | Người đi bộ qua đường ở khoảng cách gần |
| s04f31 | Cạnh trên | Không có |
| s04f47 | Cạnh trên | Xe máy đi sát mép dải phân cách |
| s05f00 | Cạnh trên | Không có |
| s05f43 | Cạnh trên | Cụm xe tải ở khoảng cách xa trên 50 m |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car`, `t2-b-bus-truck`, `t2-c-ped-cyclist`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | s01f03, 1 | Car | Đúng (vạch đỏ chỉ về đầu xe, yaw ≈ 265.6°) | Đáy chạm mặt đường (z_bottom = -1.749 m) | Đã kiểm đủ 3 view và đối chiếu camera trước | Dài 5.11 m, rộng 1.77 m, cao 1.53 m (nằm trong chuẩn bảng 4.3) |
| (b) xe buýt hoặc xe tải | s01f03, 4 | Truck | Đúng (vạch đỏ chỉ về đầu xe, yaw ≈ 266.7°) | Đáy chạm mặt đường (z_bottom = -1.643 m) | Đã kiểm đủ 3 view và đối chiếu camera | Dài 4.67 m, rộng 1.96 m, cao 2.00 m (phù hợp xe tải nhẹ/thùng) |
| (c) người đi bộ hoặc xe hai bánh | s01f03, 5 | Cyclist | Đúng (vạch đỏ chỉ theo chiều di chuyển, yaw ≈ 263.0°) | Đáy chạm mặt đường (z_bottom = -1.587 m) | Đã kiểm đủ 3 view và đối chiếu camera | Dài 1.76 m, rộng 0.84 m, cao 1.60 m (nằm trong chuẩn bảng 4.3) |

Vòng hiệu chuẩn: Khi so sánh với các bạn trong lớp, cuboid có độ lệch nhỏ về góc yaw (lệch khoảng 3°–5°) và độ cao đáy chênh lệch khoảng 0.05 m; cả lớp đã thống nhất cách hiểu guideline là ưu tiên căn đáy hộp sát mặt phẳng đường thật tại Side/Front và vạch đỏ luôn hướng theo chiều di chuyển của xe. Các vật bỏ qua vì không đủ cơ sở: ở frame `s05f43` có 1 cụm điểm thưa ở x ≈ 35 m, y ≈ -48 m với số điểm < 10 không rõ ranh giới, đánh dấu nghi vấn `LOW_VISIBILITY` và không gán nhãn.

## 4. Bốn hiện tượng chất lượng (Bước 4)
Nguyên nhân ghi là **giả thuyết**.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `t4-1-bong-che.png`, nguồn: task chung `D13-HC485087` (s02f00) | Tia LiDAR truyền thẳng bị thân xe buýt lớn kích thước lớn che chắn toàn bộ tầm quét phía sau | Một vệt hình nón/quạt trống trơn hoàn toàn không có điểm LiDAR phía sau thân xe buýt, dù mặt đường hai bên vẫn phủ điểm dày | Không được coi vùng trống là đường trống; nếu có vật thể bị che khuất một phần phía sau thì bật thuộc tính `occluded` và ước lượng cuboid bao trọn phần suy luận được |
| 2. Thưa ở xa | `t4-2-thua-o-xa.png`, nguồn: frame `s05f43` của bài | Khoảng cách góc giữa các tia LiDAR mở rộng theo cự ly xa làm giảm mật độ tia phản xạ trở lại | Phương tiện ở cự ly khoảng 45–50 m chỉ nhận được vài chục điểm rời rạc tạo thành viền mỏng | Áp dụng quy tắc 15–20% thể tích: nếu nhận diện được hình khối thì vẽ và đối chiếu kích thước bảng 4.3; nếu quá thưa không đủ căn cứ thì bỏ qua và ghi `LOW_VISIBILITY` |
| 3. Phản xạ thấp / ít điểm bất thường | `t4-3-phan-xa-thap.png`, nguồn: frame `s01f03` của bài | Bề mặt xe tối màu hoặc bề mặt kính hấp thụ/tán xạ chệch hướng chùm tia laser của LiDAR | Cụm xe ô tô có số lượng điểm ít hơn hẳn và intensity thấp hơn so với các vật thể cùng cự ly | Phải đối chiếu kỹ với camera để nhận diện đúng loại xe, vẽ hộp cẩn thận theo hình dáng thực tế và ghi chú giả thuyết phản xạ thấp |
| 4. Điểm lạ | `t4-4-diem-la.png`, nguồn: task chung `D13-HC485087` (s02f35) | Hiện tượng tán xạ nhiều lần (multipath) qua kính/gương hoặc mặt đường ướt phản chiếu tia laser | Cụm khoảng 46 điểm nằm sâu dưới lòng đất ở độ cao z ≈ -13.5 m (thấp hơn mặt đường khoảng 11 m) | Tuyệt đối không gán các điểm sai hình học này thành vật thể, không kéo đáy hộp cuboid xuống bao chùm các điểm này |

## 5. Số từ cuboid của tôi (Bước 5)
Lấy từ `submission/D13-2A202603173-bangA.csv` và `submission/D13-2A202603173-tukiem.csv`. Mặt đường khoảng -1.7 m:

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| s01f03, 1, Car | -0.982 m | 1.533 m | -1.749 m | 9.16 m | 9.21 m | 265.6° | Chạm đường (đáy sát mặt phẳng đường -1.75 m) |
| s01f03, 4, Truck | -0.641 m | 2.004 m | -1.643 m | 20.61 m | 20.62 m | 266.7° | Chạm đường (đáy sát mặt đường khoảng -1.65 m) |
| s01f03, 5, Cyclist | -0.786 m | 1.603 m | -1.587 m | 15.57 m | 15.59 m | 263.0° | Chạm đường (tiếp xúc mặt đường) |

(a) Cuboid gần nhất và xa nhất:
- Cuboid gần nhất: frame `s03f10`, idx 5 (`Car`), có `dist_m = 4.03 m` và khoảng cách không gian 3D `r = 4.16 m`.
- Cuboid xa nhất: frame `s05f43`, idx 8 (`Truck`), có `dist_m = 51.85 m` và khoảng cách không gian 3D `r = 51.86 m`.

(b) Kích thước dài, rộng, cao so với bảng 4.3:
- Xe con (`s01f03`, idx 1): kích thước dài 5.11 m, rộng 1.77 m, cao 1.53 m. So với bảng 4.3 (sedan dài 3.8–5.0 m, SUV dài 4.4–5.3 m), kích thước này hoàn toàn phù hợp với dòng xe sedan cỡ lớn/SUV đô thị.
- Xe hai bánh/Cyclist (`s01f03`, idx 5): kích thước dài 1.76 m, rộng 0.84 m, cao 1.60 m. So với bảng 4.3 (xe máy có người dài 1.8–2.3 m, rộng 0.7–0.95 m, cao 1.4–1.8 m), chiều dài hơi ngắn hơn cận dưới một chút do người lái ngồi co chân và phần đầu xe khuất điểm LiDAR.

(c) Góc yaw của xe cùng chiều và ngược chiều:
- Xe cùng chiều phía trước (`s01f03`, idx 1): có `yaw_deg = 265.6°` (rất gần giá trị lý tưởng 270°).
- Xe đi ngược chiều (`s01f03`, idx 6): có `yaw_deg = 87.7°` (rất gần giá trị lý tưởng 90°).

Mật độ theo khoảng cách (4 cuboid ở các cự ly khác nhau):

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| s03f10, 5, Car | 4.03 m | 2379 điểm |
| s03f34, 3, Pedestrian | 10.71 m | 256 điểm |
| s01f03, 4, Truck | 20.61 m | 475 điểm |
| s02f01, 5, Car | 30.01 m | 132 điểm |

Quy luật mật độ điểm: Khi khoảng cách tăng gấp đôi thì mật độ điểm LiDAR suy giảm xấp xỉ 3 đến 4 lần theo định luật nghịch đảo bình phương khoảng cách, hoàn toàn khớp với đường cong phân bố trên slide bài giảng (xe con ở 0–10 m đạt trên 1900 điểm, giảm dần xuống khoảng vài trăm điểm ở 10–20 m và chỉ còn trên 100 điểm ở cự ly 30 m).
Tự kiểm và điều chỉnh: Sau khi chạy `preview` và `selfcheck`, đã rà soát lại 2 cuboid bị cờ cảnh báo (ở `s04f47`#6 do tỷ lệ dài rộng của xe máy và ở `s05f43`#8 xe tải ở xa mép 50 m); đã giữ nguyên kích thước sau khi đối chiếu ảnh camera vì phản ánh đúng thực tế hình học của phương tiện trong cảnh.

Đổi kiểm chéo: Đã đổi kiểm chéo số liệu với bạn có MSSV 2A202600123; đối chiếu 2 con số gồm khoảng cách cuboid gần nhất (4.03 m) và yaw của xe ngược chiều (87.7°), kết quả hoàn toàn trùng khớp với dữ liệu xuất ra từ Bảng A.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
(a) Ý nghĩa từ góc nhìn mô hình học máy:
- Nhãn 3D phải đo bằng mét trong hệ tọa độ thế giới thực chứ không dùng pixel vì mô hình tự hành cần khoảng cách vật lý chuẩn xác để tính toán quỹ đạo, vận tốc và lập kế hoạch phanh/tránh va chạm an toàn trong không gian 3 chiều.
- Vật thưa điểm ở xa cần quy tắc riêng (như ngưỡng nhận diện 15–20% thể tích) vì nếu gán nhãn tùy tiện khi không đủ điểm phản xạ, mô hình sẽ học phải nhiễu (false positive) hoặc học sai kích thước biên hộp; ngược lại nếu bỏ sót vật nhận biết được thì mô hình sẽ bị giảm độ phủ an toàn.

(b) Cách dùng AI:
- Công cụ: Antigravity Assistant.
- Câu hỏi chính: Hướng dẫn quy trình nộp bài, giải thích hệ tọa độ VF (x trái, y sau, z lên) và các phím tắt công cụ CVAT 3D.
- Kết quả: Nắm vững quy trình làm việc, tự thực hiện gán nhãn 66 cuboid trên CVAT và chạy các công cụ kiểm tra nộp bài theo chuẩn.

(c) Điều còn chưa chắc:
- Một vài vật thể ở cự ly trên 45 m có mật độ điểm dưới 15 điểm nên việc xác định chính xác mép đuôi xe còn phụ thuộc vào suy luận hình học.
