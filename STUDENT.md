# Thông tin học viên

- Họ và tên: Nguyễn Sỹ Kiên
- MSHV: 2A202603245
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: https://github.com/justAchillGuy8386
- Repo bài nộp (URL): https://github.com/justAchillGuy8386/K5-L2-Day08-NguyenSyKien-2A202603245-AI-Assisted-Labeling
- Commit chốt bài (hash): 
- Máy: Win 11, GPU không có
- CVAT (Docker): v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 470 (90.4%) / 10 (1.9%) / 40 (7.7%) (mẫu số: 520 box máy); added: 23 (4.6%) (mẫu số: 503 đối tượng)
- So với gold: precision / recall của **máy**: 63.9% (46/72) / 63.0% (46/73) (mẫu số: 73 đối tượng gold)
- Bắt bẫy: 6/12 = 50.0%
- Thời gian: gán tay 7.5 phút/ảnh (30 phút / 4 ảnh); duyệt có pre-label 2.0 phút/ảnh (48 phút / 24 ảnh)
- Lỗi QC: 33.3% theo ảnh (2/6), 1.8% theo đối tượng (2/111)
- **Quyết định:** Rework — lý do một dòng: Tỉ lệ ảnh có lỗi 33.3% > 2% và tỉ lệ bắt bẫy chỉ đạt 50.0% (< 80%) do ảnh hưởng của thiên kiến tự động hóa, cần rà soát lại lô.
