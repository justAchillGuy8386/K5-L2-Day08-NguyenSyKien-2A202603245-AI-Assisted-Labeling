# Báo cáo Day 08 — 2A202603245 - Nguyễn Sỹ Kiên

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m | mặc định 0.3 | Quét threshold_sweep trên 4 ảnh gold cho thấy các lớp person, motorbike, car không đạt recall 90% ở bất kỳ ngưỡng nào <= 0.5 (tối đa 76-88%); các lớp bicycle (3 vật), truck (1 vật), bus (0 vật) có < 5 mẫu nên không đủ ý nghĩa thống kê để chọn ngưỡng riêng. Thử nghiệm r03 (conf 0.25) sinh nhiều box rác còn r04 (conf 0.5) bỏ sót nặng nề (-76 box). Do đó giữ conf mặc định 0.3 theo run r03/r04. |
| three_wheeler | YOLOE-26s, prompt: three_wheeler | 0.15 | Thử nghiệm r05 trên pilot cho thấy YOLOE bắt chính xác tuk-tuk/cyclo mà COCO bỏ qua. Quét ngưỡng trên gold cho thấy ngưỡng 0.15 đạt recall 100% (>= 90%) và precision 78% trên 7 vật gold (threshold_sweep.csv). |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn** (SAM 1 / 2.1 / 3) và lý do: Chọn SAM 2.1 (`sam2.1_b.pt` / `sam2.1_t.pt`) vì cân bằng tối ưu giữa độ chính xác bám biên polygon mask, tốc độ xử lý nhanh và độ ổn định trên máy local, không bị rào cản tài khoản gated như SAM 3.
- Lỗi nào **không sửa được bằng đổi ngưỡng** (ví dụ tuk-tuk bị gọi `car`)? Sửa thế nào? Lỗi phân loại sai bản chất do closed-set detector: YOLO26 huấn luyện trên COCO không có nhãn `three_wheeler` nên dù đổi ngưỡng thế nào thì tuk-tuk/xích lô/ba gác vẫn bị gán nhầm thành `car` hoặc `truck`. Cách sửa: Phải kết hợp mô hình open-vocabulary (YOLOE) với text prompt riêng cho `three_wheeler`, khi gộp nhãn (`merge_prelabels`) thiết lập IoU >= 0.6 để nhãn `three_wheeler` ghi đè lên box `car/truck` của YOLO26.
- Box **điểm cao mà sai** hoặc **điểm thấp mà đúng** (ảnh nào): Điểm cao mà sai: Ở `img_003.jpg`, xe tuk-tuk bị YOLO26 nhận diện thành `car` với độ tin cậy rất cao (0.87). Điểm thấp mà đúng: Ở `img_013.jpg`, YOLOE phát hiện tuk-tuk với điểm tin cậy chỉ 0.18 nhưng box bao đúng xe thực tế.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh) nên số đo máy trên gold là lạc quan: Hai giới hạn bắt buộc phải thừa nhận: (1) Chọn và chấm trên cùng dữ liệu: ngưỡng được tối ưu hóa trên chính 4 ảnh gold rồi đo lại máy trên gold nên precision/recall là con số lạc quan (in-sample bias); (2) Mẫu nhỏ: chỉ có 4 ảnh gold (73 đối tượng), các lớp hiếm (bus, truck, bicycle) có dưới 5 vật nên kết luận thống kê rất yếu và không thể chọn ngưỡng riêng.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 470 | 470/520 = 90.4% | 520 box máy |
| edited | 10 | 10/520 = 1.9% | 520 box máy |
| deleted | 40 | 40/520 = 7.7% | 520 box máy |
| added | 23 | 23/503 = 4.6% | 503 đối tượng sau duyệt |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)? Lớp `three_wheeler` là tệ nhất với 17/40 box bị xóa (tỉ lệ xóa lên tới 39.5% tổng số box máy gán lớp này). Nguyên nhân do YOLOE ở ngưỡng thấp 0.15 bị over-detection (dương tính giả), nhận diện nhầm các khối kiến trúc, biển hiệu hoặc thùng hàng trên vỉa hè thành xe ba bánh (ví dụ ở `img_008.jpg`, `img_026.jpg`). Đồng thời có 8 trường hợp máy bắt đúng vị trí nhưng gán sai tên lớp (nhầm giữa car/truck và three_wheeler).
- So với gold: precision / recall của **máy** Precision: 63.9% (46/72, KTC 95%: 52–74%), Recall: 63.0% (46/73, KTC 95%: 52–73%) (mẫu số: 73 đối tượng gold). Mô hình máy thực tế bỏ sót tới 37.0% số đối tượng thật trong ảnh gold.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: 6/12 = 50.0% (KTC 95%: 25–75%); loại nào lọt: Lọt 6 bẫy gồm 2 missing (`img_005.jpg`, `img_033.jpg`), 2 wrong_class (`img_028.jpg`, `img_034.jpg`), 1 extra (`img_025.jpg`), 1 loose (`img_035.jpg`); vì sao (bạn đã bỏ bước nào của checklist)? Đã bỏ qua Bước 1 của checklist ("Ẩn nhãn máy, quét ảnh trước") ở những ảnh có mật độ phương tiện đông (`img_005.jpg`), khiến mắt bị dẫn dắt bởi các box có sẵn và không nhận ra các vật thể bị máy bỏ sót. Với lỗi `wrong_class`, người duyệt nhìn thấy box bao quanh khít đối tượng thì chấp nhận ngay mà không kiểm tra kỹ nhãn phân loại hiển thị trên đối tượng.
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? Tỉ lệ accepted rất cao (90.4%, 470/520 box) nhưng tỉ lệ bắt bẫy chỉ đạt 50.0% (6/12 bẫy) là dấu hiệu kinh điển của **Thiên kiến tự động hóa (Automation Bias)**: người duyệt rơi vào trạng thái tin tưởng thụ động vào AI, trở thành "người bấm Accept hàng loạt" thay vì đóng vai trò chốt chặn kiểm soát chất lượng độc lập. Con số accepted 90.4% phản ánh sự dễ dãi của người duyệt chứ không đồng nghĩa với việc AI chính xác 90.4% (nhất là khi recall của máy trên gold chỉ đạt 63.0%).

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 7.5 phút/ảnh (30 phút/4 ảnh) vs duyệt 2.0 phút/ảnh (48 phút/24 ảnh). Thiên lệch (bạn gán gold trước nên đã quen cảnh): Tốc độ duyệt nhanh hơn gấp ~3.75 lần so với gán tay. Tuy nhiên có thiên lệch học tập (learning effect): việc tự gán 4 ảnh gold trước đó đã giúp người duyệt nắm rất rõ bối cảnh đường phố, hình thái phương tiện và các trường hợp nhập nhằng, giúp việc duyệt ở tập batch diễn ra nhanh hơn thực tế thông thường.
- Kết luận: CÓ ĐÁNG DÙNG nhưng CHỈ ĐÁNG DÙNG CÓ ĐIỀU KIỆN. Đáng dùng rất tốt cho các lớp có tần suất cao, hình thái chuẩn (`person`, `motorbike`, `car` với tỉ lệ accepted > 90%, tiết kiệm lượng lớn thời gian kéo thả box). KHÔNG THỂ PHÓ MẶC cho máy ở lớp `three_wheeler` (tỉ lệ xóa ~40%) và các vật thể nhỏ/bị che khuất (máy bỏ sót 37% trên gold), bắt buộc phải có quy trình rà soát riêng biệt.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Quy tắc Ẩn nhãn quét trước (Blind scan first):** Trước khi nhìn nhãn máy, tắt layer nhãn trong 15–20 giây đầu tiên để mắt tự quét toàn ảnh, chủ động phát hiện các đối tượng nhỏ, bị che hoặc nằm ở rìa ảnh trước khi bật gợi ý của máy.
  2. **Quy tắc Lọc duyệt riêng theo từng lớp (Per-class filtered review):** Sử dụng thanh công cụ Filters trên CVAT để duyệt riêng các lớp có nguy cơ sai sót cao (`three_wheeler`, `bus`, `truck`), buộc mắt phải đối chiếu kỹ nhãn phân loại với định nghĩa của guideline thay vì chỉ nhìn viền hộp.
  3. **Quy tắc Giới hạn số lượng duyệt và nghỉ định kỳ (Fatigue cap):** Duyệt tối đa 8 ảnh liên tục (hoặc không quá 20 phút), sau đó bắt buộc nghỉ mắt 3–5 phút để tránh mỏi mắt dẫn đến thói quen bấm Accept tự động theo quán tính.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 3 click (2 dương, 1 âm) | Không gồm người lái: Điểm âm ở cabin (`720,300`) đã tách thành công người lái khỏi xe theo đúng Guideline 4.2. | Mép mui xe và kính chắn gió bị lẹm nhẹ do chói sáng; phần khung sắt mỏng bị phủ lấp. | `tuktuk_img013.jpg` |
| `img_004.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 5 click (3 dương, 2 âm) | Không gồm người lái: Click âm (`750,280`) loại bỏ hoàn toàn người đạp xích lô phía sau; giữ lại giỏ và mui xe. | Khoảng trống giữa các nan hoa căm xe và khung xích lô bị SAM phủ kín (không khoét lỗ nhỏ). | `cyclo_img004.jpg` |
| `img_003.jpg` | SAM 2.1 base (`sam2.1_b.pt`) | 5 click (3 dương, 2 âm) | Tách được người ngồi trong: Click âm (`450,330`) khoét bỏ người lái bên trong mui, mask ôm sát vỏ xe. | Biên mask phía dưới bánh xe bị lem nhẹ vào vùng bóng đổ tối trên mặt đường. | `tuktuk_img003.jpg` |

- Đối chiếu guideline mục 4.2: quy tắc nào giúp, quy tắc nào thiếu? 
  - *Quy tắc giúp ích:* Quy tắc 4.2.4 (người và xe là hai đối tượng độc lập, vùng chạm nhau tách riêng) định hướng rõ ràng việc đặt điểm click âm để bóc tách người lái ra khỏi thân xe; Quy tắc 4.2.2 (bỏ qua lỗ thủng < 5% diện tích) giúp giảm thiểu số click không cần thiết vào các nan hoa hay gióng xe rỗng ruột.
  - *Quy tắc còn thiếu:* Guideline v1.0 chưa có quy định rõ ràng về kính chắn gió/cửa sổ xe nhìn xuyên thấu (có tính vào mask thân xe hay bỏ ra) và hàng hoá nhô ra khỏi xe ba gác (cắt bỏ hay ôm theo).
- *(Tuỳ chọn)* Interactor SAM trong CVAT: làm được / không (lỗi gì, đã thử gì); khác gì offline: 
  - Interactor trong CVAT trực quan và điều chỉnh theo thời gian thực rất nhanh khi có GPU. Tuy nhiên trên môi trường CPU-only (hoặc máy RAM <= 8GB), chạy Interactor qua Nuclio/CVAT Docker phản hồi khá chậm (trễ 4–8 giây mỗi lần click) và dễ gây treo worker.
  - Sử dụng script offline `sam2_points.py` có ưu điểm vượt trội về khả năng tái lập (reproducibility): lưu lại chính xác toạ độ pixel từng điểm click dương/âm, sinh file ảnh có đánh dấu điểm trực quan để kiểm toán đối chứng.

## 6. Calibration (Part 6)

- Số quyết định: 4 quyết định (`CAL-01` đến `CAL-04` trong `calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): `CAL-02` (quyết định chuẩn hoá toàn bộ xe ba bánh có mui/tuk-tuk/ba gác thành `three_wheeler` thay vì để model YOLO26 gán nhầm thành `car` hoặc `truck`) và `CAL-01` (chốt phân loại dứt điểm xe đạp điện và xe máy điện dựa trên bàn đạp trợ lực).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: 
  1. Phân loại xe điện có bàn đạp là `bicycle`, không có bàn đạp/dáng scooter là `motorbike` (mục 2).
  2. Toàn bộ xe ba bánh bản địa có mui buồng lái (tuk-tuk, cyclo, ba gác) bắt buộc gán `three_wheeler` (mục 2 & 4.1).
  3. Bóc tách người ngồi trên/trong xe ba bánh thành box `person` độc lập nếu nhìn thấy >= 20% cơ thể qua khung hở (mục 4.1.4).
  4. Quy định vẽ box `ignore_region` cho các cụm >= 5 xe máy đỗ san sát nhau ở xa có chiều cao < 20 px (mục 4.1.6).
- Nguyên nhân gốc: Phân định rõ giữa lỗi do guideline v1.0 mơ hồ (chưa hướng dẫn xe điện, hàng nhô ra và cửa sổ kính -> cần sửa guideline) và lỗi do máy/người làm sai (YOLO26 là closed-set detector thiếu lớp ba bánh; người duyệt bị thiên kiến tự động hóa bỏ qua bước quét ảnh trước -> không do guideline).

## 7. QC và quyết định (Part 6)

- Mẫu: 6 ảnh (1 ảnh `can_review` là `img_003.jpg` + 5 ảnh ngẫu nhiên: `img_006.jpg`, `img_013.jpg`, `img_014.jpg`, `img_022.jpg`, `img_033.jpg`, seed 42); người kiểm: Nguyễn Sỹ Kiên (**hạn chế**: người kiểm định QC cũng chính là người duyệt batch ở Part 4 nên có thể tồn tại điểm mù chủ quan).
- Lỗi theo **đối tượng**: 2/111 = 1.8% · theo **ảnh**: 2/6 = 33.3% (ngưỡng Release ≤ 2%).
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại:
  + `sai_vi_tri_kich_thuoc`: 1 lỗi ở `img_003.jpg` (box mui xe tuk-tuk bị rộng dư 4px về phía sau).
  + `bo_sot`: 1 lỗi ở `img_033.jpg` (bỏ sót 1 xe máy nhỏ ở hậu cảnh do bị bóng râm che khuất).
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): Cận trên 95% khoảng tin cậy Wilson của tỉ lệ ảnh lỗi thật lên tới **70%** (với 2/6 ảnh lỗi). Với kích thước mẫu nhỏ 6 ảnh, việc đạt tỷ lệ lỗi theo đối tượng 1.8% không thể xem là bằng chứng thống kê vững chắc chứng minh toàn bộ lô 24 ảnh đạt chuẩn <= 2%.
- Script kiểm được / không kiểm được: Script kiểm tra được định dạng dữ liệu, tính nhất quán của ID, toạ độ hình học hợp lệ, đếm số lỗi và tính tỷ lệ thống kê; Script **không thể kiểm tra** được box có ôm sát biên thực tế của vật thể hay không, không thể phát hiện các đối tượng mà cả AI lẫn người duyệt cùng bỏ sót ngoài thực tế, và không thể giám sát việc người duyệt có thực hiện đúng checklist hay không.

**Quyết định: REWORK**, ngày 2026-11-18. Lý do (đi theo Hold → Rework → Release):
- Đi theo cây quyết định:
  1. *Hold:* Không áp dụng vì 5 câu hỏi mở đã được chốt và văn bản hoá trong Guideline v1.1 (`guideline-gap = 0`), đồng thời không có ảnh hỏng/mờ trong tập dữ liệu (`data-problem = 0`).
  2. *Rework:* **ĐƯỢC KÍCH HOẠT** vì tỉ lệ ảnh có lỗi trong mẫu QC đạt 33.3% (2/6 ảnh), vượt xa ngưỡng tối đa của Release (<= 2%); đồng thời tỉ lệ bắt bẫy nhân tạo chỉ đạt 50.0% (6/12 bẫy), thấp hơn ngưỡng tin cậy 80% do ảnh hưởng của thiên kiến tự động hóa.
  3. Do đó, quyết định trung thực và chuẩn xác theo quy trình là **REWORK** để đảm bảo chất lượng nhãn.
Nếu Rework / Hold, việc cần làm tiếp:
- Thực hiện rà soát lại toàn bộ 24 ảnh batch với quy tắc bắt buộc: ẩn nhãn máy trong 20 giây đầu để tự quét bằng mắt.
- Sử dụng bộ lọc Filters trên CVAT để kiểm tra riêng 100% các đối tượng thuộc lớp `three_wheeler` và `motorbike` ở hậu cảnh.
- Xuất nhãn sau khi sửa thành `submission/labels/qc_fixed.json`, chạy lại `build_release.py` và lấy mẫu QC vòng 2.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): Dùng mô hình Gemini/Antigravity hỗ trợ phân tích dữ liệu thống kê, lập trình script tự động hóa và rà soát lỗi cú pháp; mọi số liệu đo lường trong báo cáo được trích xuất trực tiếp từ các file kết quả thực thi (`review_stats.json`, `threshold_sweep.csv`, `qc_report_auto.md`).
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? Không bị nhiễm. Tập gold gồm 4 ảnh được gán tay 100% độc lập từ đầu trên CVAT theo Guideline v1.0 trước khi chạy bất kỳ mô hình pre-label nào.
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, CPU-only (không có GPU rời), CVAT chạy local qua Docker v2.74.1. Thời gian suy luận mô hình trên CPU mất khoảng 0.5–1.2s/ảnh cho detector và ~8s/ảnh cho SAM.
- Dùng cache `data/cache/` thay vì tự chạy? Không dùng cache; đã tự chạy đầy đủ các lệnh thực tế trên máy cục bộ với CPU và lưu trữ vết thực thi trong `work/runs.csv`.
- Lỗi / bug script đã gặp (nếu có): Ban đầu Windows gọi nhầm Python 3.12 thiếu thư viện (đã giải quyết bằng launcher `py` trỏ vào Python 3.13); upload annotation vào CVAT ban đầu báo lỗi thiếu label (đã giải quyết bằng cách nạp `labels_generated.json` vào Task).
- Một điều bạn làm khác lần sau: Sẽ luôn ẩn layer gợi ý của AI ngay từ đầu để quét tự nhiên bằng mắt trước, tránh hiện tượng phụ thuộc vào gợi ý máy (anchoring effect) dẫn đến bỏ sót bẫy nhân tạo.
