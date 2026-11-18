# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603245 - Nguyễn Sỹ Kiên

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện: bicycle hay motorbike? | Phân loại theo cấu tạo và kiểu dáng: Nếu có bàn đạp trợ lực và khung thanh mảnh -> gán `bicycle`; nếu không có bàn đạp hoặc kiểu dáng xe tay ga/scooter điện công suất lớn -> gán `motorbike`. | `img_005.jpg` | `CAL-01` |
| 2 | Khối xe máy đỗ dày đặc gần camera vs ở xa? | Nếu vật ở tiền cảnh phân biệt được rõ từng cá thể -> gán riêng từng `motorbike`; nếu ở xa/mờ, cụm >= 5 xe san sát và chiều cao mỗi xe < 20 px -> vẽ một box `ignore_region` bao quanh cụm. | `img_028.jpg` | `CAL-04` |
| 3 | Hàng hoá chở trên xe nhô ra xa? | Hàng hoá nhô ra quá 20% chiều dài xe không tính vào box xe. Với mask phân vùng: mask bám khung sườn và giá đỡ cố định, không ôm bạt phủ hàng nhô ra. | `img_009.jpg` | `CAL-02` |
| 4 | Người ngồi trong xe ba bánh có mui che lửng? | Nếu nhìn thấy >= 20% phần cơ thể qua cạnh hở của xe -> bắt buộc gán box `person` riêng chồng lên xe. Nếu bị che khuất > 80% (chỉ thấy mờ đỉnh đầu qua kính) -> không gán `person`. | `img_013.jpg` | `CAL-03` |
| 5 | Kính chắn gió / cửa sổ xe có tính vào mask xe? | Kính chắn gió gắn liền khung xe thuộc mask xe. Người ngồi phía sau kính nhìn rõ vẫn được vẽ mask `person` riêng đè lên theo thứ tự vật thể phía trước. | `img_003.jpg` | `CAL-02` |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_013.jpg` | Xe tuk-tuk chở khách có mui che | YOLO26 gán thành `car` (conf 0.87); YOLOE bắt đúng `three_wheeler` nhưng box lỏng | Chuyển toàn bộ về `three_wheeler`, nắn box ôm khít mép mui và bánh xe. | Mục 2 & 4.1: Xe ba bánh có mui/buồng lái đặc thù Châu Á không gán nhãn car/truck. |
| `img_004.jpg` | Xích lô chở khách có người đạp phía sau | YOLOE vẽ box to bao trùm cả người đạp và xe; YOLO26 gán xe thành `truck` | Bóc tách thành 1 box `three_wheeler` cho xe xích lô và 1 box `person` cho người đạp. | Mục 4.1.4: Người điều khiển xích lô/xe ba bánh luôn là box person độc lập. |
| `img_028.jpg` | Hàng xe máy đỗ dày đặc ven đường | Máy bỏ sót nhiều xe ở xa (bẫy missing và sai nhãn lọt) | Giữ box cho các xe rõ nét ở gần; phần xe mờ ở xa cao < 20px khoanh `ignore_region`. | Mục 4.1.6: Cụm >= 5 xe nhỏ đỗ sát nhau ở xa dùng ignore_region. |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên viết sẵn | Ban hành khung guideline cơ sở |
| v1.1 | Chốt 5 câu hỏi mở (xe điện, hàng nhô ra, ignore_region, người trong xe ba bánh, kính chắn gió); bổ sung quy tắc bóc tách lớp `three_wheeler` cho tuk-tuk/xích lô và phân định ranh giới người lái. | Khắc phục lỗi lặp của máy (YOLO26 gọi tuk-tuk là car), bẫy lọt missing ở cụm xe máy và ca nhập nhằng xe điện (`CAL-01`, `CAL-02`, `CAL-03`, `CAL-04`). |
