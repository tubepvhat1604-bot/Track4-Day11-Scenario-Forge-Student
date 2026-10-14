MSSV: 202602507

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **B** (MSSV 202602507, chạy đầy đủ, không `--quick`).

## 1. Phát lại log có trung thực không? (bước A)

Trong `replay.json`: bao nhiêu kịch bản bị bỏ vì người lái gây va chạm, bao nhiêu va chạm khi planner chạy trên log phát lại, bao nhiêu trong thế giới phản ứng, theo từng loại tiếp xúc. Giải thích vì sao phần lớn va chạm của "phát lại" là `rear`, và vì sao đó là va chạm **giả**. Chọn một kịch bản trong `replay_only` và nói chuyện gì đã xảy ra.

**Trả lời.**

| | số |
|---|---|
| kịch bản dùng (`replay.json: n_used`) | 400 (bỏ 0 log có người lái va chạm, `dropped_human_crash_logs = 0`) |
| va chạm khi planner chạy trên log phát lại (`replay.crashes`) | **111**: `rear` 103, `side` 8, `front` 0 |
| va chạm trong thế giới phản ứng (`reactive.crashes`) | **0** |
| va chạm ở cả hai (`crash_in_both`) | 0 |

Gần hết va chạm "phát lại" là `rear`: planner của xe ta thận trọng hơn người lái thật (T = 1,6 s, chậm lại gần vạch bị che), nên nó phanh sớm hơn hoặc đi chậm hơn người lái trong log. Nhưng xe bám đuôi TG trong log chỉ chạy lại đúng quỹ đạo đã ghi khi người lái cũ chưa phanh, nó **không biết** xe ta đã phanh sớm nên cứ thế lao vào đuôi. Ngoài đời, TG sẽ thấy đèn phanh và phanh theo; trong thế giới phản ứng không có vụ nào. Vì vậy đây là va chạm **giả**: lỗi của cách đánh giá bằng log, không phải của planner.

Ví dụ `replay_only[0]` (kịch bản 1): TG bám rất sát (`tgT = 0.68 s`), xe ta giảm tốc trước vạch qua đường sớm hơn người lái thật, TG phát lại vẫn giữ tốc độ cũ và đâm đuôi (`contact = rear`). Trong thế giới phản ứng, cùng kịch bản đó TTC nhỏ nhất vẫn còn 2,16 s (`min_ttc_reactive`), không có va chạm.

## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

**Trả lời.**

Các luật trong `lab11/feasible.py: is_feasible`:

| luật | căn cứ vật lý |
|---|---|
| a_lat = (π/cutDur)²·3,5/2 > μg thì loại | chuyển làn 3,5 m theo hình cos cần gia tốc ngang đỉnh này; lốp chỉ cho tối đa μg (μ = 0,8 khô, 0,5 ướt) trước khi trượt |
| `cutBrake` > μg thì loại | phanh mạnh hơn ma sát lốp–đường là không thể (bánh bị khoá, trượt) |
| `pedV` > 2,5 m/s thì loại | đi bộ bình thường ~1,2–1,5 m/s; trên 2,5 m/s là chạy, không còn là "người đi bộ qua đường" của mô hình |
| `cutV` ngoài (0, 22] m/s thì loại | xe chuyển làn phải đang chạy, và 22 m/s (~80 km/h) là trần hợp lý cho phố nội đô này |

Cuộc săn sinh nhiều kịch bản bất khả thi nhất là **`crosswalk_block`: 43** (`hunt.json: crosswalk_block.infeasible`), so với 3, 3, 5 ở ba cuộc săn kia. Lý do: đề xuất CEM cuối cùng của nó đẩy `wet` lên p = 0,97 (`hunt.json: crosswalk_block.proposal.p_bin[0]`), vì đường ướt (μ = 0,5) làm quãng phanh dài ra nên xe ta dễ dừng đè lên vạch. Nhưng với μ = 0,5, một lần chuyển làn hơi nhanh của xe C2 đã vượt 0,5g ngang: lấy mẫu 4 000 kịch bản từ đề xuất này, 77 vi phạm luật a_lat, không vi phạm luật nào khác (mình tính thêm, không nằm trong `evidence/`). Tức là CEM bị kéo về phía "nguy hiểm nhất", và vùng nguy hiểm nhất lại giáp với vùng vật lý không cho phép.

Nếu **đếm** chúng là va chạm, ước lượng bị thổi phồng bằng những thất bại không thể xảy ra ngoài đời, và CEM còn bị kéo sâu hơn vào vùng bất khả thi (vì điểm nguy hiểm ở đó cao), nên cuộc săn tìm ra "lỗi" của vật lý chứ không phải lỗi của planner.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k, n, p̂ và khoảng Wilson 95 %. Với p̂ đó, cần bao nhiêu lần chạy để khoảng tin cậy hẹp lại còn ±20 % của p? Có kiểu thất bại nào **chưa hề xuất hiện** trong 3 000 lần chạy không? Con số đúng cần báo cho kiểu đó là gì?

**Trả lời.**

`mc.json`: **k = 5, n = 3 000, p̂ = 1,67·10⁻³, Wilson 95 % = [7,12·10⁻⁴; 3,90·10⁻³]**. Cả 5 vụ đều là `bus_bypass` (`mc.json: modes`). Có 15 kịch bản bất khả thi bị bỏ (`infeasible`).

Để khoảng tin cậy còn ±20 % của p: 1,96·√(p(1−p)/n) = 0,2p, nên n ≈ 96/p = 96 / 1,67·10⁻³ ≈ **57 600 lần chạy**, gấp khoảng 19 lần ngân sách hiện tại.

Có, ba kiểu **chưa xuất hiện lần nào** trong 3 000 lần: `cut_in`, `crosswalk_block`, `rear_ended` (và `other`), mỗi kiểu k = 0. Không được báo 0. Con số đúng là cận trên Wilson 95 %: **P < 1,28·10⁻³** (`mc.json: per_mode.*.hi`), cùng cỡ quy tắc ba 3/n = 10⁻³.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

**Trả lời.**

`estimate.json` (n = 600 mỗi kiểu mỗi seed):

| kiểu | seed | p | hits | ess | top_share |
|---|---|---|---|---|---|
| bus_bypass | 0 | 1,31·10⁻³ | 46 | 140 | 0,22 |
| bus_bypass | 1 | 1,30·10⁻³ | 41 | 124 | 0,32 |
| bus_bypass | 2 | 1,29·10⁻³ | 37 | 119 | 0,40 |
| crosswalk_block | 0 | 4,4·10⁻⁶ | 119 | 103 | 0,24 |
| crosswalk_block | 1 | 1,7·10⁻⁶ | 95 | 123 | 0,26 |
| crosswalk_block | 2 | 3,1·10⁻⁶ | 98 | 127 | 0,47 |
| cut_in | 0/1/2 | 0 (cận trên `upper_zero_hits` = 0,025) | 0 | 135/112/132 | 0 |
| rear_ended | 0/1/2 | 0 (cận trên 0,025) | 0 | 114/118/132 | 0 |

Tổng (`seeds[*].total`): 1,32·10⁻³ / 1,30·10⁻³ / 1,29·10⁻³.

- **Ba seed có đồng ý không?** Với `bus_bypass` thì rất khớp (1,29–1,31·10⁻³). Với `crosswalk_block` chênh khoảng 2,6 lần (1,7 đến 4,4·10⁻⁶), đúng kiểu dao động README cảnh báo.
- **Với Monte Carlo?** MC cho 5/3 000 = 1,67·10⁻³, Wilson [0,71; 3,90]·10⁻³. IS `bus_bypass` ~1,3·10⁻³ nằm gọn trong khoảng đó. MC không thấy `crosswalk_block` nào, phù hợp với IS cỡ 10⁻⁶ (cần cỡ 10⁶ lần chạy mới thấy một vụ).
- **`top_share` > 0,5?** Trong `estimate.json` không ô nào vượt 0,5 (lớn nhất 0,47 ở `crosswalk_block` seed 2). Nhưng ở `fix.json: baseline_test`, `crosswalk_block` có **top_share = 0,9999**: gần như toàn bộ p = 1,07·10⁻² là trọng số của **một** mẫu. Khi đó con số chỉ là "w của mẫu đó / n", không phải trung bình của nhiều quan sát, và khoảng ±1,96·se ([0; 3,2·10⁻²]) không đáng tin. Nó cũng lệch hơn 1 000 lần so với 1,7–4,4·10⁻⁶ ở đây, đúng là dấu hiệu của bẫy 2 và 3 trong README.
- **Ngân sách**: IS dùng `budget = 16 800` lần chạy cho cả 4 kiểu × 3 seed (gồm cuộc săn). Với `crosswalk_block`, sai số tương đối của một seed khoảng se/p ≈ 0,37; Monte Carlo muốn cùng độ chính xác cần n ≈ 1/(p·0,37²) ≈ 1/(4,4·10⁻⁶·0,137) ≈ **1,7 triệu lần chạy**, tức khoảng 100 lần ngân sách IS. Với `bus_bypass` (p ~1,3·10⁻³, se/p ≈ 0,32) MC chỉ cần ~7 500 lần, nên IS không lợi nhiều cho kiểu "không quá hiếm".
- **Phần `other`** lấy từ Monte Carlo C1 (`estimate.json: other`, `source = mc.json`): k = 0/3 000, cận trên 1,28·10⁻³. Nó không được săn vì "other" không phải một cơ chế: không có điểm nguy hiểm nào để CEM leo lên, gom mọi thứ còn lại thì cuộc săn chẳng có hướng.

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

**Trả lời.**

Kiểu tìm thấy (`hunt.json: *.found`):

- **`bus_bypass`** (cuộc săn riêng tìm 145 vụ). Xe buýt đỗ ở làn E2 chắn đường của C2. C2 muốn lách qua xe buýt nên lấn sang làn E1 của xe ta với tốc độ thấp (< 6 m/s, x < 112 m). Planner chỉ coi C2 là "xe phía trước" khi C2 lấn quá ngưỡng `cutinLat = 2,15 m`, nên phản ứng muộn và đâm vào hông/đuôi C2.
  [ẢNH: `evidence/shots/bus_bypass.png`, chương 9, kéo đến đúng lúc va chạm]
- **`crosswalk_block`** (cuộc săn riêng tìm 109 vụ). Đường ướt, người đi bộ P1 không ngó qua xe buýt (pedLook gần 0) và bước ra từ sau xe buýt với khe thời gian ngắn. Xe ta chỉ thấy P1 khi P1 đã ra khỏi bóng xe buýt, phanh gấp nhưng trên đường ướt dừng lại khi thân xe đã nằm trên vạch, P1 phải đứng lại.
  [ẢNH: `evidence/shots/crosswalk_block.png`, kéo đến lúc chặn lối]

**Không tìm thấy:** `cut_in` và `rear_ended`. Cuộc săn `cut_in` chỉ ra 31 `bus_bypass` + 1 `crosswalk_block`, cuộc săn `rear_ended` ra 3 `bus_bypass` + 1 `crosswalk_block`, không vụ nào đúng kiểu của mình. Mình tin ở mức vừa phải rằng chúng hiếm (cận trên IS 0,025 và MC < 1,28·10⁻³), nhưng không tin rằng chúng bằng 0: CEM chỉ là tìm kiếm cục bộ, và `cut_in`/`rear_ended` đều có `stopped_tightening = false` (điểm nguy hiểm còn chưa tới 0).

**Nếu chỉ chạy một cuộc săn chung**, nó sẽ co về kiểu dễ va chạm nhất là `bus_bypass`: chính các cuộc săn nhắm `cut_in` và `rear_ended` đã trôi về `bus_bypass` (31 và 3 vụ). `crosswalk_block` chỉ được tìm thấy khi có cuộc săn riêng với điểm nguy hiểm riêng (khoảng phanh còn lại trước người đi bộ). Một cuộc săn chung sẽ bỏ sót nó hoàn toàn.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

**Trả lời.**

`fix.json`:

| | policy | fail | fail_hi | fail_hi_seen | quãng đường (m) | phanh gấp (frame/kịch bản) |
|---|---|---|---|---|---|---|
| TRAIN, gốc (`baseline_train`) | mặc định | 3,02·10⁻³ | 5,85·10⁻³ | 5,85·10⁻³ | 214,3 | 8,07 |
| TRAIN, chọn (`chosen`) | `cutinLat = 3.3` | 5,2·10⁻⁷ | 8,6·10⁻⁷ | 8,6·10⁻⁷ | 214,2 | 7,34 |
| **TEST, chọn** (`chosen_test`) | `cutinLat = 3.3` | **1,24·10⁻⁶** | **3,88·10⁻²** | 1,94·10⁻⁶ | 214,1 | 7,89 |
| **TEST, gốc** (`baseline_test`) | mặc định | **1,48·10⁻²** | **4,33·10⁻²** | 4,20·10⁻² | 214,2 | 8,43 |

Policy được chọn trên TRAIN là `cutinLat = 3.3`: coi C2 là xe phía trước sớm hơn (khi C2 mới lấn 3,3 m thay vì 2,15 m). Quãng đường gần như không đổi (214,1 so với 214,2 m), số frame phanh gấp giảm nhẹ.

Trên TEST, `bus_bypass` là kiểu `unseen` (0 vụ sau khi săn lại riêng cho policy mới), nên `fail_hi` phải cộng cận `upper_zero_hits` = 3/(0,2·400) = **0,0375**. Vì thế `fail_hi` của bản sửa (3,88·10⁻²) **lớn hơn `fail` của policy gốc** (1,48·10⁻²) dù bản sửa thật sự tốt hơn: với n = 400, dữ liệu không thể chứng nhận một kiểu đã biến mất dưới ~10⁻², chỉ có thể nói "dưới 3,75 %". `fail_hi_seen` (1,94·10⁻⁶) mới là cái dữ liệu cho thấy, nhưng nó không phải cận hợp lệ.

[ẢNH: `evidence/shots/fix_ghost.png`, chương 9, một kịch bản `bus_bypass` có **bóng mờ** (policy gốc đâm C2, bản sửa giảm tốc kịp)]

**Núm "nghe thì an toàn hơn" nhưng phản tác dụng** (phần giải thích là suy luận từ cách thế giới phản ứng, nên kiểm tra lại bằng trình xem): `T = 2.0` (giữ khoảng cách xa hơn với xe trước) có TRAIN `fail` = **4,56·10⁻³ (hi 1,08·10⁻²)**, tệ nhất trong cả bảng và tệ hơn policy gốc (3,02·10⁻³). Lý do là thế giới phản ứng: khe trống lớn hơn phía trước xe ta là khe mà C2 (chuyển làn có *gap acceptance*) và xe lách xe buýt chấp nhận để chen vào. Planner giữ khoảng cách rộng thì mời thêm xe chen vào ngay trước mũi, ở khoảng cách ngắn, nên va chạm tăng. Tương tự `cautionV = 6.0` (chậm hơn gần vạch) có fail 1,27·10⁻³ nhưng mất 10 % quãng đường, và đi chậm hơn kéo dài thời gian xe ta nằm cạnh xe buýt, tức thời gian C2 có thể lách ra.
(Số TRAIN của các ứng viên khác lấy từ bảng in ra khi chạy `fix`, cũng được lưu trong `fix.json`.)

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

**Trả lời.**

| giả định trong `space.py` | dữ liệu thật để thay | lấy từ đâu | nếu sai thì con số sai theo hướng nào |
|---|---|---|---|
| `cutGap`: khe mà xe chuyển làn chấp nhận (log-normal, trung vị 12 m) | phân phối khe chấp nhận khi chuyển làn / lách xe đỗ | dữ liệu quỹ đạo như **highD / NGSIM / Waymo Open Motion**, hoặc camera trên cao ở các điểm dừng xe buýt | nếu tài xế thật chấp nhận khe nhỏ hơn (giao thông Việt Nam, xe máy), `bus_bypass` và `cut_in` bị **đánh giá thấp** |
| `pedLook = 0.85`: 85 % người đi bộ ngó qua xe buýt trước khi qua đường | tỉ lệ người đi bộ kiểm tra trước khi bước ra từ sau xe đỗ | quan sát video ở điểm dừng xe buýt có vạch qua đường; nghiên cứu hành vi người đi bộ (ví dụ dữ liệu inD/ Euro NCAP AEB pedestrian scenarios) | nếu ít người ngó hơn, `crosswalk_block` (đang ~10⁻⁶) bị **đánh giá thấp**, có thể nhiều bậc độ lớn |
| `wet = 0.20` và μ = 0,5 khi ướt | tỉ lệ thời gian đường ướt và hệ số bám thực tế | số liệu khí tượng địa phương (số ngày mưa), đo μ bằng xe đo ma sát / dữ liệu ABS của đội xe | ở nơi mưa nhiều (TP.HCM mùa mưa) tỉ lệ ướt cao hơn nhiều, nên `crosswalk_block` bị **đánh giá thấp**; nếu μ ướt thật cao hơn 0,5 thì ngược lại |

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

**Trả lời.**

1. **Một con phố, một xe buýt, một vạch qua đường.** Không có giao lộ có đèn, không có rẽ trái, không có xe máy (thứ chiếm phần lớn giao thông Việt Nam). P(va chạm) ~1,3·10⁻³ chỉ là xác suất trên đúng con phố 24 giây này, không phải "mỗi km" hay "mỗi chuyến".
2. **Người lái khác là IDM + gap acceptance có tham số cố định.** Không có người lái mất tập trung, vượt đèn, đổi ý giữa chừng, hay phản ứng chậm thật sự. Rủi ro thật chủ yếu nằm ở đuôi hành vi mà mô hình này không có.
3. **Cảm biến hoàn hảo trừ bóng xe buýt.** Xe ta biết chính xác vị trí, tốc độ mọi thứ nó "thấy"; không có nhiễu, trễ, nhận dạng nhầm hay mưa làm giảm tầm nhìn. Chỉ xe buýt che tầm nhìn, xe hơi khác thì không.

Nếu có thêm một tuần: chạy IS với ngân sách lớn hơn cho `crosswalk_block` để các seed bớt chênh, thêm nhiễu/độ trễ cảm biến và đo xem `bus_bypass` tăng bao nhiêu, và thử thay `pedLook`, `cutGap` bằng phân phối khớp từ một bộ dữ liệu quỹ đạo thật để xem con số dịch chuyển bao nhiêu.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** bạn dựng gì, và ngoài đời nó giống tình huống nào.
- **Căn cứ:** vì sao bạn nghĩ planner sẽ xử lý kém.
- **Giả định:** `space` và `exposure` lấy từ đâu (`hazard.json: why`).
- **Kết quả**, theo `mine.json`:
  - số va chạm `avoidable` (tránh được bằng phanh hết cỡ, biết trước), `unavoidable`, `unattributed`, và `low_speed`, với policy gốc và với policy bạn chọn; `accepted` (bao nhiêu phần `space` của bạn là khả thi);
  - P(lỗi planner | rắc rối) kèm khoảng Wilson;
  - tần suất mỗi chuyến, đặt cạnh tần suất lách xe buýt của bạn.
- **Hình:** một ảnh chương 9 hoặc 10 ở lúc va chạm.
- **Bạn tự lái thì sao:** nếu bạn đã thử lái, bạn tránh được không, và bạn làm gì khác planner?
- **Cách sửa:** núm chỉnh nào sửa được rắc rối này? Nếu không núm nào sửa được, planner cần thêm **logic gì** (mô tả, không cần code)?

**Trả lời.**

- **Rắc rối:** một người đi bộ **băng ngang ngoài vạch, bước ra từ ngay sau xe buýt đang đỗ** (hazard `ped`, phía S, x trong [92; 118] m, tức quanh thân xe buýt ~x 100–112 và trước vạch ở x = 122). Ngoài đời đây là kiểu "dart-out": khách vừa xuống xe buýt đi tắt qua đường từ đầu xe, không đi ra vạch.
- **Căn cứ:** planner chỉ giảm tốc "thận trọng" (`cautionV`) khi gần **vạch** bị che; ở đoạn trước vạch nó vẫn chạy ~12 m/s, và người đi bộ ở sau xe buýt thì không nhìn thấy được cho tới khi bước ra.
- **Giả định (`hazard.json: why`):** `h0_v` ∈ [1,2; 2,5] m/s từ đi bộ thường (~1,2–1,4 m/s, Knoblauch và cs. 1996) đến bước vội; `h0_trig` ∈ [8; 30] m nghĩa là người bước ra khi xe còn cách 8–30 m (khe ~0,7–2,5 s ở 12 m/s, ngắn hơn khe 3–4 s người đi bộ thường chấp nhận, vì họ không thấy xe). `exposure = 0.02` mỗi chuyến là ước lượng thô: ~5 điểm dừng xe buýt/chuyến × 40 % có xe buýt đỗ × 1 % có người lao ra đúng lúc. Đây là con số cần thay bằng dữ liệu quan sát thật.
- **Kết quả** (`mine.json`, 1 493 phiên bản khả thi trên 1 500, `accepted` = **99,5 %**; cảnh gốc dựng trong chương 10: planner đâm H0 lúc 15,5 s, `exact.planner`):

| | policy gốc (`baseline`) | policy sửa `cutinLat = 3.3` (`chosen`) |
|---|---|---|
| va chạm của xe ta | 38 | 39 |
| `avoidable` (tránh được bằng phanh hết cỡ ngay khi thấy) | **33** | **38** |
| `unavoidable` | 1 | 1 |
| `unattributed` (va chạm thường của thị trấn) | 4 | 0 |
| `low_speed` (< 1 m/s) | 2 | 2 |
| `avoidable_foreknown` | 34 | 39 |
| P(lỗi planner \| rắc rối) = avoidable/n | **2,21·10⁻²** [1,58·10⁻²; 3,09·10⁻²] | **2,55·10⁻²** [1,86·10⁻²; 3,47·10⁻²] |
| tần suất mỗi chuyến (× exposure 0,02) | **4,4·10⁻⁴** (cận trên 6,2·10⁻⁴) | 5,1·10⁻⁴ (cận trên 6,9·10⁻⁴) |

  So với tần suất lách xe buýt (`estimate.json: bus_bypass` ≈ 1,3·10⁻³ mỗi lần đi qua cảnh), rắc rối này cỡ **1/3** con số đó, cùng bậc độ lớn, trong khi thị trấn mặc định không hề có nó. Bản sửa ở bước E **không cứu** được (khoảng Wilson của hai policy chồng nhau gần hết; `cutinLat` chỉ đổi cách xử lý C2, không liên quan người đi bộ).

  Gần như mọi va chạm đều `avoidable`: phanh hết cỡ ngay khi người đi bộ vừa lộ ra khỏi xe buýt thì tránh được. Đây là chuẩn **lạc quan** theo hai chiều: (1) "người phanh" giả định biết ngay đó là nguy hiểm và phanh tối đa không trễ, điều một planner thật (có trễ nhận dạng, phải phân biệt người đứng chờ với người bước ra) không làm được, nên `avoidable` **đếm nhiều hơn** số vụ planner thật tránh được; (2) ngược lại chỉ thử phanh, không thử đánh lái tránh, nên có vụ đánh lái cứu được bị đếm là `unavoidable`, làm `avoidable` **đếm ít đi**. Khoảng cách `avoidable_foreknown` − `avoidable` = 1 vụ cho thấy cái giá của bóng che là nhỏ ở đây: vấn đề chính là planner phản ứng chậm sau khi đã thấy.
- **Hình:** [ẢNH: `evidence/shots/hazard.png`, chương 10 mở `hazard.json` (hoặc chương 9 nhóm "mine"), lúc xe ta chạm người đi bộ]
- **Bạn tự lái thì sao:** Mình tránh được va chạm khi tự lái, phanh và dừng cách người đi bộ khoảng 2 m. Mình đã biết trước kịch bản nên chủ động phanh sớm hơn planner.
- **Cách sửa:** không núm nào trong 9 núm sửa đúng chỗ: `cautionV`/`cautionRange` chỉ áp dụng gần vạch bị che, `pedBuffer` chỉ dùng khi đã thấy người. Planner cần **logic mới**: coi mọi xe lớn đang đỗ sát làn mình (xe buýt, xe tải) là một vùng che có thể có người lao ra, và giảm tốc tới mức dừng được trong khoảng cách tới mép xe đỗ (tốc độ ≤ √(2·μg·d) với d là khoảng cách tới điểm khuất), giống cách nó đã làm với vạch bị che, cộng thêm phanh khẩn cấp (gần μg) ngay khi phát hiện người đi bộ cắt vào làn thay vì phanh thoải mái `b = 2 m/s²`.
