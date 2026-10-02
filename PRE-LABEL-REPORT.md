# Báo cáo thực hành PointPillars — Day 13

Bài làm **cá nhân**. Giữ bản đã điền ngoài Git, nộp vào nơi thu private do LC chỉ định. Kiểm tra formative.

## Nhóm và provenance

- Mã nhóm/phòng: (điền theo LC) — làm cá nhân, không có thành viên khác.
- Thành viên: xem `TEAMMATES.md`.
- Trạng thái: **`executed-by-group`** (tự chạy cá nhân trên laptop của mình, không dùng kết quả có sẵn).
- Người thực sự chạy: Khuất Tuấn Anh. Ngày/giờ: 02/10/2026, 09:34–09:37 (UTC+7), tức 02:34:36–02:37:14 UTC theo `smoke.json`.
- Hệ máy: Windows 11 Home, Docker Desktop 29.8.0 (Linux containers), Python 3.11.9. Kiến trúc image và runtime: `linux/amd64`.
- Image tag: `day13-pointpillars:lc-20261001-amd64`. Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`.
- Phiên bản repo: Student repo `e226b93`. Code trong gói: revision `0831856d921609312d42c7582c366e5a311bb7b1`, `working_tree_dirty: true` (đúng như VALIDATION của gói ghi). Hash `preannotate.py`: `65edf6ac…eb5ca`; hash helper: `c177fc00…aa7`.
- PCD: `input/demo.pcd` trong gói Student, là KITTI demo `000008` đã chuyển đổi (z +1,73 m, bỏ reflectance, RGB=0), giấy phép CC BY-NC-SA 3.0. `frame_id = demo`. SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`, khớp hash ghi trong VALIDATION.
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window của checkpoint. Score threshold 0,3. Giới hạn container: 4 CPU / 4 GB. Container chạy không có mạng.
- Kênh thứ tư/intensity: PCD không có reflectance thật, adapter dùng kênh hằng. `z_ground` do script ước lượng từ PCD, được **0,075 m** (giống nhau ở cả ba lượt).
- Thời gian chạy (wall, theo `smoke.json`): docker-load 117,9 s; A 13,8 s; B 12,5 s; C 8,4 s; qc-cases 4,7 s.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ có 1 hộp `vehicles` tại x≈13,2, y≈−0,5, score 0,32 (vừa qua ngưỡng 0,3). Đáy hộp ở z≈−0,40, tức nằm dưới đường z=0 khoảng 0,4 m trên ảnh Side. Các cụm điểm dày ở x≈3–25 m gần như không có hộp nào. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Gồm 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; score 0,32–0,93. Các xe gần (x≈3,7–25) có đáy z≈−0,05…0,15, sát mặt đường trên ảnh Side. Các xe xa (x≈33–56) có đáy z≈0,32–0,46; dải điểm mặt đường ở vùng này trên ảnh Side cũng cao hơn 0. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Cả 6 hộp đều là `pedestrian`, không còn hộp `vehicles` nào; score 0,30–0,81. Một số hộp nằm trong vùng B đã gán `vehicles`, ví dụ C (13,2; −1,0) nằm gần xe B (14,8; −1,1). |

- **A/B — chỉ đổi delta:** A có 1 hộp, B có 13 hộp. Trên `side-demo-delta-0-voxel-0.16.png`, các cụm điểm ở x≈3–25 m gần như không có hộp. Trên `side-demo-delta-1.73-voxel-0.16.png`, các cụm này đều có hộp, phần lớn đáy bám dải điểm mặt đường. Hộp duy nhất của A (13,2; −0,5; z=0,33; yaw 2,67) gần nhất với xe B (14,8; −1,1; z=0,90; yaw −0,30). Chênh lệch giữa hai hộp này là Δz≈0,57 m (không phải 1,73 m), Δxy≈1,7 m, và yaw lệch ≈2,97 rad (≈170°, gần như ngược đầu). Điều này cho thấy B là **model chạy lại trên input khác**, không phải hộp A bị dịch đi. Cách hiểu của em: checkpoint KITTI quen với input mà mặt đường nằm ở khoảng −1,73 m so với gốc. Với delta=0, sau khi trừ `z_ground` thì mặt đường ở ≈0, lệch khỏi phân bố lúc train nên model gần như không phát hiện được gì. Điều em chưa chắc: không có camera hay ground truth, nên chưa thể kết luận các hộp của B là đúng. Chỉ có thể nói B hợp lý hơn về phân bố input.
- **B/C — chỉ đổi pillar:** B có 13 hộp, C có 6 hộp. Lớp thay đổi hoàn toàn: từ 10 xe + 2 người + 1 xe hai bánh thành 6 người. Có ít nhất 4 hộp C nằm trong hoặc sát vùng có hộp B: (9,1; 0,4) gần xe B (8,1; 1,2); (13,2; −1,0) gần xe B (14,8; −1,1); (19,4; −8,1) gần xe B (20,7; −8,7); (10,5; 4,9) gần xe hai bánh B (10,3; 5,2). Hộp C cao nhất có score 0,81 nhưng lại là `pedestrian` ở chỗ B thấy xe, nên score cao không chứng minh hộp đúng. Cách hiểu của em: checkpoint được train với pillar 0,16 m. Khi đổi sang 0,32 m, lưới đặc trưng đổi tỉ lệ, mỗi cột gom gấp 4 lần diện tích, nên mạng nhận một biểu diễn khác hẳn lúc train. Đây là lỗi biểu diễn input, không phải lỗi quên cộng z ngược: `mean_z` của B (1,034) và C (1,091) gần nhau. **Có đủ bằng chứng để nói C tốt hơn không?** Không. Bằng chứng nghiêng về việc C sai class, nhưng muốn chốt vẫn cần xem góc Trên/Trước và camera.
- **Giới hạn ROI và góc Side:** model chỉ xét cửa sổ phía trước, nên vật ở x<0 hoặc ngoài cửa sổ không bị tính là bỏ sót. Ảnh Side chiếu toàn cảnh xuống mặt x–z nên các vật khác y bị chồng lên nhau. Ví dụ ở x≈9–11, xe B (9,4; 4,2) và xe hai bánh B (10,3; 5,2) chồng nhau trên Side, nên không đọc được footprint hay yaw từ ảnh này. Yaw và miss phải xem ở góc Trên. Đường z=0 trên Side chỉ là đường tham chiếu, không phải mặt đường cục bộ: vùng x≈30–50 có dải điểm đất cao hơn 0.
- **JSON nào chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Không file nào được import. Cả ba là prediction KITTI demo, không có camera, và không thuộc frame Robotaxi. Nếu đây là dữ liệu thật, các điểm cần kiểm tiếp trong B là:
  1. Cặp xe (9,4; 4,2) có đáy 0,65 m và xe hai bánh (10,3; 5,2) chỉ cách nhau ≈1,4 m. Có thể là hai hộp cho cùng một vật hoặc nhầm class, cần xem góc Trên và camera.
  2. Hướng của các xe, vì có hai nhóm yaw ≈2,7–2,9 và ≈−0,3 (ngược nhau khoảng 180°). Cần đối chiếu đầu xe.
  3. Hai `pedestrian` score thấp (0,32–0,34).
  4. Đáy các xe xa so với mặt đường cục bộ.

## Ca QC có kiểm soát — không import CVAT

Nguồn: `qc-cases/manifest.json` có `training_only: true`. Source là `boxes-demo-delta-1.73-voxel-0.16.json` (lượt B), hash `16f30b08…cf61`, trùng `prediction_sha256` của run-B trong `smoke.json`. `height_offset_m = 1,805 = delta 1,73 + z_ground 0,075`.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không (mọi trường giống B) | Không có dấu hiệu lỗi pipeline. Vẫn **không** phải cuboid đúng, chỉ là bản sao prediction B | `case-correct.json` so với B: không trường nào khác. `side-correct.png` giống hệt Side của B. mean_z 1,034 |
| case-batch-z | 13 / 13 | −1,805 m, đều nhau ở mọi hộp | Không: class, x, y, yaw, kích thước, score giữ nguyên | **Dừng sửa tay cả batch.** Báo LC kiểm phép chuyển ngược (thiếu `+ z_ground + delta`) và yêu cầu chạy lại prediction | Mọi hộp lệch cùng một lượng, đúng bằng `delta + z_ground`. Đó là dấu hiệu của bước đổi tọa độ, không phải lỗi từng vật. `side-batch-z.png`: toàn bộ hộp chìm dưới z=0, nằm dưới dải điểm. mean_z giảm từ 1,034 xuống −0,771 |
| case-one-box-z | 1 / 13 (hộp thứ 1: `vehicles` x≈8,1, y≈1,2) | −1,805 m (z từ 0,92 xuống −0,88) | Không; 12 hộp còn lại giữ nguyên | **Kiểm riêng hộp đó** bằng Trên/Bên/Trước + camera, coi là lỗi đối tượng. Không kết luận lỗi pipeline chỉ vì một hộp chìm | `side-one-box-z.png`: chỉ một hộp ở x≈6–10 nằm dưới mặt đường, các hộp khác bám điểm như B. mean_z 0,895. Ghi chú: lượng lệch trùng `delta + z_ground` là dấu hiệu nên báo thêm cho LC, nhưng các hộp khác đúng nên chưa phải lỗi cả batch |

Ba ca trên do helper tạo bằng cách cố ý biến đổi prediction thật của B. Chúng không phải kết quả inference riêng và không phải nhãn đúng. Em không import file `case-*.json` nào vào CVAT.

## Nhận xét cá nhân

**Khuất Tuấn Anh — MSSV 2A202602259** (làm một mình, kiêm mọi vai: vận hành lệnh, kiểm cấu hình/JSON, xem hình học, ghi log).

- **Quan sát A/B/C có dẫn file:** trong `run-A/side-demo-delta-0-voxel-0.16.png` chỉ có 1 hộp, đáy ở z≈−0,4, dưới đường z=0. Trong `run-B/side-demo-delta-1.73-voxel-0.16.png` có 13 hộp, phần lớn đáy bám mặt đường. Trong `run-C/boxes-demo-delta-1.73-voxel-0.32.json` cả 6 hộp là `pedestrian`, có hộp nằm ngay vùng B gán xe, như (13,2; −1,0) so với (14,8; −1,1).
- **Phép z thuận/ngược:** trước inference, script tính `z_model = z_source − z_ground − delta`. Khi xuất hộp, script cộng lại: `z_source = z_model + z_ground + delta`. Đổi delta *trước* inference làm model nhìn một input khác, nên số hộp, vị trí, class và yaw đều có thể đổi. Từ A sang B, hộp gần nhau chỉ lệch Δz≈0,57 m chứ không phải 1,73 m. Ngược lại, quên phép cộng ngược *sau* inference thì mọi hộp lệch cùng một lượng mà x/y/class không đổi, đúng như `case-batch-z` (−1,805 m ở cả 13/13 hộp).
- **Quyết định lỗi batch:** với `case-batch-z`, em **dừng batch**, không sửa tay từng hộp, và báo LC kiểm bước chuyển frame/phép ngược rồi tạo lại prediction. Với `case-one-box-z`, em **kiểm từng hộp**: chỉ hộp (8,1; 1,2) cần xem lại bằng nhiều góc nhìn.
- **Điều chưa chắc:**
  1. Không có camera hay ground truth nên chưa nói được hộp nào của B đúng. B chỉ là mốc so sánh.
  2. Đáy các xe xa ở z≈0,3–0,46. Có thể mặt đường cục bộ thật sự cao hơn (dải điểm trên Side gợi ý vậy), cũng có thể do Side chồng các vật khác y. Cần góc Bên của từng hộp.
  3. Cặp xe (9,4; 4,2) và xe hai bánh (10,3; 5,2) có thể là trùng hoặc nhầm class.
  4. PCD không có reflectance thật (dùng kênh hằng), nên kết quả không đại diện cho benchmark KITTI chuẩn.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
