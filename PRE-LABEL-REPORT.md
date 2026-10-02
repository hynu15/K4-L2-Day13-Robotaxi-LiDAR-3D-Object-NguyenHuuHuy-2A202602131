# Báo cáo thực hành PointPillars — Day 13 (cá nhân)

Giữ bản đã điền ngoài Git, trong thư mục private do LC thu. Đây là kiểm tra formative; bài làm cá nhân.

## Học viên và provenance

- Họ tên / MSSV: Nguyễn Hữu Huy / 2A202602131
- Mã lớp/phòng: H210
- Trạng thái: `executed-by-student` — tự chạy đủ ba lượt trên máy cá nhân, không dùng kết quả chuẩn bị sẵn.
- Ngày/giờ chạy: 02/10/2026, 09:39–09:40 (giờ VN) — `smoke.json` ghi 02:39:57Z → 02:40:18Z.
- Hệ máy / architecture: Ubuntu 22.04.5 LTS, Linux 6.8.0-138-generic, x86_64 (`amd64` native, không emulation); Docker Engine 29.8.0.
- Image tag / image ID: `day13-pointpillars:lc-20261001-amd64` — `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
- Phiên bản repo của gói: `repo_revision 0831856d921609312d42c7582c366e5a311bb7b1`, `working_tree_dirty: true` (ghi theo manifest gói Student, không phải commit máy em).
- PCD được cấp / frame_id: `input/demo.pcd`, `frame_id = demo`, 17.238 điểm; KITTI 000008 qua MMDetection3D demo, giấy phép CC BY-NC-SA 3.0. SHA256 input: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Nơi được phép chạy: laptop cá nhân, container `--network none`, giới hạn 4 CPU / 4 GB.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1` (pretrained KITTI có sẵn trong image, không train lại).
- Phạm vi / score threshold: front-window của checkpoint; score `0.3`; cả ba lượt giữ nguyên.
- Giả định kênh thứ tư / intensity: PCD đã bỏ reflectance nguồn, RGB=0 là placeholder, script dùng kênh hằng theo lớp. **Không phải intensity thật**, nên không đọc kết quả như benchmark KITTI.
- Nguồn `z_ground`: ước lượng từ chính PCD, ra `z_ground = 0.075 m`, **giống hệt nhau ở cả ba lượt** → khác biệt A/B/C không đến từ ước lượng mặt đất.
- `smoke.json`: `status: passed`; cả 5 bước (docker-load, run-A, run-B, run-C, qc-cases) đều `passed`.

## Ba lượt inference thật

Số hộp lấy từ cột `n_boxes`, mean_z lấy từ cột `mean_z` trong `run-A/B/C/summary.csv`, không tự tính lại. `mean_z` là cao độ trung bình tâm hộp, **không phải điểm chất lượng**.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON / Side / CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ 1 hộp `vehicles` tại x≈13,2 m, y≈−0,45, score 0,322 — sát ngưỡng 0,3. Side gần như trống hộp dù cụm điểm trải x≈3–60 m. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 13 hộp: 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; x≈3,7–55,6 m; score 0,933 → 0,318. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | 6 hộp, **toàn bộ là `pedestrian`**; x≈9,1–33,5 m; không còn hộp `vehicles` nào. |

- **A/B — chỉ đổi delta (0 → 1,73 m), pillar giữ 0,16:** A có **1** hộp; B có **13** hộp. Đây không phải cùng một tập hộp bị dịch lên: hộp duy nhất của A ở `x=13.154, y=-0.451, yaw=2.671`, còn hộp B gần nó nhất là `x=14.766, y=-1.081, yaw=-0.303` — lệch cả x, y lẫn yaw, không phải chênh đúng một hằng số z. mean_z cũng chỉ chênh 0,330 → 1,034 (≈0,70 m), **không bằng 1,73 m**, và vốn là trung bình trên hai tập hộp khác nhau nên không dùng để suy ra phép dịch. Trong `side-demo-delta-0-voxel-0.16.png` vùng x≈12–15 m chỉ có một khung đỏ nhỏ, còn `side-demo-delta-1.73-voxel-0.16.png` cùng vùng đó có nhiều khung bám cụm điểm, và có thêm hộp ở x≈33 m, 41 m, 56 m mà A hoàn toàn không có. Giải thích: delta đổi *đầu vào* của model (z_model = z_source − z_ground − delta), nên checkpoint KITTI — vốn quen cao độ sensor ≈1,73 m — nhận được phân bố z khớp hơn ở lượt B và cho nhiều phát hiện vượt ngưỡng hơn. **Điều em còn chưa chắc:** không có ground truth trong bài này, nên "13 hộp" chỉ nói model tự tin hơn, chưa chứng minh B đúng hơn A về hình học từng hộp.
- **B/C — chỉ đổi pillar (0,16 → 0,32 m), delta giữ 1,73:** B có **13** hộp; C có **6** hộp. Thay đổi đáng kể nhất không phải số lượng mà là **lớp**: B có 10 `vehicles` + 1 `two-wheels` + 2 `pedestrian`, C có **0 `vehicles`** và 6 `pedestrian`. Bằng chứng theo từng vị trí trong `boxes-*.json`: chỗ B ghi `vehicles x=8.094 y=1.208` thì C ghi `pedestrian x=9.109 y=0.404`; chỗ B ghi `two-wheels x=10.318 y=5.249` thì C ghi `pedestrian x=10.459 y=4.927` — gần như cùng điểm trong scene nhưng đổi nhãn. Trên ảnh Side, `side-...voxel-0.16.png` có các khung đỏ to (xe) kéo tới x≈58 m, còn `side-...voxel-0.32.png` chỉ còn các khung cam hẹp, cao (người đi bộ) và dừng ở x≈34 m. mean_z nhích 1,034 → 1,091, nhưng đó là hệ quả của việc tập hộp đổi sang toàn người đi bộ (cao hơn, tâm cao hơn), không phải "chất lượng tốt hơn". **Có đủ bằng chứng kết luận C tốt hơn không? Không.** C dùng lại đúng checkpoint đã train với pillar 0,16; đưa pillar 0,32 vào là đổi biểu diễn đầu vào so với lúc train, nên kết quả lệch lớp là dấu hiệu model chạy ngoài cấu hình của nó, không phải "pillar lớn hợp với người đi bộ".
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Cả ba lượt chỉ chạy front-window (`rear=0` ở cả A/B/C), nên vật phía sau xe không xuất hiện và **không được tính là model bỏ sót**. Side là hình chiếu x–z của toàn scene, các xe ở y khác nhau chồng lên nhau trong cùng một ảnh, nên hai khung "đè" nhau ở x≈8–11 m chưa chắc là trùng hộp. Yaw gần như không đọc được từ Side (yaw của 13 hộp lượt B trải từ −2,468 đến 2,909 rad mà hình chiếu ngang vẫn trông giống nhau) — muốn duyệt hướng phải mở Top view và ảnh camera.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Không file nào trong ba lượt đủ cơ sở để import: đây là KITTI demo, khác frame với job Robotaxi. Riêng `run-C` còn đáng ngờ về nhãn (mất sạch lớp `vehicles`) nên càng không dùng. Cần kiểm tiếp: đối chiếu Top/Front và ảnh camera cho các hộp score thấp của B (`pedestrian` 0,339 và 0,318, `two-wheels` 0,385) trước khi tin, và kiểm hộp B ở x≈55,6 m (score 0,501) xem có đủ điểm bám không.

## Ca QC có kiểm soát — không import CVAT

Ba ca sinh từ prediction thật của lượt B (`source_prediction_sha256: 16f30b08…`), `height_offset_m = 1.805 = z_ground 0.075 + delta 1.73`. Helper **không chạy lại model**, chỉ biến đổi có chủ đích.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi gì | Không có dấu hiệu lỗi — nhưng **không phải nhãn đúng**, chỉ là bản sao prediction B | `case-correct.json` so với `run-B/boxes-demo-delta-1.73-voxel-0.16.json`: cả 13 hộp trùng từng trường |
| case-batch-z | **13 / 13** | **−1,805 m, giống hệt nhau ở mọi hộp** | Không: x, y, yaw, length, width, height và label giữ nguyên tuyệt đối | **Dừng batch** | `side-batch-z.png`: cả 13 khung nằm trọn **dưới** đường tham chiếu z=0 trong khi đám mây điểm vẫn ở trên; chênh lệch đúng bằng `height_offset_m` trong `qc-cases/manifest.json` |
| case-one-box-z | **1 / 13** | −1,805 m, chỉ hộp đầu (`vehicles`, x=8.094, y=1.208) | Không: 12 hộp còn lại y hệt B; hộp lệch cũng chỉ đổi z | **Kiểm từng hộp** | `case-one-box-z.json` vs `case-correct.json`: duy nhất index 0 lệch z; `side-one-box-z.png` chỉ có một khung chìm dưới z=0 |

**Quyết định và hành động:**

1. `case-batch-z` — mọi hộp lệch **cùng một lượng** và lượng đó **đúng bằng `z_ground + delta`**: đây là chữ ký của việc quên phép chuyển ngược `z_source = z_model + z_ground + delta`. Hành động: **không sửa tay hộp nào**, báo LC kiểm phép chuyển frame và yêu cầu sinh lại prediction từ pipeline đúng. Sửa tay 13 hộp sẽ giấu mất lỗi pipeline và lần sau lặp lại trên cả batch lớn hơn.
2. `case-one-box-z` — chỉ 1/13 hộp lệch, phần còn lại khớp: không phải lỗi transform toàn cục. Hành động: mở hộp đó bằng nhiều view (Top/Side/Front + camera) để xem là lỗi của riêng đối tượng hay hộp đặt sai, rồi sửa riêng; **không dừng cả batch chỉ vì một hộp nổi/chìm**.
3. `case-correct` — giữ nguyên chuyển đổi z của prediction gốc. Em **không** coi đây là cuboid đúng; nó chỉ chứng minh helper không tự ý đổi dữ liệu, nên hai ca kia là biến đổi có kiểm soát chứ không phải lượt detector khác.

Cả ba file đều có `training_only: true` và cảnh báo "not ground truth, not for CVAT import". Em không import `case-*.json` vào CVAT và không dùng chúng để tuyên bố chất lượng.

## Nhận xét cá nhân

Em tự chạy cả ba lượt trên máy cá nhân bằng một lệnh `student-bundle.py run`, tự kiểm `smoke.json`/`manifest.json`, tự đọc `summary.csv` + `boxes-*.json` + `side-*.png` và tự ghi log — không dùng kết quả chuẩn bị sẵn.

Quan sát em thấy rõ nhất là **B/C**: giữ nguyên delta mà chỉ đổi pillar 0,16 → 0,32 thì vị trí `x≈10,3–10,5 m, y≈4,9–5,2 m` đổi từ `two-wheels` (B) sang `pedestrian` (C), và toàn bộ 10 hộp `vehicles` của B biến mất ở C. Việc này nhắc em rằng cỡ pillar không phải tham số "chỉnh cho đẹp" — nó là biểu diễn đầu vào mà checkpoint đã học, đổi nó là chạy model ngoài điều kiện train.

Về phép z: `z_model = z_source − z_ground − delta` chạy **trước** inference, nên đổi delta làm model nhìn thấy một đám mây điểm khác và có thể ra số hộp, vị trí, lớp khác (A: 1 hộp → B: 13 hộp). Còn dịch hộp **sau** inference thì số hộp và x/y/yaw không đổi, chỉ z trượt đi — đúng như `case-batch-z`: 13/13 hộp lệch đúng −1,805 m, mọi trường khác giữ nguyên. Hai thứ nhìn giống nhau trên ảnh Side nhưng nguyên nhân khác hẳn.

Quyết định lỗi batch: với `case-batch-z` em chọn **dừng, không sửa tay**, vì dấu hiệu "cùng một lượng, đúng bằng z_ground + delta, mọi trường khác bất biến" chỉ ra lỗi ở bước chuyển hệ tọa độ chứ không phải ở từng cuboid.

**Điều em còn chưa chắc:** (1) không có ground truth nên em không kết luận được B "đúng hơn" A — chỉ nói model tự tin hơn ở B; (2) `mean_z` chênh 0,330 → 1,034 nhưng em không dùng số này làm bằng chứng vì nó là trung bình trên hai tập hộp khác nhau; (3) PCD này bỏ reflectance thật và dùng kênh hằng RGB=0, nên em chưa biết kết quả sẽ khác bao nhiêu với intensity thật; (4) các hộp B có score 0,30–0,39 em chưa kiểm được bằng camera nên chưa dám nói là đúng hay nhiễu.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét cá nhân và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
