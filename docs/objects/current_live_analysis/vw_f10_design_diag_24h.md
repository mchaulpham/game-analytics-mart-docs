# vw_f10_design_diag_24h

## 1. Mục đích

`vw_f10_design_diag_24h` là view chẩn đoán thiết kế 10 level đầu trong vòng 24 giờ kể từ lần đầu người chơi bắt đầu Level 1.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này gom nhiều nhóm chỉ số để hỗ trợ đánh giá từng level trong 10 level đầu, gồm:

* tỷ lệ người chơi tới level;
* tỷ lệ rơi sang level tiếp theo;
* tỷ lệ thắng;
* tỷ lệ thua;
* số lần thử trung bình;
* tỷ lệ `Start_level` không ghép được `End_level`;
* tỷ lệ dữ liệu bị ảnh hưởng bởi tutorial;
* chỉ số đã loại tutorial;
* thời gian đi tới level;
* thời gian ở trong level;
* lý do thua phổ biến;
* nhãn chẩn đoán tự động.

Mục tiêu chính là giúp team xác định level nào cần ưu tiên kiểm tra về thiết kế hoặc tracking.

---

## 2. Vai trò trong hệ thống

`vw_f10_design_diag_24h` thuộc nhóm:

```text id="pph0nv"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là view chẩn đoán sâu cho 10 level đầu.

Luồng phụ thuộc chính:

```text id="psaixa"
dim_f10_level_map
vw_level_start_eff
vw_level_attempts
vw_f10_funnel_timing_24h
  └── vw_f10_design_diag_24h
```

View này được dùng tiếp bởi:

```text id="s1y2yw"
vw_f10_design_board_24h
```

`vw_f10_design_board_24h` là bảng ưu tiên hành động, còn `vw_f10_design_diag_24h` là bảng chẩn đoán chi tiết.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Level nào trong 10 level đầu có tỷ lệ rơi cao?
2. Level nào có tỷ lệ rơi sau khi thắng cao?
3. Level nào có tỷ lệ thắng sạch thấp?
4. Level nào có số lần thử trung bình cao?
5. Level nào có nhiều lượt bắt đầu nhưng không có kết thúc?
6. Level nào có tỷ lệ dữ liệu bị ảnh hưởng bởi tutorial cao?
7. Level nào có tỷ lệ `End_level` thấp, có thể là rủi ro tracking?
8. Level nào có thời lượng chơi quá dài?
9. Người chơi thắng level rồi có đi tiếp level sau không?
10. Người chơi bắt đầu level rồi có đi tiếp level sau không?
11. Lý do thua phổ biến nhất là gì?
12. Level nào nên được ưu tiên kiểm tra trong thiết kế?
13. Level nào có thể quá dễ?
14. Level nào có thể quá khó?
15. Level nào có rủi ro do tutorial làm nhiễu dữ liệu?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_f10_design_diag_24h` đại diện cho một bước trong 10 level đầu của một phiên bản game.

Có thể hiểu đơn giản:

```text id="gn0w01"
Một dòng = một app_version + một progression_step + một level_name
```

View này là bảng tổng hợp cấp level, không phải dữ liệu từng người chơi và không phải dữ liệu từng lượt chơi.

---

## 5. Loại đối tượng

```text id="ytr5tr"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

View có dùng `CURRENT_TIMESTAMP()` nên thời điểm chạy view được ghi lại ở cột:

```text id="2swndd"
diagnostics_view_query_time_utc
```

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào bốn nguồn chính:

| Nguồn                      | Vai trò                                                                            |
| -------------------------- | ---------------------------------------------------------------------------------- |
| `dim_f10_level_map`        | Xác định 10 level đầu theo từng phiên bản game.                                    |
| `vw_level_start_eff`       | Xác định nhóm người chơi bắt đầu Level 1 và cửa sổ 24 giờ.                         |
| `vw_level_attempts`        | Cung cấp dữ liệu lượt chơi level, thắng/thua, số lần thử, duration, tutorial flag. |
| `vw_f10_funnel_timing_24h` | Cung cấp chỉ số phễu và thời gian tổng hợp.                                        |

---

## 7. Cách xác định phạm vi phân tích

View chỉ phân tích 10 level đầu được định nghĩa trong:

```text id="vyvwrj"
dim_f10_level_map
```

Điều kiện mapping:

```sql id="f700n7"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step BETWEEN 1 AND 10
```

Nhóm người chơi đầu vào là người chơi đã bắt đầu Level 1 và đã đủ 24 giờ quan sát.

Cửa sổ phân tích:

```text id="n127zs"
level1_start_time_utc
đến
level1_start_time_utc + 24 giờ
```

---

## 8. Logic xử lý chính

### 8.1 Tạo danh sách assignment của 10 level đầu

View lấy thông tin mapping gồm:

```text id="iax60o"
app_version
progression_step
progression_slot_id
level_name
level_design_id
level_type
previous_progression_step
next_progression_step
mapping_confidence_status
mapping_source
```

Đây là khung level để mọi chỉ số được gắn đúng vào từng bước trong 10 level đầu.

---

### 8.2 Tạo nhóm người chơi đã đủ 24 giờ

View lấy người chơi có lần đầu bắt đầu Level 1 từ `vw_level_start_eff`.

Chỉ giữ người chơi đã đủ 24 giờ kể từ Level 1.

Điều kiện:

```text id="xwj02s"
level1_start_time_utc <= thời điểm chạy view - 24 giờ
```

---

### 8.3 Lấy lượt chơi level trong 24 giờ đầu

View lấy dữ liệu lượt chơi từ:

```text id="hye8s6"
vw_level_attempts
```

Điều kiện chính:

```text id="yew2ja"
cùng user_pseudo_id
cùng app_version
cùng level_name
start_time_utc nằm trong 24 giờ sau Level 1
```

Dữ liệu này được dùng để tính:

* số lượt thử;
* lượt có kết thúc;
* lượt không có kết thúc;
* thắng;
* thua;
* số bước đi;
* thời lượng;
* cờ tutorial.

---

### 8.4 Tính lần đầu tới mỗi level

View xác định lần đầu người chơi tới từng step bằng cách lấy `Start_level` sớm nhất cho mỗi:

```text id="ml3fl7"
user_pseudo_id
app_version
progression_step
```

Phần này giúp tính số người tới level và so sánh với bước tiếp theo.

---

### 8.5 Tính lần thắng đầu tiên của mỗi level

View xác định lần thắng đầu tiên của người chơi ở từng level bằng:

```text id="qoi1oq"
success IS TRUE
AND end_time_utc IS NOT NULL
```

Sau đó dùng mốc thắng đầu tiên để kiểm tra người chơi có đi tiếp bước sau không.

---

### 8.6 Tính rơi từ level hiện tại sang level tiếp theo

View có hai nhóm rơi chính:

#### Rơi từ bắt đầu level sang bước tiếp theo

```text id="o6mqur"
start_to_next_step_drop_user_count
start_to_next_step_rate
start_to_next_step_drop_rate
```

Nhóm này trả lời:

```text id="x4fgmd"
Trong số người đã tới level hiện tại, bao nhiêu người đi tiếp level sau?
```

#### Rơi sau khi thắng level

```text id="m506hx"
win_to_next_step_drop_user_count
win_to_next_step_rate
win_to_next_step_drop_rate
```

Nhóm này trả lời:

```text id="m8tcb9"
Trong số người đã thắng level hiện tại, bao nhiêu người đi tiếp level sau?
```

Đây là chỉ số quan trọng để phân biệt:

* người chơi rơi vì level khó trước khi thắng;
* người chơi thắng rồi vẫn không đi tiếp, có thể do vấn đề chuyển tiếp, UI, thưởng, pacing, hoặc fatigue.

Trong tài liệu này, “pacing” nghĩa là nhịp độ trải nghiệm của game.

---

### 8.7 Tính chỉ số lượt thử

View tính các chỉ số theo lượt chơi:

```text id="dr6zru"
attempt_count
matched_end_count
start_without_end_count
win_attempt_count
fail_attempt_count
matched_end_rate
start_without_end_rate
win_rate_among_matched_ends
fail_rate_among_matched_ends
```

Nhóm này cho biết level có bao nhiêu lượt chơi, bao nhiêu lượt kết thúc, bao nhiêu lượt thắng/thua.

---

### 8.8 Tách dữ liệu sạch và dữ liệu bị tutorial ảnh hưởng

View tính cả chỉ số tổng và chỉ số sạch.

Dữ liệu sạch được định nghĩa là:

```sql id="5da8we"
is_near_tutorial_start IS NOT TRUE
```

Các cột sạch có tiền tố:

```text id="1v7ds3"
clean_
```

Ví dụ:

```text id="e98msl"
clean_attempt_count
clean_matched_end_count
clean_win_attempt_count
clean_win_rate_among_matched_ends
```

Việc tách dữ liệu sạch rất quan trọng vì tutorial có thể làm hành vi người chơi không còn tự nhiên.

---

### 8.9 Tính phân phối số lần thử

View tính số lượt thử mỗi người chơi ở từng level, sau đó tổng hợp thành:

```text id="i2e3jr"
avg_attempt_count_per_reached_user
median_attempt_count_per_reached_user
p75_attempt_count_per_reached_user
p90_attempt_count_per_reached_user
```

Các chỉ số này giúp phát hiện level có áp lực retry cao.

Trong tài liệu này, “retry” nghĩa là người chơi phải chơi lại level nhiều lần.

---

### 8.10 Tóm tắt lý do thua

View tạo hai cột tóm tắt lý do thua:

```text id="gttpg7"
top_fail_reason_summary
clean_top_fail_reason_summary
```

Trong đó:

* `top_fail_reason_summary` tính trên toàn bộ lượt thua;
* `clean_top_fail_reason_summary` chỉ tính trên lượt thua không gần tutorial.

Mỗi cột lấy tối đa 5 lý do thua phổ biến nhất.

---

### 8.11 Gắn chỉ số thời gian

View lấy các chỉ số thời gian từ:

```text id="8kz4wi"
vw_f10_funnel_timing_24h
```

Bao gồm:

* thời gian từ Level 1 đến step;
* thời gian từ step trước đến step hiện tại;
* thời gian ở trong level theo timestamp;
* thời lượng trong level theo `duration_sec`.

---

### 8.12 Gán nhãn chẩn đoán

View tạo ba nhóm nhãn:

```text id="k7i34q"
sample_size_status
design_sample_size_status
progression_diagnostic_bucket
level_design_performance_bucket
primary_metric_signal
```

Các nhãn này giúp team đọc nhanh tình trạng level mà không phải tự diễn giải từng cột.

---

## 9. Nhóm cột đầu ra

View có 97 cột, được chia thành các nhóm sau.

### 9.1 Nhóm mapping

```text id="nfxdvy"
app_version
progression_step
progression_slot_id
level_name
level_design_id
level_type
previous_progression_step
next_progression_step
mapping_confidence_status
mapping_source
```

Ý nghĩa: xác định level trong chuỗi 10 level đầu.

---

### 9.2 Nhóm cohort và reach

```text id="s9m7sx"
level1_cohort_user_count
matured_level1_cohort_user_count
not_matured_level1_cohort_user_count
reached_user_count
matured_reached_user_count
not_matured_reached_user_count
attempt_reached_user_count
```

Ý nghĩa: mô tả số người chơi trong nhóm Level 1 và số người tới từng level.

Lưu ý: trong logic hiện tại, `not_matured_reached_user_count` được đặt bằng 0 vì view chỉ phân tích nhóm đã đủ 24 giờ.

---

### 9.3 Nhóm chuyển tiếp sang bước tiếp theo

```text id="h4j8tf"
next_step_reached_user_count
start_to_next_step_drop_user_count
start_to_next_step_rate
start_to_next_step_drop_rate
won_user_count
matured_won_user_count
next_step_reached_after_win_user_count
win_to_next_step_drop_user_count
win_to_next_step_rate
win_to_next_step_drop_rate
```

Ý nghĩa: đo người chơi có đi tiếp level sau không, trước và sau khi xét điều kiện thắng.

---

### 9.4 Nhóm phễu tổng hợp

```text id="p2sdtz"
previous_step_user_count
drop_from_previous_step_user_count
conversion_rate_from_previous_step
drop_rate_from_previous_step
funnel_start_user_count
cumulative_drop_user_count
cumulative_conversion_rate
cumulative_drop_rate
```

Ý nghĩa: lấy từ phễu 10 level đầu, dùng để biết rơi từ bước trước và rơi cộng dồn.

---

### 9.5 Nhóm thắng theo người chơi

```text id="sx3g1p"
user_with_win_count
user_without_win_count
user_win_rate
user_without_win_rate
```

Ý nghĩa: trong số người đã tới level, có bao nhiêu người từng thắng level đó.

---

### 9.6 Nhóm lượt chơi tổng

```text id="dkf6x1"
attempt_count
matched_end_count
start_without_end_count
win_attempt_count
fail_attempt_count
matched_end_rate
start_without_end_rate
win_rate_among_matched_ends
fail_rate_among_matched_ends
```

Ý nghĩa: đo kết quả theo lượt chơi, không phải theo người chơi duy nhất.

---

### 9.7 Nhóm dữ liệu sạch, loại tutorial

```text id="ypf9jd"
tutorial_contaminated_attempt_count
tutorial_contaminated_attempt_rate
clean_attempt_count
clean_matched_end_count
clean_win_attempt_count
clean_fail_attempt_count
clean_matched_end_rate
clean_win_rate_among_matched_ends
clean_fail_rate_among_matched_ends
```

Ý nghĩa: tách lượt chơi gần tutorial khỏi dữ liệu phân tích thiết kế level.

---

### 9.8 Nhóm số lần thử

```text id="njx7xx"
avg_attempt_count_per_reached_user
median_attempt_count_per_reached_user
p75_attempt_count_per_reached_user
p90_attempt_count_per_reached_user
```

Ý nghĩa: đo áp lực retry của level.

---

### 9.9 Nhóm move và duration

```text id="rq0zrp"
avg_move_used_on_win
avg_move_used_on_fail
avg_move_left_on_win
avg_duration_sec_on_win
avg_duration_sec_on_fail
```

Ý nghĩa: đo số bước và thời lượng theo trạng thái thắng/thua.

---

### 9.10 Nhóm lý do thua

```text id="pvh2yo"
top_fail_reason_summary
clean_top_fail_reason_summary
```

Ý nghĩa: tóm tắt các lý do thua phổ biến nhất.

---

### 9.11 Nhóm thời gian đi tới level

```text id="p8pq5w"
level1_to_step_timing_sample_user_count
avg_seconds_from_level1_to_step
median_seconds_from_level1_to_step
p90_seconds_from_level1_to_step
previous_step_to_step_timing_sample_user_count
avg_seconds_from_previous_step_to_step
median_seconds_from_previous_step_to_step
p90_seconds_from_previous_step_to_step
```

Ý nghĩa: đo thời gian người chơi đi qua phễu 10 level đầu.

---

### 9.12 Nhóm thời gian trong level

```text id="e8peyl"
level_end_event_sample_user_count
level_end_event_coverage_rate
level_end_success_user_count
avg_wallclock_seconds_spent_inside_level
median_wallclock_seconds_spent_inside_level
p90_wallclock_seconds_spent_inside_level
reported_duration_sample_user_count
reported_duration_coverage_rate
avg_reported_duration_sec_inside_level
median_reported_duration_sec_inside_level
p90_reported_duration_sec_inside_level
```

Ý nghĩa: đo thời gian chơi trong level và độ phủ dữ liệu `End_level` / `duration_sec`.

---

### 9.13 Nhóm thời điểm và nhãn chẩn đoán

```text id="rm5d3f"
first_level1_start_time_utc
last_level1_start_time_utc
funnel_view_query_time_utc
timing_view_query_time_utc
sample_size_status
design_sample_size_status
progression_diagnostic_bucket
level_design_performance_bucket
primary_metric_signal
diagnostics_view_query_time_utc
```

Ý nghĩa: ghi nhận thời điểm dữ liệu và nhãn phân loại tự động.

---

## 10. Ý nghĩa các nhãn chẩn đoán

### 10.1 `sample_size_status`

Dựa trên `reached_user_count`:

| Điều kiện                  | Nhãn                |
| -------------------------- | ------------------- |
| `reached_user_count >= 50` | `sufficient_sample` |
| `reached_user_count >= 20` | `low_sample`        |
| còn lại                    | `very_low_sample`   |

Ý nghĩa: đánh giá kích thước mẫu tổng thể của level.

---

### 10.2 `design_sample_size_status`

Dựa trên `clean_matched_end_count`:

| Điều kiện                       | Nhãn                       |
| ------------------------------- | -------------------------- |
| `clean_matched_end_count >= 30` | `sufficient_design_sample` |
| `clean_matched_end_count >= 10` | `low_design_sample`        |
| còn lại                         | `very_low_design_sample`   |

Ý nghĩa: đánh giá mẫu đủ sạch để phân tích thiết kế level hay chưa.

---

### 10.3 `progression_diagnostic_bucket`

Nhãn này chẩn đoán rủi ro tiến trình từ level hiện tại sang level tiếp theo.

Các giá trị chính:

| Nhãn                                             | Ý nghĩa                                                          |
| ------------------------------------------------ | ---------------------------------------------------------------- |
| `terminal_step_no_next_step`                     | Level cuối trong nhóm phân tích, không có bước tiếp theo.        |
| `onboarding_or_tutorial_measurement_risk`        | Level đầu có tỷ lệ tutorial cao, có thể bị nhiễu bởi onboarding. |
| `tracking_risk_low_matched_end_rate`             | Tỷ lệ ghép `End_level` thấp, rủi ro tracking.                    |
| `hybrid_difficulty_and_post_win_transition_risk` | Vừa có rơi từ start sang next cao, vừa có rơi sau thắng cao.     |
| `post_win_transition_risk`                       | Người chơi thắng rồi nhưng không đi tiếp nhiều.                  |
| `difficulty_or_pre_win_churn_risk`               | Có áp lực số lần thử cao, có thể rơi trước khi thắng.            |
| `pre_win_churn_or_difficulty_risk`               | Rơi từ start sang next cao.                                      |
| `monitor`                                        | Chưa có tín hiệu rủi ro mạnh.                                    |

---

### 10.4 `level_design_performance_bucket`

Nhãn này tập trung vào hiệu năng thiết kế level sau khi loại tutorial.

Các giá trị chính:

| Nhãn                                       | Ý nghĩa                                             |
| ------------------------------------------ | --------------------------------------------------- |
| `tracking_risk_low_clean_matched_end_rate` | Tỷ lệ `End_level` sạch thấp, cần kiểm tra tracking. |
| `severe_difficulty_risk`                   | Tỷ lệ thắng sạch rất thấp, rủi ro level quá khó.    |
| `difficulty_risk`                          | Tỷ lệ thắng sạch thấp, cần kiểm tra độ khó.         |
| `friction_or_difficulty_risk`              | Số lần thử cao và tỷ lệ thắng không tốt.            |
| `possibly_too_easy`                        | Tỷ lệ thắng sạch rất cao, level có thể quá dễ.      |
| `monitor`                                  | Chưa có tín hiệu rủi ro mạnh.                       |

---

### 10.5 `primary_metric_signal`

Cột này đưa ra tín hiệu chính bằng ngôn ngữ dễ đọc hơn.

Ví dụ:

```text id="egfsd7"
Terminal step
High start-to-next drop and high post-win drop
High post-win transition drop
High attempt pressure
High start-to-next drop
Low clean win rate
Low matched End_level coverage
Monitor
```

Cột này phù hợp để hiển thị nhanh trong dashboard.

---

## 11. Kiểm tra chất lượng dữ liệu khuyến nghị

### 11.1 Kiểm tra số dòng theo app version

```sql id="6xtt0w"
SELECT
  app_version,
  COUNT(*) AS row_count,
  MIN(progression_step) AS min_progression_step,
  MAX(progression_step) AS max_progression_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

GROUP BY
  app_version

ORDER BY
  app_version;
```

Kỳ vọng thông thường:

```text id="ask7o3"
row_count = 10
min_progression_step = 1
max_progression_step = 10
```

---

### 11.2 Kiểm tra level có rủi ro tracking

```sql id="qs21nj"
SELECT
  app_version,
  progression_step,
  level_name,
  reached_user_count,
  attempt_count,
  matched_end_count,
  matched_end_rate,
  clean_matched_end_rate,
  progression_diagnostic_bucket,
  level_design_performance_bucket

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

WHERE
  matched_end_rate < 0.5
  OR clean_matched_end_rate < 0.5

ORDER BY
  matched_end_rate ASC,
  clean_matched_end_rate ASC;
```

Nếu nhiều level bị matched_end thấp, cần kiểm tra `End_level` tracking.

---

### 11.3 Kiểm tra level có rủi ro quá khó

```sql id="jq0y9h"
SELECT
  app_version,
  progression_step,
  level_name,

  clean_matched_end_count,
  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  p90_attempt_count_per_reached_user,

  level_design_performance_bucket,
  primary_metric_signal

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

WHERE
  level_design_performance_bucket IN (
    'severe_difficulty_risk',
    'difficulty_risk',
    'friction_or_difficulty_risk'
  )

ORDER BY
  clean_win_rate_among_matched_ends ASC,
  avg_attempt_count_per_reached_user DESC;
```

Truy vấn này giúp tìm level có dấu hiệu quá khó hoặc gây ma sát cao.

---

### 11.4 Kiểm tra rơi sau khi thắng

```sql id="etrw47"
SELECT
  app_version,
  progression_step,
  level_name,

  won_user_count,
  next_step_reached_after_win_user_count,
  win_to_next_step_drop_user_count,
  win_to_next_step_rate,
  win_to_next_step_drop_rate,

  progression_diagnostic_bucket,
  primary_metric_signal

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

WHERE
  win_to_next_step_drop_rate >= 0.25

ORDER BY
  win_to_next_step_drop_rate DESC;
```

Nếu người chơi thắng rồi vẫn không đi tiếp, vấn đề có thể nằm ở phần chuyển tiếp sau level, không chỉ ở độ khó level.

---

### 11.5 So sánh chỉ số tổng và chỉ số sạch

```sql id="m42afq"
SELECT
  app_version,
  progression_step,
  level_name,

  tutorial_contaminated_attempt_rate,

  win_rate_among_matched_ends,
  clean_win_rate_among_matched_ends,

  matched_end_rate,
  clean_matched_end_rate,

  win_rate_among_matched_ends
    - clean_win_rate_among_matched_ends
    AS win_rate_gap_all_vs_clean

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

ORDER BY
  tutorial_contaminated_attempt_rate DESC,
  ABS(win_rate_gap_all_vs_clean) DESC;
```

Truy vấn này giúp xem tutorial có làm lệch chỉ số thiết kế level không.

---

### 11.6 Kiểm tra nhãn chẩn đoán

```sql id="h8r69u"
SELECT
  progression_diagnostic_bucket,
  level_design_performance_bucket,
  primary_metric_signal,
  COUNT(*) AS level_row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

GROUP BY
  progression_diagnostic_bucket,
  level_design_performance_bucket,
  primary_metric_signal

ORDER BY
  level_row_count DESC;
```

Truy vấn này giúp xem toàn bộ hệ thống đang phân loại level như thế nào.

---

## 12. Cách sử dụng khuyến nghị

### 12.1 Bảng kiểm tra level ưu tiên

```sql id="fg6b76"
SELECT
  app_version,
  progression_step,
  level_name,

  reached_user_count,
  clean_matched_end_count,
  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  start_to_next_step_drop_rate,
  win_to_next_step_drop_rate,

  progression_diagnostic_bucket,
  level_design_performance_bucket,
  primary_metric_signal

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

ORDER BY
  CASE
    WHEN level_design_performance_bucket = 'severe_difficulty_risk' THEN 1
    WHEN progression_diagnostic_bucket = 'hybrid_difficulty_and_post_win_transition_risk' THEN 2
    WHEN progression_diagnostic_bucket = 'post_win_transition_risk' THEN 3
    WHEN level_design_performance_bucket = 'difficulty_risk' THEN 4
    ELSE 99
  END,
  progression_step;
```

---

### 12.2 Xem toàn bộ chỉ số thiết kế cốt lõi

```sql id="mx2r7q"
SELECT
  app_version,
  progression_step,
  level_name,

  reached_user_count,
  attempt_count,
  avg_attempt_count_per_reached_user,

  clean_matched_end_count,
  clean_win_rate_among_matched_ends,
  clean_fail_rate_among_matched_ends,

  avg_move_used_on_win,
  avg_move_used_on_fail,
  avg_move_left_on_win,

  avg_duration_sec_on_win,
  avg_duration_sec_on_fail,

  clean_top_fail_reason_summary

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

ORDER BY
  app_version,
  progression_step;
```

---

### 12.3 Xem level có thể quá dễ

```sql id="w4zugf"
SELECT
  app_version,
  progression_step,
  level_name,

  clean_matched_end_count,
  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  avg_move_left_on_win,

  level_design_performance_bucket

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

WHERE
  level_design_performance_bucket = 'possibly_too_easy'

ORDER BY
  clean_win_rate_among_matched_ends DESC;
```

---

## 13. Lưu ý khi sử dụng

### 13.1 Đây là view chẩn đoán, không phải quyết định cuối cùng

Các nhãn như:

```text id="jv7xpy"
difficulty_risk
possibly_too_easy
post_win_transition_risk
```

là nhãn hỗ trợ phân tích.

Không nên xem chúng là kết luận tuyệt đối. Cần kết hợp với:

* số lượng mẫu;
* video gameplay nếu có;
* feedback người chơi;
* thiết kế level thực tế;
* log lỗi;
* các thay đổi version.

---

### 13.2 Cần ưu tiên chỉ số sạch khi đánh giá độ khó

Khi đánh giá thiết kế level, nên ưu tiên các cột:

```text id="l4501u"
clean_matched_end_count
clean_win_rate_among_matched_ends
clean_fail_rate_among_matched_ends
clean_top_fail_reason_summary
```

Vì các cột này loại lượt chơi gần tutorial.

---

### 13.3 Một số chỉ số dựa trên `vw_level_attempts`

Các chỉ số attempt trong view này dựa trên `vw_level_attempts`.

Trong khi đó, phễu và cohort có thể dùng `vw_level_start_eff`.

Do đó, có thể có khác biệt giữa số người tới level trong phễu và số người có attempt, đặc biệt ở các trường hợp bắt đầu level được suy luận từ tutorial.

---

### 13.4 Ngưỡng chẩn đoán là ngưỡng phân tích ban đầu

Các điều kiện như:

```text id="4av751"
clean_win_rate_among_matched_ends <= 0.25
clean_win_rate_among_matched_ends <= 0.4
avg_attempt_count_per_reached_user >= 3
win_to_next_step_drop_rate >= 0.25
```

là ngưỡng khởi đầu để phân loại.

Khi game có thêm dữ liệu, team nên hiệu chỉnh ngưỡng theo thực tế.

---

## 14. Rủi ro nếu view sai logic

| Lỗi logic                      | Ảnh hưởng                                 |
| ------------------------------ | ----------------------------------------- |
| Sai mapping 10 level đầu       | Toàn bộ chẩn đoán gắn sai level.          |
| Sai cohort Level 1             | Mẫu phân tích 24 giờ bị sai.              |
| Sai `vw_level_attempts`        | Tỷ lệ thắng/thua và số lần thử sai.       |
| Không loại tutorial đúng       | Đánh giá sai độ khó level.                |
| Sai logic đi tiếp level sau    | Nhầm giữa khó level và rơi sau khi thắng. |
| Sai `End_level` coverage       | Nhầm tracking risk với design risk.       |
| Ngưỡng phân loại không phù hợp | Nhãn chẩn đoán có thể gây hiểu sai.       |
| Mẫu nhỏ nhưng diễn giải mạnh   | Dễ đưa ra quyết định sai.                 |

---

## 15. Mức độ quan trọng

```text id="nac4jb"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là nguồn chẩn đoán thiết kế level chi tiết nhất hiện tại;
* view này kết hợp phễu, timing, attempt, win rate, fail reason và tutorial;
* view này là đầu vào của bảng ưu tiên thiết kế `vw_f10_design_board_24h`;
* view này giúp phân biệt nhiều loại vấn đề: độ khó, tracking, tutorial, rơi sau thắng, và pacing;
* view này ảnh hưởng trực tiếp đến quyết định điều chỉnh 10 level đầu.

---

## 16. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="fdc5i6"
sql/ddl/current_live_analysis/vw_f10_design_diag_24h.sql
```

Cấu trúc logic chính của view:

```sql id="3yql9u"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`
AS
WITH analysis_params AS (...),

assignment AS (...),

level1_mapping AS (...),

level1_first_start_by_user AS (...),

matured_level1_cohort AS (...),

funnel_enriched AS (
  SELECT
    *
  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`
),

first10_attempts AS (
  SELECT
    -- attempt data trong 24 giờ sau Level 1
  FROM
    assignment
  INNER JOIN
    matured_level1_cohort
  INNER JOIN
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`
),

user_first_reach_by_step AS (...),

user_first_win_by_step AS (...),

user_win_to_next_step AS (...),

user_attempt_counts AS (...),

attempt_count_distribution AS (...),

fail_reason_counts AS (...),

top_fail_reason_summary AS (...),

clean_fail_reason_counts AS (...),

clean_top_fail_reason_summary AS (...),

attempt_metrics AS (...),

user_win_metrics AS (...),

diagnostics_joined AS (...),

diagnostics_classified AS (
  SELECT
    *,
    -- sample_size_status
    -- design_sample_size_status
    -- progression_diagnostic_bucket
    -- level_design_performance_bucket
    -- primary_metric_signal
  FROM
    diagnostics_joined
)

SELECT
  *,
  CURRENT_TIMESTAMP() AS diagnostics_view_query_time_utc

FROM
  diagnostics_classified;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[dim_f10_level_map]]
	- [[vw_level_start_eff]]
	- [[vw_level_attempts]]
	- [[vw_f10_funnel_timing_24h]]

- Downstream: [[vw_f10_design_board_24h]]

