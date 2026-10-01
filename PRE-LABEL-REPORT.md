# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: K4-DAY13-Nhom-Manh-Son-Anh
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Đình Mạnh; 2026-10-01 15:03:37 UTC+7; Windows 11 x86_64, Docker Engine 29.7.2 Linux/amd64
- Image tag và image ID; phiên bản repo: Image tag: `day13-pointpillars:lc-20261001-amd64`, Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`, Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1` (dirty=True)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (frame `demo`, 17238 điểm), SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (x: 0..70.4m, y: -40..40m, z: -3..1m); score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: Reflectance thật bị lược bỏ (placeholder RGB=0), adapter dùng kênh hằng số; `z_ground` ước lượng từ PCD = 0.075 m.

### Danh sách thành viên và phân công vai trò (TEAMMATES)

| Họ và tên | MSSV | Vai trò lượt A | Vai trò lượt B | Vai trò lượt C |
| --- | --- | --- | --- | --- |
| Nguyễn Đình Mạnh | 02306 | Kiểm tra cấu hình & JSON | Vận hành Docker runner | Xem hình học Side view |
| Phạm Thanh Sơn | 02274 | Ghi log & tổng hợp | Đọc JSON & so sánh số liệu | Vận hành Docker runner |
| Đỗ Tiến Anh | 02252 | Xem hình học & ảnh Side | Ghi log & đối chiếu | Đọc JSON & kiểm tra class |

---

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `boxes-demo-delta-0-voxel-0.16.json` / `side-demo-delta-0-voxel-0.16.png` / `summary.csv` | Chỉ phát hiện 1 xe (`vehicles`), score thấp 0.322, mean_z=0.330 m. Các xe khác bị chìm dưới mặt đất giả định của model nên bị bỏ sót (miss) toàn bộ. |
| B | 1.73 | 0.16 | 13 | 1.034 | `boxes-demo-delta-1.73-voxel-0.16.json` / `side-demo-delta-1.73-voxel-0.16.png` / `summary.csv` | Phát hiện 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian), score cao (nhiều xe đạt 0.8 - 0.933), mean_z=1.034 m. Hộp bám sát cụm điểm và mặt đường cục bộ. Đây là baseline chuẩn. |
| C | 1.73 | 0.32 | 6 | 1.091 | `boxes-demo-delta-1.73-voxel-0.32.json` / `side-demo-delta-1.73-voxel-0.32.png` / `summary.csv` | Số hộp giảm xuống 6, mean_z=1.091 m. Đặc biệt toàn bộ 6 hộp đều bị phân loại là `pedestrian` (mất hoàn toàn nhãn `vehicles`), do pillar quá lớn (0.32m) làm giảm độ phân giải không gian nghiêm trọng. |

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  - **Khác biệt hoàn toàn.** Đổi delta trước inference làm thay đổi cao độ z của toàn bộ đám mây điểm trước khi đưa vào quá trình voxel hóa/pillarization. Khi tọa độ z dịch chuyển, các điểm rơi vào các pillar/grid khác nhau, cấu trúc đặc trưng chiều cao biến dạng so với phân bố mà mô hình PointPillars (huấn luyện trên KITTI với cảm biến đặt cao ~1.73m) kỳ vọng. Mạng trích xuất đặc trưng khác hẳn nên số lượng hộp phát hiện thay đổi mạnh (từ 13 hộp ở lượt B giảm xuống chỉ còn 1 hộp ở lượt A), confidence score cũng thay đổi, chứ không phải là lấy nguyên vẹn 13 hộp của B rồi cộng/trừ 1.73m.
- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  - Khi tăng kích thước pillar XY từ 0.16m lên 0.32m (gấp đôi cạnh, diện tích gấp 4 lần), độ phân giải biểu diễn giảm sút nghiêm trọng. Mô hình mất hoàn toàn khả năng nhận diện lớp `vehicles` (10 xe ở B biến mất ở C) và gán nhầm 6 cụm điểm thành `pedestrian`. Điều này cho thấy pillar 0.32m quá thô đối với checkpoint này. Tuy nhiên, trong đánh giá pipeline, **không thể kết luận cấu hình nào tốt hơn chỉ vì số lượng hộp nhiều hơn hay confidence cao hơn**, mà phải đối chiếu hình học cụm điểm và tính đúng đắn của class. Ở đây, cấu hình B tốt hơn rõ rệt vì phản ánh đúng các phương tiện quan sát được trên mặt đường.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - Giới hạn ROI chỉ lấy cửa sổ phía trước (front-window: x từ 0 đến 70.4m), các vật thể nằm ngoài vùng này không được coi là model bỏ sót (miss). Ảnh Side là hình chiếu phẳng 2D toàn cảnh trên mặt phẳng x-z, do đó các vật thể có cùng khoảng cách x và cao độ z nhưng khác tọa độ y (nằm ở các làn xe song song khác nhau) sẽ bị đè chồng lên nhau trên ảnh. Vì vậy, không thể chỉ nhìn ảnh Side để xác định góc xoay/hướng (yaw) hoặc kết luận kích thước hộp, mà bắt buộc phải kết hợp góc nhìn từ trên xuống (Top/BEV), góc nhìn trực diện (Front) và ảnh camera.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - Cả JSON của lượt A (chỉ 1 hộp, miss gần hết) và lượt C (chỉ có pedestrian, sai toàn bộ class) đều không đủ cơ sở để sử dụng. Riêng JSON của lượt B (13 hộp) dù là baseline tốt nhất vẫn **chưa đủ cơ sở để import trực tiếp vào CVAT Robotaxi** vì: đây là checkpoint KITTI chạy trên một frame minh họa khác domain Robotaxi, chưa có nhãn `Animal` và `Obstacle`, chưa được kiểm tra các góc xoay tự do và đối chiếu ảnh camera để xác minh hướng đầu xe (yaw) và ranh giới bao thân xe.

---

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm tra từng hộp bình thường | Giữ nguyên dự đoán gốc của lượt B, 13 hộp bám sát cụm điểm xe trên mặt đường ở ảnh Side (`side-correct.png`), mean_z = 1.034 m. |
| case-batch-z | 13 / 13 (100%) | Lệch -1.805 m (bằng delta 1.73 + z_ground 0.075) | Không đổi (class, x, y, yaw giữ nguyên 100%) | DỪNG BATCH, báo LC kiểm tra pipeline | Toàn bộ 13/13 hộp (100%) đều bị chìm sâu xuống dưới mặt đất đúng 1.805 m trên ảnh `side-batch-z.png`. Đây là lỗi hệ thống do pipeline quên phép chuyển ngược cao độ, tuyệt đối không chỉnh sửa thủ công từng hộp. |
| case-one-box-z | 1 / 13 | Hộp đầu tiên lệch -1.805 m, 12 hộp còn lại lệch 0 m | Không đổi | Kiểm tra từng đối tượng (object-level) | Chỉ duy nhất 1 hộp đầu tiên bị chìm xuống dưới mặt đất trên `side-one-box-z.png`, 12 hộp còn lại vẫn nằm đúng trên mặt đường. Pipeline vẫn hoạt động bình thường, lỗi chỉ xuất hiện cục bộ ở đối tượng này nên không dừng batch mà chỉ cần kiểm tra sửa riêng hộp lỗi. |

*Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.*

---

## Nhận xét cá nhân

### 1. Nguyễn Đình Mạnh (MSSV: 02306)
- **Vai trò đã làm:** Vận hành Docker runner và chạy script phân tích A/B/C ở lượt B; kiểm tra tham số cấu hình ở lượt A; xem hình học Side view ở lượt C.
- **Quan sát A/B/C có dẫn chứng file:** Ở lượt A (`boxes-demo-delta-0-voxel-0.16.json`), khi delta = 0, model chỉ phát hiện duy nhất 1 xe với confidence score thấp 0.322 và mean_z = 0.330 m do point cloud bị hạ thấp so với chiều cao sensor thực tế. Trong khi ở lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`), khi đặt delta = 1.73 m, model nhận diện được 13 hộp với score lên tới 0.933, bám sát các phương tiện trên làn đường.
- **Diễn giải phép z thuận/ngược:** Pipeline sử dụng phép đổi thuận để đưa tọa độ điểm từ hệ cảm biến về hệ chuẩn của model: `z_model = z_source - z_ground - delta`. Sau khi model dự đoán cuboid trong không gian của nó, bắt buộc phải dùng phép đổi ngược: `z_source = z_model + z_ground + delta` để đưa hộp trở lại đúng hệ quy chiếu PCD ban đầu.
- **Quyết định lỗi batch và hành động:** Khi quan sát thấy toàn bộ các hộp đều bị lệch cùng một lượng z (như trong `case-batch-z`), ta phải dừng ngay việc sửa thủ công trên CVAT, báo ngay cho LC/kỹ sư pipeline để kiểm tra lại công thức chuyển đổi tọa độ của toàn bộ frame.
- **Điều chưa chắc chắn:** Với các đối tượng ở xa (>40m) hoặc bị che khuất một phần, mật độ điểm LiDAR rất thưa nên việc xác định chính xác chiều dài và hướng (yaw) qua ảnh Side vẫn còn chưa chắc chắn, cần đối chiếu thêm với ảnh camera.

### 2. Phạm Thanh Sơn (MSSV: 02274)
- **Vai trò đã làm:** Kiểm tra cấu hình và đọc dữ liệu JSON ở lượt B; vận hành lệnh ở lượt C; ghi log ở lượt A.
- **Quan sát A/B/C có dẫn chứng file:** Khi so sánh lượt B (`voxel_size=0.16`) và lượt C (`voxel_size=0.32` trong `boxes-demo-delta-1.73-voxel-0.32.json`), việc tăng kích thước pillar lên 0.32m làm giảm độ phân giải không gian, khiến model hoàn toàn không nhận diện được xe bốn bánh mà nhận nhầm 6 cụm điểm thành `pedestrian`. Điều này chứng minh cấu hình pillar ảnh hưởng rất lớn đến việc trích xuất đặc trưng hình học của mạng.
- **Diễn giải phép z thuận/ngược:** Phép đổi thuận chuẩn hóa cao độ sensor về mặt đất giúp mô hình PointPillars nhận diện đúng tương quan chiều cao của xe so với mặt đường. Phép đổi ngược khôi phục tọa độ vật thể về đúng vị trí thực tế của mây điểm đầu vào để hiển thị đồng bộ trong CVAT.
- **Quyết định lỗi batch và hành động:** Khi gặp lỗi batch lệch đồng loạt một lượng bằng `delta + z_ground`, đây là bằng chứng rõ ràng của việc pipeline bị thiếu bước cộng ngược `(delta + z_ground)`. Việc cố gắng kéo từng hộp bằng tay sẽ gây lãng phí thời gian và sai lệch tính nhất quán.
- **Điều chưa chắc chắn:** Việc phân biệt giữa xe hai bánh (`two-wheels`) và người đi bộ (`pedestrian`) khi các đối tượng đi sát nhau chỉ dựa trên point cloud thô vẫn rất khó khăn nếu không có kênh intensity hoặc ảnh RGB hỗ trợ.

### 3. Đỗ Tiến Anh (MSSV: 02252)
- **Vai trò đã làm:** Xem hình học Side PNG và Top view ở lượt A và B; kiểm tra JSON ở lượt C; ghi log ở lượt B.
- **Quan sát A/B/C có dẫn chứng file:** Khi quan sát ảnh hình chiếu `side-demo-delta-1.73-voxel-0.16.png` ở lượt B, các hộp cuboid bám sát mặt đường cục bộ quanh vật thể (mean_z = 1.034m), đáy hộp tiếp xúc đúng vị trí cụm điểm bánh xe tiếp đất. Trong khi ở lượt A, hộp duy nhất bị chìm xuống thấp với mean_z = 0.330m.
- **Diễn giải phép z thuận/ngược:** Phép z thuận `z_model = z_source - z_ground - delta` giúp bù trừ chiều cao đặt LiDAR và độ nghiêng mặt đất cục bộ. Phép z ngược `z_source = z_model + z_ground + delta` đảm bảo nhãn dự đoán khớp với tọa độ tuyệt đối của frame LiDAR.
- **Quyết định lỗi batch và hành động:** Trong `case-one-box-z`, chỉ có 1 hộp bị chìm còn 12 hộp khác bình thường, chứng tỏ pipeline hoạt động đúng, lỗi chỉ xảy ra cục bộ ở vật thể đó (có thể do nhiễu điểm hoặc phản xạ bất thường). Ta xử lý bằng cách kiểm tra và chỉnh sửa riêng hộp đó, không dừng cả batch.
- **Điều chưa chắc chắn:** Ở góc nhìn Side, các đối tượng có cùng tọa độ x nhưng khác làn xe y bị chồng lấp lên nhau, khiến việc xác định biên giới giữa hai xe đi song song bị mơ hồ nếu không xoay góc tự do trong không gian 3D.

---

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
