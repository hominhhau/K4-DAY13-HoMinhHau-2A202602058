# Báo cáo thực hành PointPillars — Day 13

## Nhóm và provenance

- Mã nhóm/phòng: EmAnhNing/Phòng C402
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: executed-by-group (chạy thật bằng runner trên máy của thành viên 1 — xem STT 1 trong `TEAMMATES.md`).
- Người thực sự chạy: thành viên 1 (xem `TEAMMATES.md`); ngày/giờ: 01/10/2026 ~14:51–14:53 (giờ Hà Nội, theo `smoke.json`); hệ máy/architecture: linux / amd64.
- Image tag và image ID: `day13-pointpillars:lc-20261001-amd64` / `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`.
- Phiên bản repo: `0831856` (`working_tree_dirty: true` theo `smoke.json`).
- PCD được cấp / frame_id: `demo.pcd` (KITTI 000008 chuyển đổi) / frame `demo`; SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`; nơi chạy: chạy trên linux fedora.
- Checkpoint: PointPillars KITTI `epoch_160.pth` có sẵn trong image, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance gốc bị bỏ, script dùng kênh hằng số theo lớp (RGB=0 placeholder); `z_ground = 0.075 m` do script ước lượng từ PCD.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD `demo`. Số hộp từ `n_boxes`, mean_z từ `mean_z` trong `run-A/B/C/summary.csv`.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `boxes-demo-delta-0-voxel-0.16.json` | 1 hộp `vehicles` duy nhất tại x=13.15, y=-0.45, z=0.330, score 0.322 (sát ngưỡng 0.3); ảnh Side chỉ 1 ô đỏ vùng x≈11–15 |
| B | 1.73 | 0.16 | 13 | 1.034 | `boxes-demo-delta-1.73-voxel-0.16.json` | 10 vehicles + 2 pedestrian + 1 two-wheels, score cao nhất 0.933 (vehicles x=8.09); hộp rải x≈3.7–55.6, z≈0.70–1.43 |
| C | 1.73 | 0.32 | 6 | 1.091 | `boxes-demo-delta-1.73-voxel-0.32.json` | Cả 6 hộp đều `pedestrian` (không còn vehicles/two-wheels nào); score cao nhất 0.808 |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side vùng x≈5–35 m: A gần như trống (1 ô đỏ), B phủ kín hộp từ x≈2 đến x≈58. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ — số hộp/lớp/vị trí đổi hẳn, không có hộp nào lệch đúng 1.73 m theo kiểu cộng hằng số. Điều còn chưa chắc: hộp vehicles x=13.15 của A (score 0.322) có tương ứng hộp nào ở B không (gần nhất là vehicles x=14.77, score 0.928) — cần so thêm yaw/kích thước, chưa kết luận.
- B/C — chỉ đổi pillar: B có 13 hộp đủ 3 lớp; C còn 6 hộp toàn pedestrian. Pillar 0.16 → 0.32 m làm mất toàn bộ 10 vehicles và 1 two-wheels trên PCD này; 2 pedestrian của B (x=18.67 và x=34.03) cũng không trùng vị trí 6 pedestrian của C — tức đổi biểu diễn đầu vào làm detector ra tập hộp khác hẳn, không phải quên cộng z ngược. Không đủ bằng chứng nói C tốt hơn: số hộp nhiều hơn / score cao hơn không phải tiêu chí đúng.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Chỉ chạy front ROI nên vật ngoài cửa sổ trước không tính là miss. Ảnh Side là chiếu toàn scene trên mặt x-z (chồng các xe khác y), hộp B x=33.71 và x=40.98 nhìn trên Side gần nhau nhưng y khác (−7.18 so với −9.83) — không dùng một mình ảnh Side để chốt hình học từng hộp; cần Top/Front + camera.
- JSON nào còn chưa đủ cơ sở để import? Cả ba JSON đều là prediction KITTI demo, không JSON nào được import vào job Robotaxi (khác frame, khác sensor). Cần kiểm tiếp: xin prediction đúng frame Robotaxi từ LC/operator rồi đối chiếu job ID/frame/schema/transform trước khi sửa.

## Ca QC có kiểm soát — không import CVAT

Ba ca tạo từ prediction B (13 hộp) bằng helper; lượng lệch `z_ground + delta = 0.075 + 1.73 = 1.805 m`. Đã kiểm bằng script: `batch-z` lệch cả 13 hộp đúng −1.805 m; `one-box-z` chỉ hộp đầu (vehicles x=8.09, z 0.921 → −0.884), 12 hộp còn lại giữ nguyên.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 | Không đổi (copy prediction B, hash `c2a8db24…`) | Baseline đối chiếu | Tâm z trùng JSON B từng hộp |
| case-batch-z | 13/13 | −1.805 m mọi hộp | Chỉ z đổi | Dừng batch — báo LC kiểm transform, yêu cầu tạo lại prediction | `side-batch-z.png`: cả batch chìm cùng lượng dưới đường tham chiếu |
| case-one-box-z | 1/13 | −1.805 m hộp đầu | Chỉ 1 hộp đổi z | Kiểm từng hộp bằng nhiều view | `side-one-box-z.png`: 1 hộp lệch, 12 hộp còn lại khớp B |

Helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng. Không import `case-*.json` vào CVAT.

## Nhận xét cá nhân

- Thành viên 1 (xem STT 1 trong `TEAMMATES.md`) — vận hành lệnh cả 3 lượt: chạy một lệnh runner duy nhất, runner tự load image `day13-pointpillars:lc-20261001-amd64` rồi chạy tuần tự A (6.07 giây, 1 hộp), B (5.70 giây, 13 hộp), C (3.41 giây, 6 hộp), `smoke.json` báo `status: passed`. Quan sát: A/B chỉ đổi delta mà số hộp nhảy từ 1 lên 13, chứng tỏ model chạy lại trên input đã dịch z chứ không phải dịch hằng số output cũ. Phép z thuận `z_model = z_source − z_ground − delta` (0.075 + 1.73 ở lượt B), phép ngược cộng lại khi xuất JSON nên JSON đã ở hệ nguồn, không cộng thêm. Quyết định: nếu gặp cả batch lệch −1.805 m như `case-batch-z` thì dừng sửa tay, báo LC kiểm transform. Chưa chắc: máy chạy là Fedora chưa nằm trong danh sách đã thử (Linux generic/Intel-AMD và Mac ARM), cần LC xác nhận kết quả này được chấp nhận.
- Thành viên 2 (xem STT 2 trong `TEAMMATES.md`) — kiểm cấu hình/JSON (lượt A), ghi log (lượt B), xem hình học (lượt C): đối chiếu `delta`/`voxel_size`/`z_ground`/`frame_id` trong 3 file `boxes-*.json` khớp đúng lệnh đã chạy (A: 0/0.16; B: 1.73/0.16; C: 1.73/0.32; cả ba `z_ground = 0.075`, frame `demo`). Quan sát: B/C chỉ đổi pillar 0.16 → 0.32 m mà mất toàn bộ 10 vehicles và 1 two-wheels, 6 hộp còn lại toàn pedestrian với vị trí không trùng 2 pedestrian của B — đổi biểu diễn đầu vào làm detector ra tập hộp khác hẳn. Diễn giải phép ngược: `z_source = z_model + z_ground + delta`, đã kiểm `case-one-box-z` chỉ hộp đầu lệch −1.805 m còn 12 hộp khớp B nên đây là lỗi từng đối tượng, kiểm bằng nhiều view. Chưa chắc: 2 pedestrian của B (x=18.67, x=34.03) biến mất ở C trong khi 6 pedestrian của C nằm chỗ khác — chưa rõ là cùng người bị dịch hay đối tượng khác, cần góc Trên.
- Thành viên 3 (xem STT 3 trong `TEAMMATES.md`) — xem hình học (lượt A), kiểm cấu hình/JSON (lượt B), ghi log (lượt C): đọc 3 ảnh Side thấy A gần như trống (1 ô đỏ x≈11–15, z≈0–1), B phủ hộp từ x≈2 tới x≈58 ở dải z≈0–2, C chỉ còn 6 ô hẹp (pedestrian) cụm x≈8–20 và một ô ở x≈33. Quan sát: hộp vehicles xa ở B (x=40.98 và x=55.58, score 0.670 và 0.501) vẫn bám cụm điểm thưa phía xa — điểm thưa không có nghĩa đối tượng nhỏ, không co hộp theo vài điểm gần. Hai hộp B x=33.71 và x=40.98 nhìn trên Side gần nhau nhưng y khác hẳn (−7.18 so với −9.83) nên không chốt hình học bằng một ảnh Side. Quyết định: batch lệch đều thì dừng pipeline; một hộp lệch thì kiểm Top/Bên/Trước + camera. Chưa chắc: đáy các hộp xe xa so với mặt đường cục bộ — đường z=0 trên plot chỉ là tham chiếu, cần xem góc Bên trước khi kết luận.
- Thành viên 4 (xem STT 4 trong `TEAMMATES.md`) — ghi log (lượt A), xem hình học (lượt B), kiểm cấu hình/JSON (lượt C): ghi provenance đầy đủ (image ID `e03983bd…`, checkpoint `482dfcf6…`, PCD `3b5ea3da…`, repo rev `0831856` dirty). Quan sát: 3 ca QC đều gắn nhãn `training_only`, manifest trỏ đúng prediction B (hash `c2a8db24…`, 13 hộp) — `case-correct` là bản copy giữ nguyên, không phải nhãn đúng hay đáp án. Quyết định: `case-batch-z` (13/13 lệch −1.805 m) thì dừng batch báo LC; `case-one-box-z` (1/13) thì kiểm từng hộp; không import bất kỳ `case-*.json` nào vào CVAT và không nạp prediction demo vào job Robotaxi. Chưa chắc: mean_z của C (1.091) cao hơn B (1.034) dù C toàn pedestrian — chưa rõ do phân bố vị trí hay bias ước lượng, cần so tâm z từng hộp trước khi bàn sâu.

## LC ghi nhận riêng


LC ghi nhận ngày 01/10/2026. Kết luận: ĐẠT.

- Quyền dùng PCD/image và đúng ca: Gói Student KITTI 000008 (giấy phép CC BY-NC-SA 3.0), không dùng dữ liệu Robotaxi. Input SHA-256 3b5ea3da… và image sha256:e03983bd… (amd64) khớp smoke.json.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Chạy thật (executed-by-group, Linux Fedora). smoke.json passed, chạy 14:51–14:52; nạp image 20,9 s, A/B/C 6,1 / 5,7 / 3,4 s, kết quả 1/13/6; không trùng nhóm nào khác. Máy Fedora cho kết quả giống hệt các máy khác nên được chấp nhận. Không cần lượt bổ sung.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đủ run-A/B/C và qc-cases. Không sửa JSON. Báo cáo nêu rõ không import JSON KITTI và case-* vào CVAT.
- Nhận xét từng thành viên và quyết định dừng pipeline: Đủ 4 người, mỗi người có quan sát riêng dẫn file và tọa độ (đều khớp file thật), phép z đúng (JSON đã ở hệ nguồn, không cộng thêm), quyết định batch/one-box đúng, điều chưa chắc cụ thể. Nhận ra C mất hết xe, 6/6 pedestrian, và biết không ghép hộp giữa các lượt khi chưa đủ bằng chứng.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đồng ý chuyển sang chỉnh/QC. Một trong những báo cáo có bằng chứng tốt nhất lớp.
