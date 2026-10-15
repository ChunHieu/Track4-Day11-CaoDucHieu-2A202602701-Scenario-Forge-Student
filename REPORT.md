MSSV: 2A202602701

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): C

## 1. Phát lại log có trung thực không? (bước A)

Trích số từ `replay.json`:
- Số kịch bản bị loại do người lái thật va chạm: `dropped_human_crash_logs = 5` (trên tổng số `n = 400`, giữ lại `n_used = 395`).
- Planner chạy trên **log phát lại** (`replay`): có **114** va chạm, gồm: `rear = 101`, `side = 13`, `front = 0`.
- Planner chạy trong **thế giới phản ứng** (`reactive`): có **1** va chạm, gồm: `side = 1`, `rear = 0`, `front = 0`.
- Số kịch bản va chạm ở cả hai bên: `crash_in_both = 1`.

**Giải thích:**
Phần lớn va chạm của log phát lại là `rear` (101/114 vụ, chiếm 88.6%). Đây là **va chạm giả (open-loop replay artefact)**. Trong phương pháp phát lại vết lái thật, xe bám đuôi `TG` chỉ tái hiện máy móc quỹ đạo và gia tốc trong quá khứ của người lái cũ. Khi planner của xe ta phát hiện mối nguy (người đi bộ bị xe buýt che khuất) và phanh sớm/thận trọng hơn người lái thật, xe `TG` trong log không hề "biết" xe ta đã phanh sớm và cứ tiếp tục phóng tới theo tốc độ cũ dẫn tới tông đuôi xe ta (`rear`). Trong thế giới phản ứng tương tác (`reactive`), xe `TG` được điều khiển bởi mô hình IDM nhận thức được khoảng cách và tốc độ của xe ta nên đạp phanh giảm tốc kịp thời, hoàn toàn không gây tai nạn.

**Kịch bản trong `replay_only`:** Kịch bản `scenario: 0` (`replay.json: replay_only[0]`):
- Tiếp xúc: `contact = "rear"`. Ở thế giới phản ứng, khoảng thời gian va chạm nhỏ nhất là `min_ttc_reactive = 2.564s` (xe sau duy trì cự ly an toàn).
- Chuyện xảy ra: Mặt đường trơn ướt (`mu = 0.5`), xe ta giảm tốc cẩn trọng trước giao lộ và vạch người đi bộ. Xe bám đuôi `TG` (`tgT = 0.66s`) trong log phát lại không phanh theo hành vi mới của xe ta nên húc thẳng vào đuôi xe ta; trong khi ở chế độ phản ứng, `TG` phanh theo xe ta và giữ khoảng cách an toàn.

## 2. Kịch bản bất khả thi (bước B)

**Các luật trong `is_feasible` và căn cứ vật lý:**
1. $a_{\text{lat}} = \left(\frac{\pi}{\text{cutDur}}\right)^2 \frac{3.5}{2} \le \mu g$: Giới hạn ma sát bám ngang khi chuyển làn theo quỹ đạo cosin 3.5m. Theo định luật Newton và ma sát trượt Coulomb $F_{\text{lat}} \le \mu m g$, gia tốc ngang tối đa không thể vượt quá $\mu g$. Nếu $a_{\text{lat}} > \mu g$, xe bị trượt lết ngang mất lái và không thể thực hiện thao tác chuyển làn.
2. $\text{cutBrake} \le \mu g$: Giới hạn gia tốc phanh dọc của xe. Lực hãm tối đa của lốp trên mặt đường bị giới hạn bởi $\mu m g$; phanh lớn hơn $\mu g$ sẽ làm khóa bánh trượt trượt không thể xảy ra ngoài đời thực.
3. $\text{pedV} \le 2.5\text{ m/s}$: Giới hạn sinh học của người đi bộ. Vận tốc đi bộ bình thường là $1.2 - 1.5\text{ m/s}$; $2.5\text{ m/s}$ ($9\text{ km/h}$) là tốc độ chạy nước rút tối đa của người băng qua đường. Vượt quá ngưỡng này là phi thực tế với người đi bộ đô thị.
4. $0 < \text{cutV} \le 22.0\text{ m/s}$: Giới hạn tốc độ lưu thông đô thị ($22\text{ m/s} \approx 79.2\text{ km/h}$). Vận tốc âm hoặc bằng 0 thì xe không thể vượt và cắt làn.

**Cuộc săn sinh nhiều kịch bản bất khả thi nhất:**
Theo `hunt.json`, cuộc săn `crosswalk_block` loại bỏ nhiều kịch bản bất khả thi nhất với `infeasible = 8` (so với `cut_in: 2`, `bus_bypass: 1`, `rear_ended: 1`).
CEM bị kéo về phía đó vì hàm mục tiêu của CEM tối ưu hóa theo điểm nguy hiểm (danger score). Những kịch bản vi phạm vật lý (như người đi bộ chạy nhanh phi lý hoặc xe phanh gấp vượt giới hạn ma sát) tạo ra các tình huống bất khả kháng khiến planner không thể xử lý kịp, đạt điểm nguy hiểm cao nhất. Do đó, CEM tự nhiên hội tụ mạnh vào vùng kịch bản cực đoan đó.

**Hậu quả nếu ĐẾM chúng là va chạm:** Xác suất va chạm ước lượng sẽ bị **thổi phồng giả tạo (overestimation)**, báo cáo sai lệch những tai nạn ảo không thể xảy ra trong thế giới thực, làm sai lệch đánh giá an toàn và lãng phí tài nguyên phát triển để xử lý những ca phi vật lý.

## 3. Monte Carlo cho bạn biết gì (bước C1)

Trích số từ `mc.json`:
- Số va chạm: `crashes = 4` ($k = 4$)
- Tổng số lần chạy: `n = 3000`
- Ước lượng điểm: $\hat{p} = \frac{4}{3000} \approx 1.33 \times 10^{-3}$ (`p = 0.001333`)
- Khoảng tin cậy Wilson 95%: $[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$ (`lo = 0.000519`, `hi = 0.003424`)

**Số lần chạy cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**
Bán kính sai số của khoảng tin cậy là $z \sqrt{\frac{p(1-p)}{n}} \approx z \sqrt{\frac{p}{n}}$.
Để sai số tương đối $\le 20\% \cdot p$:
$$z \sqrt{\frac{p}{n}} \le 0.20 \cdot p \iff n \ge \frac{z^2}{0.20^2 \cdot p} \approx \frac{1.96^2}{0.04 \cdot p} = \frac{96.04}{p}$$
Với $\hat{p} \approx 0.001333$:
$$n \approx \frac{96.04}{0.001333} \approx 72,030 \text{ lần chạy}$$

**Kiểu thất bại chưa hề xuất hiện:**
Trong 3.000 lần chạy, chỉ duy nhất kiểu `bus_bypass` xuất hiện ($k = 4$). Các kiểu `cut_in`, `crosswalk_block`, `rear_ended`, và `other` **chưa hề xuất hiện** ($k = 0$).
Con số đúng cần báo cho các kiểu đó **tuyệt đối không được bằng 0** (chưa gặp không có nghĩa là không xảy ra). Phải báo bằng **cận trên 95% Wilson**:
$$p \le \frac{z^2}{n + z^2} = \frac{1.96^2}{3000 + 1.96^2} \approx 1.28 \times 10^{-3} \quad (\text{trích } \texttt{mc.json: per\_mode.cut\_in.hi = 0.001279})$$
(Hoặc báo theo Rule of Three: $\frac{3}{n} = \frac{3}{3000} = 1.0 \times 10^{-3}$).

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng số liệu trích từ `estimate.json` (ngân sách `budget = 16800`):

| Kiểu thất bại | Seed | $p$ | Hits ($k$) | ESS | `top_share` | Cận trên 95% |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **bus_bypass** | 0 | $2.01 \times 10^{-3}$ | 39 | 132.7 | 0.290 | $3.39 \times 10^{-3}$ |
| | 1 | $3.50 \times 10^{-3}$ | 34 | 111.1 | **0.577** | $7.56 \times 10^{-3}$ |
| | 2 | $5.81 \times 10^{-4}$ | 23 | 129.7 | 0.268 | $9.87 \times 10^{-4}$ |
| **crosswalk_block** | 0 | $6.60 \times 10^{-8}$ | 58 | 119.4 | 0.244 | $1.12 \times 10^{-7}$ |
| | 1 | $6.04 \times 10^{-8}$ | 43 | 110.2 | **0.683** | $1.43 \times 10^{-7}$ |
| | 2 | $3.42 \times 10^{-8}$ | 47 | 121.2 | 0.287 | $6.03 \times 10^{-8}$ |
| **cut_in** | 0, 1, 2 | 0.0 | 0 | ~135 | 0.0 | 0.025 (`upper_zero_hits`) |
| **rear_ended** | 0, 1, 2 | 0.0 | 0 | ~125 | 0.0 | 0.025 (`upper_zero_hits`) |
| **Tổng (Total)** | 0 | $2.01 \times 10^{-3}$ | — | — | — | $[6.29 \times 10^{-4}, 4.67 \times 10^{-3}]$ |
| | 1 | $3.50 \times 10^{-3}$ | — | — | — | $[0.0, 8.84 \times 10^{-3}]$ |
| | 2 | $5.81 \times 10^{-4}$ | — | — | — | $[1.76 \times 10^{-4}, 2.27 \times 10^{-3}]$ |

- **Độ đồng thuận giữa các seed và với Monte Carlo:** Cả 3 seed đều có cùng bậc độ lớn: `bus_bypass` cỡ $10^{-3}$, `crosswalk_block` cỡ $10^{-8}$. Khoảng tin cậy của 3 seed giao nhau và bao phủ khoảng Wilson của Monte Carlo ($[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$). Trung vị của `total` qua 3 seed là $2.01 \times 10^{-3}$ (Seed 0), rất phù hợp với Monte Carlo ($1.33 \times 10^{-3}$).
- **Vị trí `top_share > 0.5` và ý nghĩa:** Ở Seed 1, `bus_bypass` có `top_share = 0.577` và `crosswalk_block` có `top_share = 0.683`. Khi `top_share > 0.5`, hơn một nửa giá trị của ước lượng $\hat{p} = \frac{1}{n} \sum w_i y_i$ chỉ do một mẫu duy nhất mang lại. Điều này xảy ra do phân phối đề xuất $q$ co hẹp lại hoặc bốc phải một mẫu ở vùng đuôi có trọng số $w = p/q$ rất lớn, chi phối toàn bộ con số trung bình.
- **So sánh ngân sách (`budget`):** Ngân sách IS là 16.800 lần chạy. Để Monte Carlo đo được kiểu hiếm như `crosswalk_block` ($p \approx 6 \times 10^{-8}$), MC cần $n \approx \frac{96}{p} \approx 1.6 \times 10^9$ lần chạy (1.6 tỷ lần chạy!). IS tiết kiệm chi phí tính toán hơn 5 bậc độ lớn.
- **Phần `other`:** Lấy từ `mc.json: other` ($k = 0, n = 3000, \text{hi} = 0.00128$). Không săn `other` vì đây là tập hợp mọi va chạm ngẫu nhiên, không có cấu trúc hình học hay hàm điểm nguy hiểm cụ thể để thuật toán CEM tối ưu hóa.

## 5. Các kiểu thất bại (bước C2 + 3D)

**1. `bus_bypass`:**
- Ảnh 3D: [evidence/shots/bus_bypass.jpg](file:///e:/Track4/Day11/Track4-Day11-Scenario-Forge-Student/evidence/shots/bus_bypass.jpg)
- Chuỗi sự kiện: Xe buýt dừng đỗ trước vạch qua đường che khuất tầm nhìn. Xe C2 ở làn ngoài lách qua xe buýt với vận tốc chậm ($v < 6\text{ m/s}$) để nhập vào làn E1 tại $x < 112\text{ m}$. Xe ta bị xe buýt che khuất tầm nhìn nên không nhận diện C2 là xe dẫn đầu cho đến khi C2 lấn sang tim làn, dẫn đến va chạm góc trước-bên.

**2. `crosswalk_block`:**
- Ảnh 3D: [evidence/shots/crosswalk_block.jpg](file:///e:/Track4/Day11/Track4-Day11-Scenario-Forge-Student/evidence/shots/crosswalk_block.jpg)
- Chuỗi sự kiện: Người đi bộ P1 bước ra từ sau đuôi xe buýt đỗ. Do xe buýt che khuất tầm nhìn, xe ta phát hiện P1 rất muộn và phanh khẩn cấp hết cỡ. Xe ta dừng lại hoàn toàn ($v = 0$), nhưng thân xe đè lên vạch qua đường (`WALK_X`), chặn ngay trước mặt P1 khiến người đi bộ phải đứng lại (sự kiện `block`).

**Kiểu cuộc săn không tìm thấy và mức độ tin cậy:**
Cuộc săn `cut_in` và `rear_ended` không tìm thấy va chạm đúng kiểu của mình (trong `hunt.json`, cuộc săn `cut_in` tìm được 37 vụ `bus_bypass`, cuộc săn `rear_ended` tìm được 56 vụ `bus_bypass`). Mức độ tin cậy được bảo chứng ở cận trên $3 / (0.2 \cdot n) = 0.025$ (hoặc cận Wilson MC $1.28 \times 10^{-3}$): trong điều kiện bình thường, planner IDM giữ khoảng cách đủ an toàn với các pha chuyển làn thông thường và duy trì gia tốc phù hợp nên hiếm khi bị đâm đuôi trong môi trường phản ứng.

**Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu:**
Thuật toán CEM sẽ bị **suy biến và co cụm (mode collapse)** về kiểu va chạm dễ xảy ra nhất là `bus_bypass`. Các kiểu thất bại hiếm khác như `crosswalk_block` sẽ hoàn toàn bị bỏ sót (trong `hunt.json`, cả 2 cuộc săn `cut_in` và `rear_ended` đều tự động bị kéo về tìm thấy toàn bộ là `bus_bypass`). Vì vậy bắt buộc phải tổ chức các cuộc săn độc lập cho từng kiểu thất bại.

## 6. Sửa planner (bước E + 3D)

Trích số từ `fix.json`:
- **Policy được chọn:** `{'cutinLat': 3.0}` (tăng ngưỡng nhận diện xe lấn làn từ 2.15m lên 3.0m).
- **Trên TRAIN:** `fail = 7.58e-08`, `fail_hi = 1.63e-07`, `progress = 211.4 m`, `unseen = ['bus_bypass']`.
- **Trên TEST:**
  - Bản sửa (`chosen`): `fail = 7.75e-08`, `fail_hi_seen = 1.81e-07`, `fail_hi (valid) = 0.0388` ($3.88 \times 10^{-2}$), `unseen = ['bus_bypass']`, `progress = 211.6 m`, `harsh = 8.30`.
  - Policy gốc (`baseline`): `fail = 4.52e-04`, `fail_hi_seen = 7.57e-04`, `fail_hi (valid) = 0.00204` ($2.04 \times 10^{-3}$), `unseen = []`, `progress = 211.9 m`, `harsh = 8.74`.

**Vì sao `fail_hi` của bản sửa lại lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật?**
Trên TEST, bản sửa loại bỏ toàn bộ va chạm `bus_bypass`, khiến kiểu này trở thành `unseen` ($k = 0$). Về mặt toán học, khi không có va chạm trên $n = 400$ mẫu IS, cận trên 95% hợp lệ của kiểu đó không thể bằng 0 mà phải cộng thêm cận `upper_zero_hits` $= 3 / (0.2 \cdot n) = 3 / 80 = 0.0375$. Do đó `fail_hi = 1.81e-07 + 0.0375 = 0.0388`. Con số này lớn hơn `fail` của policy gốc không phải vì bản sửa kém hơn, mà vì với ngân sách $n = 400$ ta chưa đủ mẫu để chứng nhận một kiểu đã biến mất dưới $10^{-2}$ (muốn chứng nhận $p < 10^{-3}$ cần cỡ 15.000 lần chạy).

**Quãng đường và phanh gấp:**
- Quãng đường: bản sửa đạt $211.6\text{ m}$ (so với baseline $211.9\text{ m}$, chỉ giảm $0.14\%$, thấp hơn nhiều ngưỡng trần $5\%$).
- Phanh gấp: giảm từ $8.74$ xuống $8.30$ frames (xe vận hành êm ái hơn).

**Ảnh chụp 3D có bóng mờ:** [evidence/shots/bus_bypass.jpg](file:///e:/Track4/Day11/Track4-Day11-Scenario-Forge-Student/evidence/shots/bus_bypass.jpg) (bóng mờ xe ta theo policy gốc tiến sát xe buýt và va chạm với C2, trong khi bản sửa `cutinLat = 3.0` phát hiện C2 từ khoảng cách ngang 3.0m nên chủ động phanh sớm và tránh được va chạm).

**Núm chỉnh "nghe an toàn hơn" nhưng làm tăng thất bại:**
Núm `cautionV`: Giảm `cautionV = 5.0 m/s` nghe có vẻ an toàn khi qua đoạn khuất tầm nhìn. Tuy nhiên, xe ta chạy quá chậm ($5\text{ m/s}$) làm quãng đường giảm mạnh xuống $172.5\text{ m}$ (mất $> 18\%$ quãng đường), đồng thời khiến các xe phía sau và xe cắt làn bị dồn ứ, cố lách qua xe ta, làm tăng đột biến nguy cơ va chạm từ bên hông và phía sau.

## 7. Từ mô phỏng ra đường thật

Ba giả định ảnh hưởng nhiều nhất trong `space.py`:
1. **Tỉ lệ người đi bộ ngó qua xe buýt (`pedLook = 0.85`):**
   - Dữ liệu thật thay thế: Dữ liệu theo dõi hướng nhìn và hành vi người đi bộ qua camera góc nhìn người thứ ba (bộ dữ liệu PIE - Pedestrian Intent Estimation hoặc JAAD).
   - Hướng sai lệch: Nếu ngoài đời người đi bộ ít ngó nghiêng hơn (ví dụ chỉ $50\%$, nhiều người cúi đầu nhìn điện thoại), tần suất người đi bộ bước thẳng ra làn đường sẽ cao hơn, khiến con số va chạm `crosswalk_block` trong mô phỏng bị thấp hơn thực tế.
2. **Khe thời gian xe chuyển làn chấp nhận (`cutGap ~ LogNormal(12.0, 0.5)`):**
   - Dữ liệu thật thay thế: Dữ liệu quỹ đạo xe tự nhiên (Naturalistic Driving Study) từ Waymo Open Dataset hoặc HighD ghi nhận khoảng cách chuyển làn thực tế theo mật độ dòng xe.
   - Hướng sai lệch: Nếu tài xế ngoài đời bạo dạn hơn, chấp nhận các khe hẹp hơn ($< 10\text{ m}$), các tình huống cắt đầu xe sẽ diễn ra gấp gáp hơn, khiến rủi ro va chạm `bus_bypass` và `cut_in` ngoài đời cao hơn trong mô phỏng.
3. **Hệ số ma sát đường ướt (`mu = 0.5` với tỉ lệ $20\%$):**
   - Dữ liệu thật thay thế: Số liệu trạm khí tượng thủy văn kết hợp thiết bị đo hệ số ma sát mặt đường (Skid Resistance Data) trên các loại vật liệu asphalt thực tế.
   - Hướng sai lệch: Nếu mặt đường khi mưa trơn trượt hơn ($\mu \approx 0.35 - 0.4$), quãng đường phanh thực tế $d = \frac{v^2}{2 \mu g}$ sẽ dài hơn đáng kể, làm tăng số vụ xe trượt đè vạch người đi bộ (`crosswalk_block`).

## 8. Giới hạn

Ba điều mà xác suất báo **không** phản ánh:
1. **Hình học đường phố quá đơn giản:** Thế giới chỉ có một con phố thẳng tắp, không có nút giao ngã tư chữ thập, bùng binh, góc cua khuất tầm nhìn, hay độ dốc mặt đường.
2. **Hành vi tài xế chỉ là mô hình IDM lý tưởng:** Các xe khác lái xe hoàn hảo theo IDM, không mô phỏng các hành vi phức tạp của con người như mất tập trung, dùng điện thoại, lạng lách bất quy tắc, hay bóp còi cảnh báo.
3. **Cảm biến giả định hoàn hảo:** Hệ thống nhận diện bounding box tức thì không có độ trễ khung hình (latency), không có nhiễu cảm biến (sensor noise), không bị điểm mù lidar và không bị ảnh hưởng bởi chói sáng hay mưa bão.

**Nếu được thêm một tuần:** Sẽ mở rộng mô hình sang **ngã tư có đèn tín hiệu (signalized intersection)** và các kịch bản rẽ trái không được ưu tiên (unprotected left turns), đồng thời đưa thêm **độ trễ nhận thức (perception latency ~ 100-200ms)** vào planner để kiểm thử độ bền vững.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Xe đạp đi ở làn ngoài `E2` bất ngờ lạng gấp sang trái $1.6 - 2.2\text{ m}$ vào thẳng làn `E1` của xe ta khi xe ta còn cách $11 - 20\text{ m}$. Ngoài đời thực, đây là tình huống người đi xe đạp lạng ra tim đường để tránh nắp cống sụt, ổ gà, hoặc cửa ô tô đỗ bên đường bất ngờ mở ra.
- **Căn cứ:** Planner của xe ta có hành lang nhận diện ngang mặc định `cutinLat = 2.15 m`. Khi xe đạp ở làn `E2` ($z = 5.25\text{ m}$, cách ngang $3.5\text{ m}$), xe ta không coi xe đạp là vật cản phía trước. Khi xe đạp lạng gấp vào làn ở cự ly gần ($11 - 20\text{ m}$), IDM chỉ phanh êm $b = 2.0\text{ m/s}^2$, hoàn toàn không đủ quãng đường để dừng lại từ tốc độ $12\text{ m/s}$, dẫn đến va chạm trực diện.
- **Giả định:**
  - `space`: `h0_x: [-25.0, 5.0]`, `h0_swerve: [1.6, 2.2]`, `h0_trig: [11.0, 20.0]`, `h0_swDur: [2.0, 2.6]`. Gia tốc ngang cực đại của xe đạp đạt đỉnh $a_{\text{lat}} \le 2.7\text{ m/s}^2 < 3.0\text{ m/s}^2$ (hoàn toàn khả thi vật lý).
  - `exposure`: $0.02$ (xảy ra khoảng 2 lần trong 100 chuyến đi đô thị, theo khảo sát xung đột giao thông xe đạp của FHWA/NHTSA).
- **Kết quả (trích từ `mine.json`):**
  - Tỉ lệ kịch bản khả thi: `accepted = 99.2%` (1488/1500 kịch bản).
  - Với **policy gốc (baseline)**:
    - Tổng va chạm: `crashes = 667`
    - `avoidable = 666` (tránh được bằng phanh hết cỡ ngay khi thấy)
    - `unavoidable = 1`
    - `unattributed = 0`
    - `low_speed = 0`
    - P(lỗi planner | rắc rối) = $0.4476$ ($44.76\%$), khoảng Wilson 95%: $[0.4225, 0.4729]$ (`p = 0.44758`, `lo = 0.42248`, `hi = 0.47295`).
    - Tần suất mỗi chuyến: $0.4476 \times 0.02 = 8.95 \times 10^{-3}$ lần/chuyến (`per_trip = 0.00895`, cận trên `per_trip_hi = 0.00946`). Đặt cạnh tần suất lách xe buýt ($2.01 \times 10^{-3}$), rắc rối này nguy hiểm gấp hơn **4 lần** kịch bản lách xe buýt!
  - Với **policy bản sửa (`chosen`: `cutinLat = 3.0`)**:
    - Tổng va chạm: `crashes = 0`
    - `avoidable = 0`, `unavoidable = 0`, `unattributed = 0`, `low_speed = 0`.
    - P(lỗi planner | rắc rối) = $0.0$, khoảng Wilson 95%: $[0.0, 0.00258]$.
    - Tần suất mỗi chuyến: $0.0$ (cận trên $5.15 \times 10^{-5}$).
- **Hình:** [evidence/shots/hazard_studio.jpg](file:///e:/Track4/Day11/Track4-Day11-Scenario-Forge-Student/evidence/shots/hazard_studio.jpg) (ảnh giao diện Xưởng kịch bản chương 10 thiết lập thẻ rắc rối H0 cho xe đạp lạng làn).
- **Bạn tự lái thì sao:** Khi tự lái (chế độ "Bạn"), người lái có khả năng nhận thức ngữ cảnh từ xa: khi thấy xe đạp đi sát mép đường phía trước, người lái chủ động nhả ga rà phanh chuẩn bị trước (defensive driving) thay vì giữ nguyên ga $12.5\text{ m/s}$ như planner, nên khi xe đạp vừa bắt đầu lạng ra là xe dừng êm ái cách xa 5-8 mét.
- **Phân biệt avoidable, unavoidable, unattributed và chuẩn lạc quan:**
  - `avoidable`: Va chạm quy cho rắc rối mà nếu phanh khẩn cấp hết cỡ ở giới hạn bám của lốp ($\mu g$) ngay từ khung hình đầu tiên nhìn thấy thì sẽ không va chạm (đây là tiêu chuẩn tính lỗi planner).
  - `unavoidable`: Va chạm mà ngay cả khi đạp lút phanh ở giới hạn vật lý ngay từ lúc nhìn thấy vẫn đâm (tình huống bất khả kháng, không tính là lỗi planner).
  - `unattributed`: Va chạm thường của thị trấn xảy ra trước khi rắc rối bắt đầu kích hoạt (không liên quan đến rắc rối).
  - **Vì sao "tránh được" là chuẩn lạc quan?**
    - Theo một hướng, nó **lạc quan quá mức (đếm nhiều hơn lỗi thực)**: Oracle giả định hệ thống phát hiện rắc rối tức thì ở khung hình đầu tiên và lập tức đạp phanh tối đa $\mu g$ không trễ. Một planner thật trong thực tế không thể lúc nào cũng phanh cắm đầu $\mu g$ mỗi khi thấy xe đạp ở làn bên vì sẽ gây giật cục và bị xe sau tông đuôi.
    - Theo hướng ngược lại, nó **bi quan (đếm ít hơn)** vì phương thức tránh duy nhất được thử nghiệm là phanh dừng thẳng hàng, hoàn toàn không thử đánh lái tránh (swerve/evasive maneuver) - vốn là cách người lái thường dùng để tránh xe đạp mà không cần dừng xe.
- **Cách sửa:**
  - Bộ tham số tối ưu ở bước E (`cutinLat = 3.0`) đã giải quyết hoàn hảo rắc rối này (giảm từ 666 va chạm xuống 0 va chạm), nhờ mở rộng hành lang nhận diện ngang từ 2.15m lên 3.0m giúp phát hiện xe đạp lấn làn sớm hơn nhiều và chủ động phanh nhường đường.
  - Về mặt thuật toán dài hạn: Planner cần bổ sung **mô-đun nhận thức dự đoán ý định (cyclist intent prediction)** và cơ chế giảm tốc thận trọng khi phát hiện người đi xe đạp ở làn lân cận thay vì chỉ phản ứng thụ động khi vật thể đã lấn vào làn xe chạy.
