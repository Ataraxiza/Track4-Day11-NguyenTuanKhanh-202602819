MSSV: 20262819

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): 

## 1. Phát lại log có trung thực không? (bước A)

Trong `replay.json`: bao nhiêu kịch bản bị bỏ vì người lái gây va chạm, bao nhiêu va chạm khi planner chạy trên log phát lại, bao nhiêu trong thế giới phản ứng, theo từng loại tiếp xúc. Giải thích vì sao phần lớn va chạm của "phát lại" là `rear`, và vì sao đó là va chạm **giả**. Chọn một kịch bản trong `replay_only` và nói chuyện gì đã xảy ra.

### Trả lời
- Số liệu:
n = 400, `dropped_human_crash_logs` = 3, `n_used` = 397
`replay.crashes` = 107
`replay.contact` = {"rear": 97, "side": 10}
`reactive.crashes` = 0

- Kết luận: Không “trung thực” theo nghĩa là không phải một mô phỏng thế giới phản ứng (reactive world simulation). Phần lớn va chạm trong replay là rear vì khi phát lại log, planner đang chạy trên quỹ đạo đã được ghi sẵn; nó không biết người lái phía trước sẽ phanh sớm hơn, nên planner “đuổi” theo từ phía sau và va chạm từ sau. Trong khi ở mô phỏng reactive world, cùng kịch bản đó không có va chạm nào (`reactive.crashes` = 0).

- Ví dụ kịch bản trong replay_only: scenario 1 (`contact` = "rear", `min_ttc_reactive` = 2.281132...)
Có nghĩa là: trong reactive world, cùng cảnh đó vẫn còn khoảng cách an toàn ~2.28 s trước khi có nguy hiểm; nhưng khi chạy theo log đã ghi, planner không được “nhìn thấy” sự phanh sớm trước đó, nên nó chạy đến chỗ bị đuổi từ sau. Nói ngắn gọn: “log đã khóa động tác của đối tượng phía trước; planner chỉ theo, không có thời gian phanh trước.”


## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

### Trả lời
- Các luật trong is_feasible:
Vật lý ban đầu không thể nếu cần lực ngang vượt ma sát: `a_lat` = (pi / cutDur)^2 * 3.5 / 2 > mu * g
Không thể nếu phanh sau khi cắt làn `cutBrake` > mu * g
Không thể nếu người đi bộ chạy quá nhanh `pedV` > 2.5 m/s
Không thể nếu tốc độ cắt làn không nằm trong (0, 22] m/s

- Căn cứ vật lý:
mu * g là lực bám tối đa cho phanh/lực ngang.
cutDur và quỹ đạo cắt làn 3.5 m là bài toán dao động bánh xe; nếu yêu cầu gia tốc ngang lớn hơn lực bám, đó là kịch bản không thể xảy ra.

- Cuộc săn nào sinh ra nhiều kịch bản bất khả thi nhất?
Theo `hunt.json`, `bus_bypass` có `infeasible` = 12, cao nhất.
`cut_in` = 6, `rear_ended` = 4, `crosswalk_block` = 3

- Vì sao CEM bị kéo về đó?
CEM chọn “nguy hiểm nhất” theo điểm nguy hiểm của kiểu đó. Mỗi mẫu được đánh cao khi TTC gần 0 hoặc khoảng phanh còn lại rất nhỏ. Vùng `bus_bypass` cho ra nhiều điểm cực kỳ nguy hiểm, nên q bị tụ vào đó.
Nhưng nhiều kịch bản trong vùng đó là không thực tế mà chỉ là các bộ tham số vượt giới hạn vật lý: ví dụ cắt làn quá gấp, phanh sau khi đổi làn quá mạnh, hoặc chắn đường không hợp lý.

- Nếu tính cả những kịch bản bất khả thi, chẳng hạn như va chạm, kết quả ước lượng sẽ bị phóng đại. Khi đó, các tình huống không thể xảy ra vẫn bị xem là lỗi của planner, dẫn đến xác suất ước lượng sai lệch và khiến CEM tập trung quá mức vào những vùng phi thực tế, thay vì các khu vực thực sự tiềm ẩn rủi ro.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k, n, p̂ và khoảng Wilson 95 %. Với p̂ đó, cần bao nhiêu lần chạy để khoảng tin cậy hẹp lại còn ±20 % của p? Có kiểu thất bại nào **chưa hề xuất hiện** trong 3 000 lần chạy không? Con số đúng cần báo cho kiểu đó là gì?

### Trả lời
- Theo mc.json:
k = 7
n = 3000
p̂ = 7 / 3000 = 0.0023333333333333335
Wilson 95% khoảng: [0.0011307176584015297, 0.004808876493300746]

- Số lần chạy cần thiết để khoảng tin cậy còn ±20% của p:
Với p ≈ 0.00233, cần khoảng n ≈ 96 / p ≈ 4.1 × 10^4. Tức là khoảng 41,000 lượt chạy.

- Có kiểu thất bại nào chưa xuất hiện trong 3000 lần không?
Có. `cut_in`, `crosswalk_block`, `rear_ended`, `other` đều có k = 0.

- Con số đúng cần báo cho kiểu chưa thấy:
Không báo 0. Phải báo cận trên 95% Wilson: với k = 0, n = 3000, hi ≈ 0.0012789.
Trong `mc.json`, ví dụ `other.hi` = 0.0012788956648046956. Đây là “cận trên khi chưa thấy va chạm”, không phải “xác suất bằng 0”.


## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

### Trả lời:
- Dữ liệu chính từ `estimate.json`:
Seed 0:
bus_bypass: p = 0.0018171, hits = 33, ess = 149.44, top_share = 0.39595
crosswalk_block: p = 4.85e-7, hits = 70, top_share = 0.2515
Seed 1:
bus_bypass: p = 0.00032866, hits = 21, ess = 120.80, top_share = 0.2684
crosswalk_block: p = 6.18e-7, hits = 72, top_share = 0.2053
Seed 2:
bus_bypass: p = 0.00045690, hits = 33, ess = 123.42, top_share = 0.2115
crosswalk_block: p = 7.84e-7, hits = 74, top_share = 0.6835

- Ba seed có đồng ý với nhau không?
Không hoàn toàn. bus_bypass chênh từ 3.29e-4 đến 1.82e-3, tức gần 5,5 lần.
`crosswalk_block` thì ổn định hơn, quanh 5–8e-7, nhưng cả hai mode đều thấp hơn MC ở kiểu chính (`bus_bypass` trong MC là 7/3000 = 0.002333).
Về tổng thể: total của 3 seed lần lượt là 0.0018176, 0.0003293, 0.0004577. Có độ lệch rất lớn, cho thấy IS trên ngân sách này không “đẹp” như MC.

- Có đồng ý với Monte Carlo không?
Đúng kiểu chính `bus_bypass`: điểm số IS nằm cùng bậc với MC, nhưng chưa sát; seed 0 gần MC, seed 1–2 thấp hơn khá nhiều.
Với `crosswalk_block` và các mode khác, MC thấy 0 nên không thể so khớp trực tiếp; IS cho ra số rất nhỏ nhưng không bằng 0.

- Nơi `top_share` lớn (> 0.5)?
Chỉ có seed 2 ở `crosswalk_block`: `top_share` = 0.6835430462100572. Khi `top_share` lớn, nghĩa là ước lượng bị chi phối bởi một mẫu nặng nhất. Một số rất lớn w khiến ước lượng gần như “trọng số của một mẫu” thay vì nhiều mẫu cùng đóng góp. Đây là dấu hiệu của q quá hẹp hoặc q không bao phủ đủ vùng nguy hiểm.

- So ngân sách với số Monte Carlo cần:
Ngân sách IS trong `estimate.json` là `budget` = 16800 tổng. Nhưng vì `top_share` cao và ESS chỉ ~100–150, nó vẫn là ước lượng chưa ổn định; đó là lý do estimator thay đổi mạnh giữa các seed.
Một MC “nguyên chất” cho cùng độ chính xác của p≈0.0023 cần khoảng 4.1e4 lượt, nên ngân sách 16800 không đủ nếu muốn ổn định hoàn toàn.

- Phần other trong tổng lấy từ đâu?
`estimate.json` ghi `other.source` = "mc.json", và `other.p` = 0.0, `other.hi` = 0.0012789
Nói cách khác: phần other không được tìm bằng CEM, mà được “nhúng” từ MC vì tự nhiên không thấy kiểu đó trong các cuộc săn. Đó là lý do không săn tiếp: other là “phần dư” không thuộc các kiểu có cấu trúc rõ ràng và không hấp dẫn cho việc giới hạn dải tìm kiếm.


## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

### Trả lời
- Các kiểu thất bại:
`B-01..B-04`: `bus_bypass`
`C-01..C-04`: `crosswalk_block`
`R-01..R-03`: `replay` / replay-only crash, không phải dạng thất bại thật do planner tạo ra trong thế giới phản ứng
-> Chụp 2 ảnh (1 ảnh cho bus_bypass, 1 ảnh cho crosswalk_block): 
1 ảnh cho `bus_bypass`: xe buýt che tầm nhìn, xe khác lách qua phía sau, planner không kịp tránh và va chạm. Đây là kiểu chiếm nhiều nguyên mẫu nhất và cũng là vùng mà CEM bị kéo về nhất. Xem hình `evidence/shots/Câu 5_B-01.png`
1 ảnh cho `crosswalk_block`: người đi bộ đang bước qua vạch qua đường, xe ta chặn đường hoặc không đủ khoảng phanh, tạo va chạm chặn lối đi. Xem hình `evidence/shots/Câu 5_C-01.png`

- Theo viewer.json và hunt.json, các kiểu mà cuộc săn thực sự tìm thấy rõ ràng là `bus_bypass` và `crosswalk_block` (`hunt.json: found.bus_bypass = 79`, `found.crosswalk_block = 35`). `cut_in` không xuất hiện trong các cuộc săn thật (`mc.json: k = 0` cho kiểu đó, `cut_in.hi ≈ 0.0012789`), và `rear_ended` cũng không là mode được hunt “nắm” như một thành phần độc lập; trong `hunt.json` nó chỉ xuất hiện như một số trường hợp lẫn lộn với `bus_bypass` / `crosswalk_block` (`found = {"bus_bypass": 4, "crosswalk_block": 5}`), không phải thành một mode riêng biệt. Vì vậy, với câu 5, bạn chỉ cần chọn một hình đại diện cho từng mode thật sự tìm thấy, không phải upload 11 ảnh riêng.

- Nếu chỉ chạy một cuộc săn chung cho mọi kiểu, kết quả sẽ bị kéo về vùng nguy hiểm chiếm ưu thế, tức `bus_bypass`: `hunt.json` cho thấy khi tìm từng mode, `bus_bypass` và `crosswalk_block` được tìm thấy rõ ràng, nhưng các mode khác dễ bị “ngập” trong vùng này. Đó là lý do chiến lược tìm riêng từng mode lại quan trọng hơn nhiều so với một cuộc săn tổng hợp.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

### Trả lời

- **Policy được chọn:** `fix.json: chosen.policy = {"cutinLat": 3.0}`.
- **TRAIN:** policy gốc có `fail = 0.0011553`, `fail_hi = 0.0024130`; policy được chọn có `fail = 1.31e-7`, `fail_hi = 2.24e-7` (`fix.json: baseline_train`, `chosen`). Quãng đường trung bình gần như giữ nguyên, từ 211.80 m xuống 211.43 m; số khung hình phanh mạnh trung bình (`harsh`, gia tốc < −3 m/s²) giảm từ 9.10 xuống 8.24.
- **TEST:** policy được chọn có `fail = 5.88e-7`, `fail_hi_seen = 1.18e-6`, nhưng cận hợp lệ `fail_hi = 0.03878`; policy gốc có `fail = 2.69e-4`, `fail_hi_seen = 4.84e-4`, `fail_hi = 0.001763` (`fix.json: chosen_test`, `baseline_test`). Quãng đường trung bình lần lượt là 216.51 m và 216.54 m; số khung hình phanh mạnh trung bình lần lượt là 7.54 và 8.17.
- `fail_hi` của policy sửa cao hơn `fail` của policy gốc vì đây không phải phép so sánh cùng loại số: `fail` là ước lượng điểm, còn `fail_hi` là cận trên 95% bảo thủ. TEST không quan sát thấy `bus_bypass` với policy sửa (`chosen_test.unseen = ["bus_bypass"]`), nên cận hợp lệ phải cộng `upper_zero_hits` cho mode chưa thấy; điều này đẩy `fail_hi` lên 0.03878 dù các va chạm quan sát được ít hơn. Không được diễn giải con số này là bằng chứng bản sửa tệ hơn; nó cho thấy TEST chưa đủ dữ liệu để chứng nhận mode đã biến mất.
- Trong lần chạy này, không có bằng chứng rằng một núm nghe an toàn hơn đã làm tăng `fail` quan sát được. Ví dụ, hạ `cautionV` xuống 5.0 để chạy chậm hơn khi bị che khuất làm `fail` TRAIN giảm, nhưng quãng đường cũng giảm rõ (211.80 m xuống 169.07 m) và số khung hình phanh mạnh tăng (9.10 lên 13.43) (`fix.json: table`). Trong thế giới phản ứng, phanh sớm/mạnh có thể khiến xe bám đuôi áp sát rồi tông từ sau; mẫu progress lần này không có crash `other`, nên đây là nguy cơ cần kiểm tra chứ không phải kết quả đã chứng minh. Policy được chọn tăng `cutinLat` từ 2.15 lên 3.0, mở rộng vùng mà planner xem xe cắt làn là vật cản; điều này giúp phát hiện xe sớm hơn, nhưng `bus_bypass` không xuất hiện trong TEST nên độ bất định còn lớn.
- Hình: `evidence/shots/Câu 6_Bóng mờ.png`

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

### Trả lời

- **Khe xe chuyển làn chấp nhận (`cutGap`)**: mô hình dùng phân phối log-normal với trung vị 12 m (`avsim/space.py: CONT`, `Natural.cut_gap_median`). Tôi sẽ thay bằng khoảng cách trước/sau thực tế tại thời điểm bắt đầu và hoàn tất chuyển làn, trích từ video/trajectory đã gán nhãn như INTERACTION hoặc Waymo Open Motion Dataset; cần lọc các lần chuyển làn thực sự có chủ đích để ước lượng khe mà tài xế chấp nhận. Nếu khe thực tế thường nhỏ hơn giả định, các lần cắt làn sát xe ta phổ biến hơn và xác suất lỗi hiện tại có thể bị đánh giá thấp; nếu khe thực tế lớn hơn thì mô phỏng có thể đánh giá rủi ro quá cao.
- **Người đi bộ nhìn/nhô ra khỏi sau xe buýt (`pedLook`)**: thị trấn A giả định xác suất này là 0,85 (`avsim/space.py: BIN`). Tôi sẽ lấy video tại điểm dừng xe buýt có người đi bộ, gán nhãn hành vi nhìn/nhô người và thời điểm họ trở nên nhìn thấy được, hoặc dùng nghiên cứu quan sát hành vi người đi bộ tại điểm bị che khuất. Nếu tỷ lệ thực thấp hơn 0,85, mô hình hiện tại cho người đi bộ lộ ra sớm quá thường xuyên, nên có thể đánh giá xác suất va chạm thấp hơn thực tế; nếu tỷ lệ cao hơn, rủi ro có thể bị đánh giá cao.
- **Tỷ lệ đường ướt và độ bám (`wet`, `mu`)**: mô hình giả định 20% đường ướt; dùng `mu = 0.5` khi ướt và `0.8` khi khô (`avsim/space.py: BIN`, `to_params`). Tôi sẽ kết hợp dữ liệu thời tiết và cảm biến trạng thái mặt đường từ các chuyến xe với phép đo độ bám/phanh của đúng loại lốp và tốc độ. Nếu đường ướt thường xuyên hơn hoặc độ bám thấp hơn các giả định, quãng đường phanh dài hơn và nguy cơ tăng, nên ước lượng hiện tại có thể thấp; nếu đường khô/độ bám cao hơn thì có thể cao quá.

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

### Trả lời

- Xác suất này chỉ áp dụng cho bản đồ đường/giao lộ tổng hợp của thị trấn A và các phân phối đã giả định trong `avsim/space.py`; nó không đại diện cho mọi loại đường, mật độ giao thông hay thời tiết ngoài đời.
- Xe khác chủ yếu theo mô hình IDM và người đi bộ theo luật băng qua đơn giản; chúng không bao quát đầy đủ hành vi bất ngờ, tương tác xã hội, hoặc khác biệt giữa từng người lái.
- Mô phỏng không đánh giá đầy đủ lỗi cảm biến, độ trễ điều khiển và sai số định vị ngoài đời. Vì vậy, kết quả không phải là bằng chứng rằng xe thật an toàn với cùng xác suất.

Nếu có thêm một tuần, tôi sẽ thu thập và gán nhãn video/trajectory ở các điểm dừng xe buýt để đo phân phối khe chuyển làn, tần suất người đi bộ bị che và độ trễ phát hiện; đồng thời đo ma sát mặt đường theo điều kiện thời tiết. Sau đó tôi sẽ cập nhật các phân phối này và chạy lại kiểm tra trên các cảnh/địa điểm độc lập để xem xác suất và khoảng tin cậy thay đổi thế nào.

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

### Trả lời

- **Rắc rối và căn cứ:** Thêm một xe đạp đang chạy ở làn bên cạnh rồi bất ngờ lạng vào làn xe ta, như khi tránh chướng ngại vật gần khu vực giao cắt. Planner có thể tiếp tục đi theo làn/tốc độ hiện tại cho đến khi xe đạp bắt đầu lạng, khiến khoảng cách phanh không còn đủ. Cảnh chính xác trong `hazard.json` có va chạm giữa `ego` và `H0` ở 7.85 s (`mine.json: exact.planner`).
- **Giả định:** `hazard.json` thay đổi vị trí dọc `h0_x = [0, 15]`, độ lệch `h0_d = [1, 1.5]`, tốc độ `h0_v = [4, 6]`, độ lạng `h0_swerve = [3.2, 3.8]`, thời gian lạng `h0_swDur = [3.2, 4]`, thời điểm bắt đầu `h0_t0 = [0, 1]` và ngưỡng kích hoạt `h0_trig = [18, 24]`. Các giá trị nằm trong `studio.LIMITS`; gia tốc ngang lớn nhất theo biên khoảng 1.83 m/s², dưới giới hạn 3 m/s². `mine.json: accepted = 0.99` (1485/1500 mẫu khả thi; 15 bị loại). `exposure = 0.01` nghĩa là giả định ban đầu một tình huống như vậy mỗi 100 chuyến, chưa phải số đo thực địa; `hazard.json: why` nêu rõ cần thay giả định này bằng đếm xung đột xe đạp từ video/trajectory trên tuyến đường tương ứng.
- **Kết quả (`mine.json`, n = 1485):**

  | Policy | Va chạm | Avoidable | Unavoidable | Unattributed | Low-speed |
  |---|---:|---:|---:|---:|---:|
  | Gốc | 455 | 454 | 1 | 0 | 0 |
  | Được chọn (`cutinLat = 3.0`) | 87 | 86 | 0 | 1 | 22 |

  Với policy gốc, `P(lỗi planner | rắc rối) = 0.306` (Wilson 95%: [0.283, 0.330]); với policy được chọn là `0.0579` (Wilson 95%: [0.0471, 0.0710]). `low_speed` là cờ đếm riêng, có thể nằm trong các nhóm va chạm ở trên.
- **Tần suất mỗi chuyến:** với giả định exposure 0.01, policy gốc cho `0.00306` lỗi/chuyến (cận trên 0.00330); policy được chọn cho `0.000579` lỗi/chuyến (cận trên 0.000710). Để so sánh, `mc.json` quan sát `bus_bypass` 7/3000 = `0.00233` mỗi lượt mô phỏng. Theo exposure giả định này, lỗi xe đạp của policy gốc cùng bậc và cao hơn khoảng 1.3 lần; policy được chọn thấp hơn, nhưng phép so sánh phụ thuộc trực tiếp vào exposure 1% chưa được đo.
- **Diễn giải “avoidable”:** đây là chuẩn oracle lạc quan: giả sử rắc rối được nhận biết ngay khi vừa nhìn thấy và xe phanh hết cỡ tức thì. Vì tài xế/planner thực không có biết trước hoàn hảo, số `avoidable` có thể cao hơn số lỗi thật sự tránh được. Mặt khác, phép thử chỉ cho phép phanh, không thử đánh lái; một số tình huống có thể tránh được bằng đánh lái nhưng bị xếp không tránh được.
- **Tự lái:** tôi chưa tự lái thử cảnh này nên không khẳng định mình chắc chắn tránh được. Người lái có thể nhận ra xe đạp đổi hướng sớm hơn qua tư thế/chuyển động ngang và chủ động giảm tốc trước khi nó vào hẳn làn.
- **Cách sửa:** policy được chọn giảm va chạm trong mẫu này, nhưng `cutinLat` không phải logic dự đoán xe đạp. Cần theo dõi vận tốc ngang và dự đoán quỹ đạo xe đạp; khi thấy chuyển động hướng vào làn, giảm tốc sớm và giữ khoảng trống phanh, đồng thời kiểm tra xe phía sau trước khi phanh mạnh.
- **Hình:** `Câu 9_Hazard.png`
