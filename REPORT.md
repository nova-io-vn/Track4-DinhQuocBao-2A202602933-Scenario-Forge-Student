MSSV: 2202602933
MSSV đầy đủ theo danh sách lớp: 2A202602933

# Lab 11 — Scenario Forge

Thị trấn B (meta.json: _run.town), chạy đầy đủ, không --quick (evidence/meta.json: _run.quick = false). Dòng MSSV đầu giữ định dạng số mà run_all.py yêu cầu; mã đầy đủ được ghi ở dòng kế tiếp.

## 1. Phát lại log (bước A)

replay.json: n=400, dropped human-crash logs=0; chạy replay có 123 va chạm (115 rear, 8 side), chạy reactive có 0. Phần lớn va chạm rear là xe sau trong log vẫn giữ quỹ đạo người lái và không biết ego đã phanh sớm hơn; đó là va chạm phản thực giả do đóng băng hành vi xe khác. replay.json: scenario 1 là replay_only, contact rear; TTC reactive tối thiểu 2.61 s. Một xe cắt làn xuất hiện rồi rời đi trong bản reactive, trong khi xe theo sau theo log không giảm tốc tương ứng và đâm đuôi ego trong bản replay.

## 2. Tính khả thi (bước B)

feasible.py loại chuyển làn đòi gia tốc ngang (pi/cutDur)^2*3.5/2 vượt mu*g, phanh xe chuyển làn vượt mu*g, người đi bộ nhanh hơn 2.5 m/s, và cutV ngoài (0,22] m/s. Đây là giới hạn bám đường/ma sát, khả năng phanh, tốc độ đi bộ và miền tốc độ xe hợp lý. hunt.json: cuộc săn bus_bypass có 7 mẫu bất khả thi, cut_in 2, crosswalk_block 7, rear_ended 6. Nếu tính mẫu bất khả thi là va chạm, CEM bị kéo về những tình huống không thể xảy ra và xác suất thất bại bị thổi phồng; cần loại trước khi cập nhật/đếm.

## 3. Monte Carlo (bước C1)

mc.json: crashes=3, n=3000, p=0.001, Wilson 95%=[0.000340, 0.002936]. Muốn nửa độ rộng khoảng xấp xỉ 20% p cần n≈96/p≈96,000 lần chạy (công thức rubric). cut_in và rear_ended không xuất hiện trong 3000 mẫu tự nhiên; với 0 hit, cận trên 95% theo quy tắc 3/n là khoảng 0.001 (mc.json per-mode zero-hit bounds), nên 0 không có nghĩa là xác suất bằng 0.

## 4. Importance sampling (bước C3)

estimate.json, mỗi seed chạy n=600 cho mỗi kiểu; giá trị p / hits / ESS / top_share:
| Seed | Kiểu | p | hits | ESS | top_share |
|---|---|---:|---:|---:|---:|
| 0 | bus_bypass | 0.000848 | 29 | 136.3 | 0.200 |
| 0 | cut_in | 0 | 0 | 124.6 | 0 |
| 0 | crosswalk_block | 1.35e-6 | 60 | 105.1 | 0.326 |
| 0 | rear_ended | 0 | 0 | 128.1 | 0 |
| 1 | bus_bypass | 0.000776 | 22 | 145.3 | 0.458 |
| 1 | cut_in | 0 | 0 | 108.8 | 0 |
| 1 | crosswalk_block | 7.82e-7 | 64 | 108.8 | 0.253 |
| 1 | rear_ended | 0 | 0 | 121.7 | 0 |
| 2 | bus_bypass | 0.002847 | 38 | 135.5 | 0.546 |
| 2 | cut_in | 0 | 0 | 124.7 | 0 |
| 2 | crosswalk_block | 5.70e-7 | 64 | 100.9 | 0.388 |
| 2 | rear_ended | 0 | 0 | 112.0 | 0 |

Ba seed bất đồng đáng kể: bus_bypass dao động 0.000776–0.002847; crosswalk_block cũng đổi gần một bậc độ lớn. Seed 2 bus_bypass có top_share=0.546, tức hơn nửa ước lượng đến từ một mẫu có trọng số lớn. Điều này và ESS khoảng 101–145 trên 600 mẫu cho thấy độ lệch trọng số; top_share thấp cũng không chứng minh hội tụ. Cộng ước lượng loại với other từ estimate.json: seed 0 tổng=0.000849, seed 1=0.000777, seed 2=0.002847; không thể đòi khớp chính xác MC (mc.json p=0.001) vì bất định lớn và các kiểu được săn riêng. estimate.json budget=16800 rollout so với khoảng 96,000 rollout MC cần cho độ chính xác tương đối ±20%. other lấy từ mc.json (k=0/3000, upper zero-hit=0.001279), vì không có một cuộc săn riêng cho kiểu other và tổng các mode đã đặt tên không bao quát kiểu đó.

## 5. Kiểu va chạm tìm được (bước C2 + 3D)

hunt.json: bus_bypass tìm thấy 145, crosswalk_block tìm thấy 36; cut_in không được tìm thấy, rear_ended cũng không. Các cuộc săn riêng tìm các kiểu dễ bị mục tiêu hoá, không chứng minh kiểu vắng mặt nếu không tìm thấy. evidence/shots/q5_bus_bypass.png ghi lại cảnh lách qua xe buýt; evidence/shots/q5_crosswalk_block.png ghi C-01 tại 23,3 s, lúc xe ta chạm vùng người đi bộ trên vạch cạnh xe buýt.

## 6. Sửa planner (bước E)

fix.json chọn policy {cutinLat: 3}. TRAIN: fail=1.55e-6, fail_hi=4.16e-6, progress=212.14 m/24s, harsh=8.01. TEST: fail=3.07e-6, fail_hi_seen=6.22e-6 nhưng fail_hi=0.03879 do bus_bypass unseen có upper_zero_hits=0.0375; progress=214.38 m, harsh=7.86. Baseline TEST: fail=0.001432, fail_hi_seen=0.003976, fail_hi=0.005255, progress=214.49 m, harsh=8.18. fail_hi của policy mới lớn vì phạt cận cho mode chưa thấy; không phải ước lượng điểm. Trên TRAIN, cautionV=6 có fail_hi=0.001694 và tiến độ 188.68 m, kém hơn cautionV=6.5 (fail_hi=0.000156, tiến độ 198.25 m; fix.json: table). Đây là kết quả mẫu hữu hạn; thay đổi tốc độ làm đổi thời điểm ego phản ứng và khiến xe khác phản ứng lại, nên trực giác “chậm hơn luôn an toàn hơn” không được bảo đảm. evidence/shots/q6_policy_comparison.png cho thấy GB-01: lượt baseline đâm xe đạp lúc 13,6 s, còn bóng mờ của policy đã chọn không va chạm.

## 7. Giả định mô phỏng so với đường thật

Ba giả định chi phối kết quả: (1) phân phối tự nhiên space.py về khoảng trống xe nhập làn; cần thay bằng phân bố đo từ video dashcam đã gán nhãn theo thời điểm nhập và khoảng cách, sai lệch về gap nhỏ sẽ làm tỷ lệ cut-in nguy hiểm tăng/giảm theo hướng tương ứng; (2) xác suất người đi bộ nhìn qua xe buýt; cần khảo sát/video nhiều điểm sang đường, đánh dấu hành vi nhìn và che khuất, đánh giá thấp xác suất không nhìn sẽ đánh giá thấp rủi ro dart-out; (3) mu trên đường ướt; cần thử nghiệm phanh có kiểm soát hoặc dữ liệu đo ma sát mặt đường theo loại mặt đường/mưa, mu bị giả định quá cao làm đánh giá thấp va chạm và tính khả thi của phanh. Các mô hình hiện tại chỉ là giả định giảng dạy, không đại diện phân bố giao thông thật.

## 8. Giới hạn

Xác suất này chỉ cho một con phố/một thị trấn mô phỏng; hành vi xe khác là IDM và mô hình chuyển làn đơn giản; cảm biến gần như hoàn hảo ngoài vùng bị xe buýt che khuất. Nếu có thêm một tuần, tôi sẽ thu thập và gán nhãn log thực ở nhiều giao lộ, hiệu chỉnh phân phối tham số và che khuất, sau đó kiểm định ngoài mẫu trên nhiều tuyến đường và hành vi người lái.

## 9. Rắc rối tự dựng (bước G)

Rắc rối là xe đạp lạng tránh chướng ngại đột ngột ở mép làn (hazard.json: H0 tại x≈80 m); tình huống tương tự xe đạp tránh ổ gà/mảnh vụn. Giả thuyết là planner chỉ phản ứng khi vật thể vào hành lang, trong khi phanh/né muộn có thể không đủ khoảng cách. hazard.json: why ghi nhận tài liệu FHWA về nguy cơ ổ gà/mảnh vụn và Schleinitz et al. naturalistic cycling study (63 critical events/332 trips), đồng thời nói rõ các nguồn này không trực tiếp đo xác suất “swerve” của hazard cụ thể; exposure=0.01/chuyến và space (x 65–95m, trigger 4–10s, swerve 1.55–1.8m, swDur 1.9–2.4s) là miền sàng lọc/giả định ban đầu cần hiệu chỉnh tại địa phương, không phải tần suất đã được chứng minh. mine.json: accepted=0.992 (1488/1500); baseline có 1102 va chạm, gồm avoidable=636, unavoidable=463, unattributed=3, low_speed=348; P(avoidable)=0.4274, Wilson 95%=[0.4025,0.4527], tần suất suy ra 0.00427/chuyến khi nhân exposure 0.01. Policy chọn có 0/1488 va chạm, cận trên Wilson 0.002575, quy đổi per-trip upper=0.00002575. Trong mine.json, avoidable_foreknown=636: giả định biết trước thời điểm rắc rối là lạc quan; chỉ thử phanh mà không đánh lái có thể bỏ sót cách tránh thực tế, nên hai chiều sai lệch khác nhau. evidence/shots/q9_bicycle_hazard.png cho thấy Chương 10 đã có H0 xe đạp lạng ra cùng cảnh giao thông. Chưa có kết quả tự lái được lưu, nên chưa kết luận cá nhân lái tránh được. Nút cutinLat là ứng viên xử lý xe lấn làn nhưng không trực tiếp giải quyết vật thể xe đạp; planner cần dự đoán đường chuyển động vật thể nhỏ và so sánh cả phương án phanh lẫn đánh lái có kiểm tra vùng trống.
