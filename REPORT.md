MSSV: 202602438

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): B

## 1. Phát lại log có trung thực không? (bước A)

Theo `replay.json`:
- Số kịch bản bị bỏ vì người lái gây va chạm: `dropped_human_crash_logs = 0` (sử dụng đủ `n_used = 400` kịch bản).
- Số va chạm khi planner chạy trên **log phát lại**: `replay.crashes = 127` (trong đó tiếp xúc `rear`: 114 vụ, `side`: 13 vụ, `front`: 0 vụ).
- Số va chạm trong **thế giới phản ứng**: `reactive.crashes = 2` (trong đó `side`: 1 vụ, `front`: 1 vụ, `rear`: 0 vụ).
- Cả hai cùng va chạm: `crash_in_both = 1`.

**Giải thích:** Phần lớn va chạm trong log phát lại là va chạm từ phía sau (`rear`: 114/127 vụ, chiếm ~90%). Đây là **va chạm giả** vì trong chế độ phát lại, các tác nhân phía sau (xe bám đuôi TG) chỉ chạy lại quỹ đạo ghi sẵn của người lái ban đầu mà không có phản hồi với hành vi của xe ta. Khi planner lái thận trọng hơn, giảm tốc sớm hơn người lái thật, xe bám đuôi không hề biết để phanh mà đâm thẳng vào đuôi xe ta. Trong thế giới phản ứng, xe sau lái theo IDM sẽ tự động phanh giữ khoảng cách an toàn, triệt tiêu toàn bộ va chạm đuôi.

**Ví dụ kịch bản `scenario: 0` trong `replay_only`:** 
Trong log phát lại xảy ra va chạm `rear`, nhưng trong thế giới phản ứng xe sau giữ được `min_ttc_reactive = 3.14 s` và hoàn toàn không va chạm. Người lái thật trước đó đi nhanh hơn, còn planner phanh chuẩn bị cho người đi bộ/vạch qua đường khiến log phát lại bị đâm từ phía sau.

## 2. Kịch bản bất khả thi (bước B)

Các luật trong `is_feasible` và căn cứ vật lý:
1. `a_lat = (pi / cutDur)^2 * 3.5 / 2 <= mu * g`: Xe chuyển làn theo đường cong cosine $3.5\text{ m}$. Gia tốc ngang cực đại không được vượt quá độ bám dính của ma sát lốp $\mu g$, nếu vượt quá xe sẽ bị trượt văng mất kiểm soát.
2. `cutBrake <= mu * g`: Gia tốc phanh dọc sau khi nhập làn bị giới hạn bởi ma sát giữa lốp và mặt đường $\mu g$ theo định luật Coulomb.
3. `pedV <= 2.5 m/s`: Tốc độ đi bộ của người qua đường tối đa $2.5\text{ m/s}$ ($\approx 9\text{ km/h}$). Vượt quá mức này là chạy nước rút, không phù hợp với mô hình người đi bộ đô thị.
4. `0 < cutV <= 22 m/s`: Tốc độ xe cắt làn phải dương và không vượt quá $22\text{ m/s}$ ($\approx 79.2\text{ km/h}$), giới hạn tốc độ tối đa cho phép trên đường đô thị.

**Cuộc săn sinh nhiều kịch bản bất khả thi nhất trong `hunt.json`:**
Cuộc săn **`crosswalk_block`** sinh nhiều kịch bản bất khả thi nhất với `infeasible = 16` kịch bản bị loại (so với `bus_bypass`: 5, `cut_in`: 7, `rear_ended`: 7). 
**Vì sao CEM bị kéo về phía đó:** Điểm nguy hiểm của `crosswalk_block` tối ưu hóa việc làm giảm khoảng cách phanh còn lại trước người đi bộ (`ped_room = ahead - v^2 / (2*mu*g)`). Để làm cho kịch bản nguy hiểm nhất có thể, CEM bị thúc đẩy đẩy vận tốc người đi bộ `pedV` lên cực đại (vượt $2.5\text{ m/s}$) và giảm hệ số ma sát $\mu$ xuống đường ướt ($0.5$), kéo theo các điều kiện chuyển làn khác cũng bị vi phạm $a_{\text{lat}} > \mu g$.

**Hậu quả nếu đếm chúng:** Ước lượng xác suất rủi ro $P(\text{crash})$ sẽ bị thổi phồng nghiêm trọng, làm sai lệch đánh giá an toàn của xe tự lái bằng những "bóng ma" tai nạn không bao giờ tồn tại trong thực tế vật lý.

## 3. Monte Carlo cho bạn biết gì (bước C1)

Theo `mc.json`:
- $n = 3000$
- $k = 2$ (`crashes = 2`)
- $\hat{p} = 6.67 \times 10^{-4}$ (`0.000666667`)
- Khoảng Wilson 95%: $[1.83 \times 10^{-4}, 2.43 \times 10^{-3}]$ (chính xác `lo = 0.000182839`, `hi = 0.002427685`).

**Số lần chạy cần thiết để khoảng tin cậy hẹp còn $\pm 20\%$ của $p$:**
Nửa độ rộng khoảng tin cậy là $z \sqrt{\frac{p(1-p)}{n}} = 0.20 p$.
Suy ra:
$$n \approx \frac{z^2}{(0.20)^2} \frac{1-p}{p} \approx \frac{1.96^2}{0.04} \frac{1}{p} \approx \frac{96.04}{p}$$
Với $\hat{p} = 6.67 \times 10^{-4}$:
$$n \approx \frac{96.04}{6.67 \times 10^{-4}} \approx 144\,060 \text{ lần chạy}$$
(Cần tới hơn 144.000 lần chạy mô phỏng).

**Kiểu thất bại chưa xuất hiện:**
Trong 3.000 lần chạy, chỉ có 2 vụ đều thuộc kiểu `bus_bypass`. Có tới 4 kiểu **chưa hề xuất hiện** ($k = 0$): `cut_in`, `crosswalk_block`, `rear_ended`, và `other`.

**Con số đúng cần báo:**
Tuyệt đối không được báo xác suất bằng 0. Khi $k = 0$, con số trung thực cần báo là **cận trên 95%** (theo khoảng Wilson hoặc quy tắc Rule of Three $3/n$):
- Cận trên Wilson 95% trong `mc.json: per_mode.<mode>.hi`: $p \le 1.28 \times 10^{-3}$ ($0.00128$).
(Tương ứng với quy tắc $3/n = 3/3000 = 1.0 \times 10^{-3}$). Báo $p \in [0, 1.28 \times 10^{-3}]$ với độ tin cậy 95%.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng kết quả từ `evidence/estimate.json` (ngân sách tổng `budget = 16800`):

| Kiểu thất bại | Seed | $p$ | Hits | ESS | Top share |
|---|---|---|---|---|---|
| **bus_bypass** | 0 | $6.56 \times 10^{-4}$ | 31 | 128.57 | 0.180 |
| | 1 | $3.93 \times 10^{-4}$ | 35 | 134.89 | 0.303 |
| | 2 | $9.78 \times 10^{-4}$ | 35 | 126.75 | 0.270 |
| **cut_in** | 0 | $0.0$ | 0 | 121.84 | 0.000 (cận $2.5 \times 10^{-2}$) |
| | 1 | $0.0$ | 0 | 117.01 | 0.000 (cận $2.5 \times 10^{-2}$) |
| | 2 | $0.0$ | 0 | 117.09 | 0.000 (cận $2.5 \times 10^{-2}$) |
| **crosswalk_block** | 0 | $2.12 \times 10^{-5}$ | 45 | 103.50 | **0.841** |
| | 1 | $1.62 \times 10^{-5}$ | 51 | 125.16 | **0.699** |
| | 2 | $1.21 \times 10^{-6}$ | 49 | 135.49 | **0.579** |
| **rear_ended** | 0 | $0.0$ | 0 | 141.66 | 0.000 (cận $2.5 \times 10^{-2}$) |
| | 1 | $0.0$ | 0 | 121.06 | 0.000 (cận $2.5 \times 10^{-2}$) |
| | 2 | $0.0$ | 0 | 113.56 | 0.000 (cận $2.5 \times 10^{-2}$) |

**Phân tích sự đồng thuận giữa các seed và với Monte Carlo:**
- **Với Monte Carlo:** Tổng xác suất qua 3 seed (Seed 0: $6.77 \times 10^{-4}$, Seed 1: $4.09 \times 10^{-4}$, Seed 2: $9.79 \times 10^{-4}$) có trung vị $6.77 \times 10^{-4}$, hoàn toàn khớp và nằm sâu trong khoảng Wilson 95% của Monte Carlo $[1.83 \times 10^{-4}, 2.43 \times 10^{-3}]$ ($\hat{p}_{MC} = 6.67 \times 10^{-4}$).
- **Giữa 3 seed:** `bus_bypass` rất ổn định (chênh giữa min-max chỉ ~2.5 lần). Tuy nhiên, `crosswalk_block` có sự chênh lệch lớn giữa Seed 0, 1 ($\approx 2 \times 10^{-5}$) và Seed 2 ($1.21 \times 10^{-6}$, chênh nhau gần 18 lần).

**Ở đâu `top_share` lớn (> 0.5) và nguyên nhân:**
- `top_share` vượt 0.5 ở kiểu `crosswalk_block` trên toàn bộ cả 3 seed (Seed 0: 0.841, Seed 1: 0.699, Seed 2: 0.579).
- **Vì sao con số chỉ do một mẫu quyết định:** Phân phối đề xuất $q$ sau các vòng săn CEM bị co quá hẹp tại một điểm va chạm cực đoan. Khi lấy mẫu hỗn hợp phòng thủ, thỉnh thoảng có một mẫu rơi vào vùng có $p(u) \gg q(u)$, làm trọng số $w_i = p(u)/q(u)$ tăng vọt. Mẫu đơn lẻ này mang tới 84.1% giá trị ước lượng của cả đợt, dẫn đến phương sai ước lượng lớn giữa các seed.

**So sánh ngân sách (`budget`):**
- Ngân sách IS: $16\,800$ lần chạy (bao gồm cả săn CEM và 3 seed IS).
- Để đo được xác suất hiếm cỡ $p \approx 2 \times 10^{-5}$ của `crosswalk_block` bằng Monte Carlo thuần túy với độ chính xác $\pm 20\%$, cần $n \approx 96 / p \approx 96 / (2 \times 10^{-5}) \approx 4.8 \times 10^6$ lần chạy (gần 5 triệu lần). IS tiết kiệm tài nguyên tính toán gần 300 lần.

**Phần `other` trong tổng lấy từ đâu và vì sao không săn:**
- Lấy từ kết quả Monte Carlo C1 (`source: mc.json`, $k=0, n=3000, p=0.0, hi=0.00128$).
- Không săn `other` vì đây là tập hợp các va chạm ngẫu nhiên ngoại lai không có cơ chế vật lý hay dấu hiệu dẫn đường cụ thể (không có hàm điểm nguy hiểm/proxy metric để hướng CEM vào).

## 5. Các kiểu thất bại (bước C2 + 3D)

Các kiểu thất bại tìm thấy qua các cuộc săn CEM trong `evidence/hunt.json`:
1. **`bus_bypass` (92 vụ tìm thấy trong cuộc săn `bus_bypass`):**
   - *Chuỗi sự kiện:* Xe buýt đỗ ở lề đường che khuất tầm nhìn; xe hàng xóm C2 đi chậm sau xe buýt ($x < 112\text{ m}, v < 6\text{ m/s}$) bất ngờ đánh lái lách sang làn của ego để vượt; xe ego do ngưỡng nhận diện ngang `cutinLat` mặc định hẹp ($2.15\text{ m}$) nên phản ứng phanh quá muộn, đâm vào sườn/góc C2. *(Ảnh chụp: `evidence/shots/q5_bus_bypass.png`)*.
2. **`crosswalk_block` (19 vụ tìm thấy trong cuộc săn `crosswalk_block`):**
   - *Chuỗi sự kiện:* Người đi bộ P1 bước ra vạch qua đường từ điểm mù sau xe buýt đỗ; xe ego phát hiện muộn và phanh khẩn cấp, tuy dừng lại được (vận tốc về 0) nhưng thân xe đã trượt đè lên vạch kẻ đường, chắn ngang lối đi của P1 (`kind: block`). *(Ảnh chụp: `evidence/shots/q5_crosswalk_block.png`)*.

**Kiểu cuộc săn KHÔNG tìm thấy:**
- Cuộc săn `cut_in` và `rear_ended` không tìm thấy bất kỳ va chạm nào đúng kiểu của chúng (`cut_in = 0`, `rear_ended = 0`).
- *Mức độ tin cậy:* Rất tin cậy. Trong thế giới phản ứng, xe sau (TG) tuân thủ IDM nên luôn tự động phanh giữ khoảng cách an toàn, không thể đâm đuôi ego trừ khi ego dừng tức thời (bất khả thi về vật lý). Với `cut_in` ở tốc độ thông thường trên đường thẳng, thuật toán IDM của ego hoàn toàn đủ thời gian phanh nhường nhịp nhàng khi xe kia chuyển làn.

**Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu thì điều gì xảy ra:**
- Như thấy rõ trong `hunt.json`, ngay cả khi được chỉ định mục tiêu riêng, các cuộc săn `cut_in` và `rear_ended` vẫn bị điểm nguy hiểm "kéo lệch" sang sinh ra 26 vụ và 10 vụ `bus_bypass`.
- Nếu chỉ chạy duy nhất 1 cuộc săn chung, phân phối đề xuất $q$ của CEM sẽ nhanh chóng bị **co cụm (mode collapse)** hoàn toàn vào kiểu va chạm dễ xảy ra nhất (`bus_bypass`), triệt tiêu độ biến thiên và bỏ sót hoàn toàn các kiểu va chạm hiếm khác như `crosswalk_block`. Việc săn riêng từng kiểu với hàm điểm mục tiêu chuyên biệt (TTC, ped_room, headway) là điều kiện bắt buộc để bao phủ toàn diện không gian lỗi.

## 6. Sửa planner (bước E + 3D)

Theo `evidence/fix.json`:
- **Policy được chọn:** `{'cutinLat': 3.0}` (tăng ngưỡng ngang nhận diện xe cắt làn từ mặc định $2.15\text{ m}$ lên $3.0\text{ m}$).
- **Trên tập TRAIN:**
  * `fail = 6.29e-06` ($6.287 \times 10^{-6}$), `fail_hi = 1.79e-05` ($1.790 \times 10^{-5}$).
  * Cột `unseen: ['bus_bypass']` (do bản sửa loại sạch va chạm kiểu này trên TRAIN).
  * Quãng đường `progress = 213.69 m` (giữ nguyên so với gốc $213.71\text{ m}$, mất $< 0.01\%$), số khung phanh gấp `harsh = 8.10` frames (giảm so với gốc $8.55$).
- **Trên tập TEST:**
  * **Policy được chọn:** `fail = 7.13e-06`, `fail_hi_seen = 1.94e-05`, `fail_hi (valid) = 3.88e-02` ($0.0388$), `unseen: ['bus_bypass']`. Quãng đường `progress = 213.58 m`, phanh gấp `harsh = 7.97`.
  * **Policy gốc (baseline):** `fail = 4.72e-04` ($4.722 \times 10^{-4}$), `fail_hi_seen = 7.85e-04`, `fail_hi (valid) = 2.06e-03` ($0.00206$), `unseen: []`. Quãng đường `progress = 213.75 m`, phanh gấp `harsh = 8.40`.

**Vì sao `fail_hi` của bản sửa ($0.0388$) lại lớn hơn `fail` của policy gốc ($0.00047$) dù bản sửa tốt hơn thật?**
Bản sửa `cutinLat = 3.0` đã loại bỏ hoàn toàn các va chạm `bus_bypass` trong 400 lần chạy của TEST (hits = 0), làm cho kiểu này trở thành `unseen`. Theo nguyên lý thống kê bảo toàn (95% upper bound), với mẫu $n = 400$ từ phân phối hỗn hợp phòng thủ $\beta = 0.2$, không có va chạm chỉ cho phép chứng nhận cận trên `upper_zero_hits = 3 / (0.2 * n) = 3 / 80 = 0.0375` ($3.75\%$). Cận trên này được cộng trực tiếp vào `fail_hi` của bản sửa ($7.13 \times 10^{-6} + 0.0375 + 1.28 \times 10^{-3} \approx 0.0388$). 
Với ngân sách $n = 400$, ta không thể chứng nhận một rủi ro đã giảm xuống dưới $10^{-2}$ (muốn chứng minh $p < 10^{-3}$ thì cần tới $\approx 15\,000$ lần chạy). Nếu chỉ so sánh trên các kiểu thực tế quan sát được, `fail_hi_seen` của bản sửa là $1.94 \times 10^{-5}$, tốt hơn policy gốc ($7.85 \times 10^{-4}$) tới **40 lần**.

**Núm chỉnh "nghe an toàn hơn" nhưng thực tế làm tăng thất bại:**
Núm `cautionV` (tốc độ thận trọng khi đi qua đoạn bị che khuất bởi xe buýt):
- Mặc định là `cautionV = 7.5 m/s` (fail trên TRAIN là $1.91 \times 10^{-4}$).
- Khi giảm tốc độ xuống `cautionV = 6.0 m/s` (nghĩ rằng đi chậm sẽ an toàn hơn): `fail` trên TRAIN tăng vọt lên **$6.59 \times 10^{-4}$** (tăng gấp gần 3.5 lần!) và `fail_hi` tăng lên $1.42 \times 10^{-3}$.
- **Giải thích bằng phản ứng của thế giới:** Khi xe ego giảm tốc độ quá sớm và bò quá chậm ($6.0\text{ m/s}$), thời gian xe ego phơi nhiễm (exposure time) bên cạnh sườn xe buýt đỗ kéo dài hơn đáng kể. Việc bò chậm khiến xe hàng xóm C2 (vừa vượt qua xe buýt) và người đi bộ P1 có thời gian lao ra cắt đầu đúng lúc xe ego vừa trườn tới. Ngoài ra, việc phanh giật cục làm xe ego dừng chắn đúng vạch qua đường, biến thành lỗi `crosswalk_block`. Thế giới giao thông phản ứng động với tốc độ của ego, nên đi quá chậm tại điểm nghẽn làm tăng nguy cơ va chạm.

*(Ảnh chụp 3D so sánh bản sửa và bóng mờ của policy gốc được lưu tại `evidence/shots/q6_fix_comparison.png`).*

## 7. Từ mô phỏng ra đường thật

Ba giả định trong `space.py` ảnh hưởng nhiều nhất đến kết quả:
1. **`pedLook` (Tỉ lệ người đi bộ dừng ngó qua xe buýt - hiện tại là $0.85$ ở thị trấn B):**
   - *Dữ liệu thật thay thế:* Trích xuất từ camera giám sát giao thông tại các điểm dừng xe buýt đô thị (hoặc các bộ dữ liệu quan sát hành vi người đi bộ như JAAD / PIE), dùng Computer Vision để đo xác suất người qua đường có ngoái nhìn dòng xe trước khi bước ra.
   - *Hướng sai lệch:* Ngoài đời nhiều người đi bộ mất tập trung (dùng điện thoại, vội chạy theo xe buýt) khiến tỷ lệ ngó thực tế thấp hơn ($< 50\%$). Khi đó va chạm `crosswalk_block` và đâm người đi bộ ngoài đời sẽ cao hơn nhiều so với mô phỏng.
2. **`cutGap` (Khe hở an toàn mà xe cắt làn chấp nhận - Log-Normal trung vị $12\text{ m}$):**
   - *Dữ liệu thật thay thế:* Dữ liệu quỹ đạo xe trích xuất từ drone quay góc nhìn trên cao tại các tuyến phố đô thị (như bộ dữ liệu HighD, CitySim), đo phân phối khoảng cách bumper-to-bumper thực tế khi tài xế tạt đầu chuyển làn.
   - *Hướng sai lệch:* Nếu tài xế đô thị liều lĩnh hơn (chấp nhận khe $5-8\text{ m}$), số vụ tạt đầu nguy hiểm sẽ tăng vọt; ngược lại nếu tài xế điềm tĩnh hơn thì con số mô phỏng đang bị bi quan quá mức.
3. **`wet` (Xác suất và độ trơn trượt của đường ướt - $\mu = 0.5$ với xác suất $0.30$ ở thị trấn B):**
   - *Dữ liệu thật thay thế:* Dữ liệu khí tượng thủy văn theo giờ (tần suất mặt đường ướt do mưa) kết hợp với dữ liệu đo độ nhám và hệ số ma sát lốp - mặt đường thực tế từ cơ quan quản lý giao thông đô thị bằng thiết bị GripTester.
   - *Hướng sai lệch:* Nếu đường đô thị có độ thoát nước và độ bám tốt hơn ($\mu \approx 0.65$ khi ướt), quãng đường phanh khẩn cấp sẽ ngắn hơn, làm giảm đáng kể các vụ va chạm do trượt lốp so với mô phỏng.

## 8. Giới hạn

Ba điều mà xác suất báo cáo **không** phản ánh:
1. **Hành vi tài xế xung quanh chỉ là mô hình toán IDM 1D:** Không phản ánh được tài xế say xỉn, bấm còi, giật mình đánh lái tránh né linh hoạt, hoặc các tình huống tranh chấp không gian phức tạp.
2. **Cảm biến và thị giác hoàn hảo không độ trễ:** Xe ego biết ngay lập tức vị trí, vận tốc của mọi đối tượng khi không bị che khuất. Thực tế có độ trễ nhận diện (perception latency 100–300 ms), nhiễu tracking (bounding box jitter) và nhận diện sót (false negatives).
3. **Hình học không gian chỉ là một đoạn phố thẳng đơn lẻ:** Không có giao lộ ngã tư, không có vòng xuyến, không có độ dốc mặt đường, không có sự thay đổi điều kiện ánh sáng (chói nắng ban ngày, tối ban đêm).

*Nếu được thêm một tuần:* Tôi sẽ mô hình hóa thêm **nhiễu và độ trễ nhận diện (Perception Latency & Detection Noise)** lên hệ thống cảm biến của xe ego, đồng thời mở rộng mô phỏng cho khu vực ngã tư giao cắt có điểm mù rẽ phải để đo rủi ro va chạm ở nút giao phức tạp.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Người đi bộ bất ngờ băng ngang đường tự phát ngoài vạch kẻ (`type: ped`, xuất phát từ lề đường phía Nam $z = 9.2\text{ m}$ bước sang làn phía Bắc). Ngoài đời, đây là tình huống người đi bộ sang đường tùy tiện (jaywalking) giữa phố đô thị từ sau hàng cây hoặc vỉa hè ra lòng đường.
- **Căn cứ:** Planner lái theo IDM dọc làn $E1$ và chỉ có các núm thận trọng nhắm vào vạch người đi bộ cố định (`cautionRange`, `cautionV`). Khi người đi bộ cắt ngang ở vị trí ngẫu nhiên ngoài vạch, planner không có vùng giảm tốc phòng ngừa từ xa mà chỉ phản ứng khi người đi bộ lọt vào hành lang an toàn `pedBuffer` hẹp ($2.5\text{ m}$), dễ dẫn đến phanh gấp nguy hiểm.
- **Giả định (`hazard.json: why`):**
  - `space`: Vị trí $x \in [210.0, 250.0]\text{ m}$ (`h0_x`), vận tốc đi bộ $v \in [1.3, 1.7]\text{ m/s}$ (`h0_v`), cự ly kích hoạt $trig \in [45.0, 65.0]\text{ m}$ (`h0_trig`). Dựa trên vận tốc đi bộ trung bình đô thị ($1.3 - 1.5\text{ m/s}$) và tầm nhìn nhận biết chướng ngại vật của tài xế.
  - `exposure`: $0.04$ (trung bình $0.04$ lần mỗi chuyến đi đô thị 5km, tương ứng 1 lần trong 25 chuyến đi, dựa trên thống kê tai nạn và vi phạm bộ hành đô thị).
- **Kết quả (theo `evidence/mine.json`):**
  - Tỷ lệ kịch bản khả thi vật lý: `accepted = 99.4%` ($1491 / 1500$ kịch bản hợp lệ, `infeasible = 9`).
  - **Policy gốc (baseline):**
    * Tổng va chạm: `crashes = 1` (`avoidable = 0`, `unavoidable = 0`, `unattributed = 1` - va chạm thông thường của thị trấn với C2 lúc $t = 12.55\text{ s}$ trước khi rắc rối tác động, `low_speed = 0`).
    * $P(\text{lỗi planner | rắc rối}) = 0.00$, khoảng Wilson 95%: $[0.00, 2.57 \times 10^{-3}]$.
    * Tần suất mỗi chuyến: $0.00$ vụ/chuyến (cận trên $1.03 \times 10^{-4}$ vụ/chuyến).
  - **Policy đã chọn (`chosen` với `cutinLat = 3.0`):**
    * Tổng va chạm: `crashes = 0` (`avoidable = 0`, `unavoidable = 0`, `unattributed = 0`, `low_speed = 0`).
    * $P(\text{lỗi planner | rắc rối}) = 0.00$, khoảng Wilson 95%: $[0.00, 2.57 \times 10^{-3}]$.
    * Tần suất mỗi chuyến: $0.00$ vụ/chuyến (cận trên $1.03 \times 10^{-4}$ vụ/chuyến).
  - **So sánh với lách xe buýt (`bus_bypass`):** Tần suất va chạm `bus_bypass` của policy gốc là $\approx 6.56 \times 10^{-4}$ vụ/chuyến, lớn hơn gấp nhiều lần so với cận trên rủi ro của rắc rối jaywalking này ($1.03 \times 10^{-4}$).
- **Giải thích vì sao planner xử lý tốt:** Trong mô phỏng `avsim`, người đi bộ dừng lại khi có thân xe chắn trước mặt (`PED_LOOK = 0.5 m`). Khi người đi bộ kích hoạt từ cự ly $45 - 65\text{ m}$ ($3.6 - 5.2\text{ s}$), planner IDM có đủ thời gian phanh nhường mượt mà hoặc lướt qua an toàn trước khi người đi bộ chạm làn. Policy sửa (`cutinLat = 3.0`) còn loại bỏ luôn vụ va chạm ngoại lai `unattributed` với C2, đạt mức an toàn tuyệt đối.
- **Hình:** Ảnh chụp kịch bản trong trình xem 3D lưu tại `evidence/shots/hazard_studio.jpg`.
- **Phân biệt `avoidable`, `unavoidable`, `unattributed`:**
  * `avoidable`: Va chạm do rắc rối gây ra mà hệ thống phanh tối đa $\mu g$ ngay khung hình đầu tiên phát hiện có thể tránh được hoàn toàn (lỗi của planner).
  * `unavoidable`: Va chạm mà dù đã phanh tối đa $\mu g$ ngay lập tức từ khi nhìn thấy vẫn không thể tránh do vượt quá giới hạn vật lý.
  * `unattributed`: Va chạm xảy ra trước khi rắc rối bắt đầu hoặc va chạm với xe khác không liên quan đến rắc rối (va chạm nền ngẫu nhiên của thị trấn).
- **Vì sao "tránh được" (`avoidable`) là chuẩn lạc quan:**
  * *Theo hướng lạc quan (đếm nhiều hơn lỗi thật):* Oracle phanh ngay lập tức không có độ trễ phản xạ ($0\text{ ms}$) và phanh hết cỡ $\mu g$. Planner thực tế cần thời gian xử lý nhận diện ($200 - 500\text{ ms}$) và phanh êm ái để bảo đảm an toàn cho hành khách, nên chuẩn này gán cho planner trách nhiệm phải phản ứng hoàn hảo như máy.
  * *Theo hướng bi quan (đếm ít đi):* Chuẩn này chỉ thử nghiệm phương án phanh thẳng, hoàn toàn **không thử nghiệm đánh lái né tránh (swerving)**. Trong thực tế, nhiều tình huống không đủ quãng đường phanh nhưng tài xế hoặc planner thông minh có thể lách lái sang làn trống để thoát nạn.
- **Bạn tự lái thì sao:** Khi tự lái trong chương 10, tôi chủ động rà phanh giảm tốc từ sớm ngay khi nhìn thấy người đi bộ ngập ngừng ở mép đường, trong khi planner chỉ phanh khi người đi bộ đã thực sự bước vào vùng `pedBuffer` trên làn xe.
- **Cách sửa cho planner:**
  * *Núm chỉnh hiện có:* Tăng `pedBuffer` từ $2.5\text{ m}$ lên $3.5\text{ m}$ để phát hiện và phanh nhường người đi bộ sớm hơn khi họ vừa tiến sát mép vỉa hè.
  * *Logic mới cần bổ sung:* Bổ sung module **dự báo cắt ngang quỹ đạo (Time-to-Collision Cross-Prediction)**: tính vector vận tốc người đi bộ $\vec{v}_{\text{ped}}$ để dự đoán điểm giao cắt tương lai $(x_{\text{cross}}, t_{\text{cross}})$. Nếu $TTC < 3.0\text{ s}$, planner chủ động giảm tốc độ hành trình mục tiêu $v_0$ từ xa thay vì chỉ dựa vào IDM khi vật thể đã chắn trước đầu xe.
