MSSV: 202602670

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): **D** (khô hơn: wet 0,15; người đi bộ ngó qua xe buýt 0,90; xe chuyển làn phanh sau khi vào làn 0,20; khe chấp nhận trung vị 14 m, theo `avsim/space.py`).

## 1. Phát lại log có trung thực không? (bước A)

`replay.json` (n = 400): `dropped_human_crash_logs` = 0 (không log nào có chính người lái gây va chạm). Planner trên **log phát lại**: `replay.crashes` = 100, theo loại tiếp xúc `rear` = 93, `side` = 7. Trong **thế giới phản ứng**: `reactive.crashes` = 1 (`side` = 1). `crash_in_both` = 0.

Phần lớn va chạm của "phát lại" là `rear` và là va chạm **giả**: xe bám đuôi trong log chạy đúng như đã ghi theo nhịp phanh của người lái thật; planner phanh sớm/khác người lái nên xe phía sau (không biết planner đã phanh) cứ đi theo quỹ đạo cũ và húc vào đuôi. Trong thế giới phản ứng, xe bám đuôi phanh lại nên không có vụ nào. Ví dụ `replay_only[0]` (scenario 0): va chạm `rear` ở phát lại, còn trong thế giới phản ứng `min_ttc_reactive` = 7,46 s, tức không có tình huống nguy hiểm nào cả.

## 2. Kịch bản bất khả thi (bước B)

Luật trong `feasible.is_feasible` (từ chối nếu một trong các điều kiện sau):
- đổi làn cần a_lat = (π/cutDur)²·3,5/2 > μ·g: lốp không đủ bám để quẹo gắt như vậy;
- phanh sau khi vào làn `cutBrake` > μ·g: không ai phanh mạnh hơn ma sát cho phép;
- người đi bộ `pedV` > 2,5 m/s: nhanh hơn chạy bộ thông thường nhiều;
- tốc độ vào làn `cutV` ngoài (0, 22] m/s: đứng yên hoặc nhanh quá mức cho con phố này.

`hunt.json`: số kịch bản bất khả thi bị bỏ qua (`infeasible`): `bus_bypass` 1, `cut_in` 0, `crosswalk_block` **8**, `rear_ended` 5. Cuộc săn `crosswalk_block` sinh nhiều nhất, vì điểm nguy hiểm (khoảng phanh còn lại trước người đi bộ) lớn nhất ở những kịch bản phi vật lý: người đi bộ chạy quá nhanh hoặc xe chuyển làn quẹo/phanh quá giới hạn ma sát, nên CEM bị kéo về đó. Nếu đếm chúng như va chạm thì ước lượng bị thổi phồng bằng những thất bại không thể xảy ra ngoài đời.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k = `crashes` = 5, n = 3000, p̂ = 1,67e-3, Wilson 95 % [7,12e-4; 3,90e-3] (cả 5 vụ là `bus_bypass`). Để khoảng hẹp còn ±20 % của p cần n ≈ 96/p̂ ≈ 57 600 lần chạy (gấp ~19 lần số đã chạy). Các kiểu **chưa hề xuất hiện** trong 3 000 lần: `cut_in`, `crosswalk_block`, `rear_ended`, `other` (k = 0, `per_mode`). Số đúng cần báo cho mỗi kiểu là cận trên Wilson `hi` = 1,28e-3, không phải 0. (`mc.json: infeasible` = 15 kịch bản bất khả thi đã bị bỏ.)

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json`, n = 600 mỗi seed (`hits` = số mẫu có va chạm đúng kiểu):

| seed | kiểu | p | hits | ess | top_share |
|---|---|---|---|---|---|
| 0 | bus_bypass | 5,78e-4 | 27 | 117 | 0,43 |
| 0 | crosswalk_block | 1,72e-6 | 33 | 131 | 0,39 |
| 1 | bus_bypass | 2,69e-4 | 27 | 120 | 0,34 |
| 1 | crosswalk_block | 1,11e-7 | 22 | 118 | 0,36 |
| 2 | bus_bypass | 2,36e-4 | 20 | 135 | **0,70** |
| 2 | crosswalk_block | 1,41e-7 | 18 | 123 | **0,55** |

`cut_in` và `rear_ended`: 0 hits ở cả ba seed, báo `upper_zero_hits` = 0,025 (= 3/(0,2·n)), không báo 0. `total`: 5,79e-4 / 2,69e-4 / 2,36e-4 (seed 0/1/2).

Ba seed đồng ý về bậc độ lớn của `bus_bypass` (chênh ~2,5 lần) nhưng **không** đồng ý ở `crosswalk_block` (1,7e-6 so với 1,1e-7: chênh ~15 lần). So với Monte Carlo (`bus_bypass` 1,67e-3): IS thấp hơn 3–7 lần, đúng chiều "thường thấp hơn" README cảnh báo (còn nằm trong khoảng tin cậy MC, nhưng MC chỉ có 5 vụ). `top_share` > 0,5 ở seed 2 (`bus_bypass` 0,70; `crosswalk_block` 0,55): khi đó một mẫu nặng nhất mang hơn nửa ước lượng nên con số thực chất là trọng số của một mẫu, `ess` ~ 120 không phản ánh điều đó. `budget` = 16 800 lần chạy cho cả IS. Để MC có độ chính xác ±20 % ở p ~ 3e-4 cần ~96/3e-4 ≈ 320 000 lần; với `crosswalk_block` ~1e-7 cần ~1e9: MC không làm nổi. Phần `other` = 0 (k = 0/3000, cận trên 1,28e-3) lấy từ `mc.json` (`estimate.json: other.source`), vì `other` không có cuộc săn nào nhắm vào (không có điểm nguy hiểm cho nó).

## 5. Các kiểu thất bại (bước C2 + 3D)

- **bus_bypass** (`evidence/shots/B-01.png`, run B-01, va chạm 12,6 s): xe cắt làn C2 chen qua xe buýt đang đỗ với tốc độ thấp; xe ta bị xe buýt/làn che, không dự đoán C2 sẽ lách ra, và va chạm vào C2 (`ego`–`C2`, `contact`).
- **crosswalk_block** (`evidence/shots/C-01.png`, run C-01, chặn lối 20,8 s): người đi bộ bước ra từ sau xe buýt đỗ; xe ta phanh quá muộn, dừng với thân xe nằm trên vạch qua đường, người đi bộ phải đứng lại (`kind = block`).
- **cut_in** và **rear_ended**: cuộc săn **không** tìm thấy (`hunt.json: found` của hai cuộc săn này lần lượt là `{bus_bypass: 8}` và `{bus_bypass: 3}`, không có `cut_in` hay `rear_ended` nào). Mức tin: thấp. Chỉ là cận trên (`upper_zero_hits` = 0,025 trên mỗi seed IS; 1,28e-3 từ MC), không phải bằng chứng là chúng không tồn tại.

Nếu chỉ chạy một cuộc săn chung: cả bốn cuộc săn đều bị kéo về `bus_bypass` (kiểu dễ nhất): `found` của săn `cut_in` và `rear_ended` toàn `bus_bypass`, chỉ cuộc săn riêng cho `crosswalk_block` mới tìm ra `crosswalk_block` (12 lần). Một cuộc săn chung sẽ co về `bus_bypass` và bỏ sót `crosswalk_block`.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn bằng TRAIN: `{cutinLat: 2.6}`. Trên TRAIN: `fail` = 1,21e-6, `fail_hi` = 3,48e-6 (kiểu `bus_bypass` là `unseen`); policy gốc TRAIN `fail` = 1,13e-4, `fail_hi` = 2,32e-4.

Trên **TEST** (seed mới + cuộc săn mới):

| | fail | fail_hi_seen | fail_hi (hợp lệ) | quãng đường | phanh gấp |
|---|---|---|---|---|---|
| chosen `cutinLat 2,6` | 4,01e-5 | 1,10e-4 | 1,39e-3 | 216,9 m | 7,63 |
| baseline | 4,07e-4 | 7,09e-4 | 1,99e-3 | 216,9 m | 7,85 |

`fail` của bản sửa giảm khoảng 10 lần (4,07e-4 → 4,01e-5). Nhưng `fail_hi` của bản sửa (1,39e-3) vẫn lớn hơn `fail` của policy gốc (4,07e-4): `fail_hi` là cận trên hợp lệ cho cả policy, gồm cận 3/(0,2·n) cho kiểu có thể đã biến mất và `other_crashes` (n = 3000, hi = 1,28e-3); với n = 400, không ngân sách nào ở lab chứng nhận được < 1e-2. Cận này là bài học, không phải lỗi. Cũng lưu ý: trên TRAIN `bus_bypass` là `unseen` (0 va chạm) nhưng trên TEST nó vẫn xuất hiện 3 hit: "0 va chạm trên TRAIN" không chứng minh gì.

Ảnh có bóng mờ: `evidence/shots/B-01.png` (run B-01). Policy được chạy trong ảnh là policy gốc (đã va chạm lúc 12,6 s); bóng mờ là bản sửa `cutinLat 2,6`, trên cùng kịch bản **không** va chạm (readout của trình xem hiển thị dòng "chính sách gốc: không va chạm" cho bóng mờ này, nhãn bị đảo vì run lưu theo policy gốc).

Núm "nghe an toàn hơn" nhưng làm tăng thất bại (TRAIN `fix.json`, so với baseline `fail_hi` 2,32e-4): `cautionV` 5,0 → `fail_hi` 4,82e-4 (quãng đường chỉ 173,8 m); `cautionV` 6,5 → 5,29e-4; `T` = 2,0 → 1,18e-3. Giải thích (giả thuyết, chưa tách riêng bằng thí nghiệm): thế giới phản ứng lại policy. Đi chậm/giữ khoảng cách lớn thì xe khác (C2 chấp nhận khe, người đi bộ chờ xe) gặp ego ở thời điểm/vị trí khác: khe trước mũi xe ta rộng hơn nên C2 dễ chen vào gần hơn và xe ta bị kẹt/chắn lâu hơn ở vạch qua đường; chạy chậm cũng đổi thời điểm xe ta gặp xe buýt/người đi bộ. Do đó "an toàn hơn" theo trực giác không tự động an toàn hơn.

## 7. Từ mô phỏng ra đường thật

Ba giả định ảnh hưởng nhiều nhất (đều là "giả định dạy học" theo `space.py`, không khớp dữ liệu thật):
1. **Khe chấp nhận của xe chuyển làn** (`cutGap`, lognormal trung vị 14 m ở thị trấn D). Dữ liệu thật: quỹ đạo xe thực (NGSIM / highD / Waymo Open Motion / log đội xe) để đo khe chấp nhận. Nếu người thật chen ở khe nhỏ hơn thì va chạm `bus_bypass`/`cut_in` thật **nhiều hơn** số của mình; nếu thận trọng hơn thì ít hơn.
2. **Tỉ lệ người đi bộ ngó qua xe buýt** (0,90 ở D). Dữ liệu: video giám sát vạch qua đường (thống kê hành vi người đi bộ ở ga xe buýt), khảo sát tuyến thực. Nếu thật sự ít người ngó (hay bước ra liều hơn) thì `crosswalk_block` **cao hơn**.
3. **Tần suất đường ướt / ma sát** (0,15 ở D; μ 0,5 so với 0,8). Dữ liệu: thống kê thời tiết và đo ma sát mặt đường tại tuyến, ví dụ hồ sơ khí tượng địa phương. Nếu mưa nhiều hơn thì mọi kiểu va chạm **tăng** (khoảng phanh dài hơn; ngưỡng khả thi `is_feasible` cũng hẹp lại).

## 8. Giới hạn

1. Thế giới chỉ có **một con phố** với một xe buýt, một vạch qua đường và một xe chuyển làn: xác suất không nói gì về ngã tư, đường cao tốc, xe máy, v.v.
2. Người lái khác chỉ là **IDM + gap acceptance**, cảm biến hoàn hảo trừ phần bị xe buýt che: không có nhiễu cảm biến, không có người lái hung hăng hay xao nhãng.
3. Ước lượng IS thường **thấp** và các seed lệch nhau (`crosswalk_block` chênh ~15 lần; `top_share` 0,70 ở seed 2); với n = 400 ở bước TEST, không chứng nhận được p < 1e-2 cho kiểu đã biến mất.

Nếu thêm một tuần: chạy IS nhiều seed hơn (≥ 10) với ngân sách lớn hơn và chạy ≥ 15 000 lần trên đúng vùng `crosswalk_block` để thu hẹp cận; và đo lại ba giả định bằng dữ liệu thật.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** `hazard.json`: một hazard `brake` trên xe cắt làn **C2** (phanh `decel` 6,3 m/s², `dur` 2,5 s, `t0` 4,6 s): xe vừa chen vào làn của mình thì phanh gấp ngay trước mũi xe ta. Ngoài đời giống tình huống xe chen vào rồi giảm tốc để rẽ/đỗ.
- **Căn cứ:** planner dùng IDM với khoảng cách an toàn cho xe đang ở trong làn; xe vừa vào làn có khoảng trống nhỏ, nên nếu nó phanh ngay thì thời gian phản ứng còn lại ngắn.
- **Giả định:** `space`: `h0_t0` [2; 8] s (quanh thời điểm C2 chen vào), `h0_decel` [3; 7,5] m/s² (dưới μ·g = 7,85), `h0_dur` [1; 5] s. `exposure` = 0,05 mỗi chuyến là **giả định của mình**, không dựa trên dữ liệu thật (`hazard.json: why`); cần dữ liệu lái xe tự nhiên để thay.
- **Kết quả** (`mine.json`, 1 347 phiên bản khả thi trên 1 500, `accepted` = 0,898):

| | crashes | avoidable | unavoidable | unattributed | low_speed | P(lỗi planner \| rắc rối) [Wilson] | mỗi chuyến (×0,05) |
|---|---|---|---|---|---|---|---|
| baseline | 5 | 5 | 0 | 0 | 0 | 3,71e-3 [1,59e-3; 8,66e-3] | 1,86e-4 (hi 4,33e-4) |
| chosen `cutinLat 2,6` | 2 | 2 | 0 | 0 | 0 | 1,48e-3 [4,07e-4; 5,40e-3] | 7,42e-5 (hi 2,70e-4) |

  Cảnh gốc (`exact`): planner va chạm C2 lúc 6,3 s. Tần suất mỗi chuyến của rắc rối này (baseline 1,86e-4) cùng bậc với `bus_bypass` của mình (IS ~2,4e-4 đến 5,8e-4; MC 1,67e-3 [7,1e-4; 3,9e-3]).
- **Hình:** `evidence/shots/GB-01.png` (chương 9, run GB-01 từ `viewer.json`, va chạm lúc 6,9 s).
- **Bạn tự lái thì sao:** chưa thử lái.
- **Phân loại:** `avoidable` = cùng planner nhưng phanh hết cỡ ngay khi thấy rắc rối thì không va chạm. Tất cả 5/5 va chạm baseline đều `avoidable` (`unavoidable` = 0, `unattributed` = 0). "Tránh được" là chuẩn **lạc quan**: người phanh biết trước và phanh tối đa nên `avoidable` đếm *nhiều hơn* số va chạm một planner thật tránh được (cận trên của lỗi planner), nhưng chỉ thử phanh (không đánh lái) làm nó đếm *ít* đi. Và `avoidable_foreknown` = `avoidable` (5 và 2) ở đây vì C2 không bị che.
- **Cách sửa:** `cutinLat` 2,6 giảm từ 5 xuống 2 va chạm nhưng hai khoảng Wilson chồng lấn mạnh nên chưa kết luận được là cải thiện thật, và không núm nào xử lý được hết. Cần thêm **logic**: nhận ra xe đang chen (độ lệch ngang đang giảm) và chủ động giãn khoảng cách / phanh nhẹ ngay khi nó bắt đầu vào làn (dự đoán xe sẽ ở trước mũi mình và có thể phanh), thay vì chỉ coi nó là "xe phía trước" khi đã vượt ngưỡng ngang.

---

Số từ lần chạy ngoài `evidence/`: không có. Dùng các tập lệnh phụ chỉ để thử nghiệm chọn hazard, không trích số từ đó.
