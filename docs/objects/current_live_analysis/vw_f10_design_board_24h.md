# vw_f10_design_board_24h

## 1. Mục đích

`vw_f10_design_board_24h` là view tạo bảng ưu tiên hành động cho 10 level đầu.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này lấy toàn bộ dữ liệu chẩn đoán từ:

```text id="4ampv5"
vw_f10_design_diag_24h
```

Sau đó bổ sung các trường phục vụ ưu tiên hành động:

```text id="qfglwe"
sort_priority
priority_tier
risk_family
recommended_owner
final_action_note
final_metric_signal
```

Mục tiêu chính là biến bảng chẩn đoán chi tiết thành một bảng có thể dùng trực tiếp để ra quyết định:

* level nào cần kiểm tra trước;
* vấn đề thuộc nhóm nào;
* ai nên chịu trách nhiệm chính;
* cần xem chỉ số nào;
* hành động đề xuất là gì.

---

## 2. Vai trò trong hệ thống

`vw_f10_design_board_24h` thuộc nhóm:

```text id="6bs5b9"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là lớp hành động nằm phía sau view chẩn đoán:

```text id="ipokbs"
vw_f10_design_diag_24h
  └── vw_f10_design_board_24h
```

Nếu `vw_f10_design_diag_24h` trả lời câu hỏi “level đang có vấn đề gì?”, thì `vw_f10_design_board_24h` trả lời câu hỏi “nên ưu tiên xử lý level nào trước?”.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Level nào cần ưu tiên kiểm tra cao nhất?
2. Level nào thuộc nhóm P0, P1, P2, P3?
3. Vấn đề chính của level là gì?
4. Chủ sở hữu hành động nên là Game Design, UX, Analytics hay Tracking?
5. Level nào có rủi ro difficulty?
6. Level nào có rủi ro post-win transition?
7. Level nào có rủi ro tracking?
8. Level nào có rủi ro tutorial hoặc onboarding measurement?
9. Level nào có thể quá dễ?
10. Chỉ số chính cần xem khi xử lý level là gì?
11. Ghi chú hành động đề xuất cho từng level là gì?

Trong tài liệu này:

* “difficulty” nghĩa là độ khó level;
* “post-win transition” nghĩa là trải nghiệm sau khi người chơi thắng level và chuyển sang bước tiếp theo;
* “tracking” nghĩa là hệ thống ghi nhận sự kiện dữ liệu.

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_f10_design_board_24h` đại diện cho một level trong 10 level đầu của một phiên bản game.

Có thể hiểu đơn giản:

```text id="87rpp0"
Một dòng = một app_version + một progression_step + một level_name
```

Độ chi tiết này giống với:

```text id="fsykqp"
vw_f10_design_diag_24h
```

---

## 5. Loại đối tượng

```text id="c2evma"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

View có cột:

```text id="x2rs7s"
priority_board_view_query_time_utc
```

để ghi nhận thời điểm bảng ưu tiên được truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này chỉ phụ thuộc trực tiếp vào:

```text id="d7gmz7"
vw_f10_design_diag_24h
```

Nó không tự tính lại phễu, timing hoặc attempt metrics.

Nó chỉ:

1. đọc toàn bộ cột từ `vw_f10_design_diag_24h`;
2. thêm lớp phân loại ưu tiên;
3. sắp xếp ý nghĩa hành động bằng các cột mới.

---

## 7. Các trường ưu tiên mới

### 7.1 `sort_priority`

`sort_priority` là số dùng để sắp xếp mức độ ưu tiên.

Số càng nhỏ thì mức ưu tiên càng cao.

| Giá trị | Điều kiện chính                                                                     | Ý nghĩa                                                                   |
| ------: | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
|       1 | `hybrid_difficulty_and_post_win_transition_risk`                                    | Vấn đề kết hợp: vừa có rủi ro độ khó vừa có rủi ro chuyển tiếp sau thắng. |
|       2 | `severe_difficulty_risk`                                                            | Rủi ro độ khó rất nặng.                                                   |
|       3 | `post_win_transition_risk`                                                          | Người chơi thắng rồi nhưng không đi tiếp.                                 |
|       4 | `difficulty_risk`                                                                   | Rủi ro độ khó.                                                            |
|       5 | `difficulty_or_pre_win_churn_risk`                                                  | Có thể do khó hoặc người chơi rời trước khi thắng.                        |
|       6 | `pre_win_churn_or_difficulty_risk`                                                  | Rơi cao trước khi đi tiếp, có thể liên quan đến khó hoặc friction.        |
|       7 | `friction_or_difficulty_risk`                                                       | Có ma sát hoặc độ khó cao.                                                |
|       8 | `tracking_risk_low_matched_end_rate` hoặc `onboarding_or_tutorial_measurement_risk` | Rủi ro tracking hoặc tutorial measurement.                                |
|       9 | `tracking_risk_low_clean_matched_end_rate` hoặc `possibly_too_easy`                 | Rủi ro tracking sạch hoặc level có thể quá dễ.                            |
|      20 | Mẫu chưa đủ                                                                         | Chỉ nên theo dõi.                                                         |
|      99 | Không có tín hiệu khẩn cấp                                                          | Theo dõi bình thường.                                                     |

Trong tài liệu này, “friction” nghĩa là ma sát trải nghiệm, tức là những điểm làm người chơi chậm lại, khó hiểu, khó thao tác, hoặc dễ bỏ cuộc.

---

### 7.2 `priority_tier`

`priority_tier` là nhãn ưu tiên dạng ngắn.

| Giá trị | Ý nghĩa                                            |
| ------- | -------------------------------------------------- |
| `P0`    | Cần kiểm tra rất sớm. Đây là nhóm rủi ro cao nhất. |
| `P1`    | Cần ưu tiên kiểm tra.                              |
| `P2`    | Cần theo dõi hoặc kiểm tra sau nhóm P0/P1.         |
| `P3`    | Theo dõi, mẫu thấp, hoặc chưa có tín hiệu mạnh.    |

Logic chính:

```text id="0axm4p"
P0 = hybrid issue hoặc severe difficulty
P1 = post-win transition, difficulty, hoặc difficulty/pre-win churn
P2 = pre-win churn, tracking risk, tutorial risk, friction, possibly too easy
P3 = low sample hoặc monitor
```

---

### 7.3 `risk_family`

`risk_family` là nhóm rủi ro chính của level.

Các giá trị chính:

| Giá trị                              | Ý nghĩa                                                        |
| ------------------------------------ | -------------------------------------------------------------- |
| `hybrid_difficulty_and_transition`   | Vấn đề kết hợp giữa độ khó và chuyển tiếp sau thắng.           |
| `severe_difficulty`                  | Độ khó rất cao.                                                |
| `post_win_transition`                | Người chơi thắng nhưng không đi tiếp.                          |
| `difficulty`                         | Rủi ro độ khó.                                                 |
| `difficulty_or_pre_win_churn`        | Có thể do độ khó hoặc người chơi rơi trước khi thắng.          |
| `pre_win_churn_or_difficulty`        | Người chơi rơi trước khi đi tiếp, có thể liên quan đến độ khó. |
| `friction_or_difficulty`             | Ma sát trải nghiệm hoặc độ khó.                                |
| `onboarding_or_tutorial_measurement` | Rủi ro do onboarding hoặc tutorial làm nhiễu dữ liệu.          |
| `tracking_or_data_quality`           | Rủi ro tracking hoặc chất lượng dữ liệu.                       |
| `possibly_too_easy`                  | Level có thể quá dễ.                                           |
| `low_sample_monitor`                 | Mẫu thấp, chỉ nên theo dõi.                                    |
| `monitor`                            | Chưa có tín hiệu rủi ro mạnh.                                  |

---

### 7.4 `recommended_owner`

`recommended_owner` gợi ý nhóm nên kiểm tra chính.

Các giá trị chính:

| Giá trị                        | Ý nghĩa                                                    |
| ------------------------------ | ---------------------------------------------------------- |
| `Analytics / Tracking`         | Nên kiểm tra tracking hoặc chất lượng dữ liệu trước.       |
| `Game Design + UX`             | Nên kiểm tra thiết kế gameplay và trải nghiệm chuyển tiếp. |
| `Game Design + UX + Analytics` | Vấn đề phức hợp, cần phối hợp nhiều nhóm.                  |
| `Game Design`                  | Chủ yếu là vấn đề thiết kế level.                          |
| `Game Design / Analytics`      | Cần theo dõi thêm hoặc phân tích bổ sung.                  |

Trong tài liệu này, “UX” nghĩa là trải nghiệm người dùng, ví dụ luồng màn hình, nút bấm, chuyển cảnh, animation, cảm giác sau khi thắng level.

---

### 7.5 `final_action_note`

`final_action_note` là ghi chú hành động tự động.

Cột này hiện đang dùng tiếng Anh trong dữ liệu vì được viết trực tiếp trong SQL.

Ví dụ ý nghĩa:

| Nhóm vấn đề         | Hành động gợi ý                                                           |
| ------------------- | ------------------------------------------------------------------------- |
| Hybrid issue        | Kiểm tra độ khó, trải nghiệm thắng, phần thưởng và chuyển tiếp level sau. |
| Severe difficulty   | Kiểm tra số move, blocker, bố cục bàn chơi, mục tiêu và lý do thua.       |
| Post-win transition | Kiểm tra reward, animation, nút đi tiếp, home return, fatigue.            |
| Tracking risk       | Kiểm tra ghép `Start_level` / `End_level` và sự kiện thiếu.               |
| Possibly too easy   | Kiểm tra level có chủ ý dễ hay đang bị tune quá thấp.                     |
| Low sample          | Chỉ theo dõi, chưa nên ra quyết định mạnh.                                |

Nếu dashboard hướng tới người dùng nội bộ tiếng Việt, nên cân nhắc tạo thêm bảng dịch nhãn hành động sang tiếng Việt.

---

### 7.6 `final_metric_signal`

`final_metric_signal` cho biết nhóm chỉ số chính cần xem.

Ví dụ:

| Giá trị                                                                  | Nên xem                                        |
| ------------------------------------------------------------------------ | ---------------------------------------------- |
| `start_to_next_step_drop_rate + win_to_next_step_drop_rate`              | Rơi từ start sang next và rơi sau khi thắng.   |
| `clean_win_rate_among_matched_ends + avg_attempt_count_per_reached_user` | Tỷ lệ thắng sạch và số lần thử trung bình.     |
| `win_to_next_step_drop_rate`                                             | Rơi sau khi thắng.                             |
| `start_to_next_step_drop_rate + avg_attempt_count_per_reached_user`      | Rơi sang bước sau và áp lực retry.             |
| `matched_end_rate + tutorial_contaminated_attempt_rate`                  | Chất lượng tracking và tutorial contamination. |
| `monitoring_metrics`                                                     | Chỉ số theo dõi chung.                         |

---

## 8. Nhóm cột đầu ra

View có 104 cột, gồm 6 cột ưu tiên mới và phần lớn cột kế thừa từ `vw_f10_design_diag_24h`.

### 8.1 Nhóm ưu tiên hành động

```text id="n9a2kp"
sort_priority
priority_tier
risk_family
final_metric_signal
recommended_owner
final_action_note
```

Ý nghĩa: giúp sắp xếp và giao việc.

---

### 8.2 Nhóm định danh level

```text id="bnw3qo"
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

Ý nghĩa: xác định level trong 10 level đầu.

---

### 8.3 Nhóm nhãn chẩn đoán kế thừa

```text id="ergn83"
sample_size_status
design_sample_size_status
progression_diagnostic_bucket
level_design_performance_bucket
primary_metric_signal
```

Ý nghĩa: nhãn chẩn đoán được tạo ở `vw_f10_design_diag_24h`.

---

### 8.4 Nhóm cohort và reach

```text id="h24pqm"
level1_cohort_user_count
matured_level1_cohort_user_count
not_matured_level1_cohort_user_count
reached_user_count
matured_reached_user_count
not_matured_reached_user_count
attempt_reached_user_count
```

Ý nghĩa: số người chơi trong cohort Level 1 và số người tới từng level.

---

### 8.5 Nhóm chuyển tiếp sang bước tiếp theo

```text id="2ln4zh"
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

### 8.6 Nhóm phễu tổng hợp

```text id="jn6q70"
previous_step_user_count
drop_from_previous_step_user_count
conversion_rate_from_previous_step
drop_rate_from_previous_step
funnel_start_user_count
cumulative_drop_user_count
cumulative_conversion_rate
cumulative_drop_rate
```

Ý nghĩa: đo rơi từ bước trước và rơi cộng dồn.

---

### 8.7 Nhóm thắng theo người chơi

```text id="knc0sx"
user_with_win_count
user_without_win_count
user_win_rate
user_without_win_rate
```

Ý nghĩa: đo người chơi có từng thắng level hay không.

---

### 8.8 Nhóm lượt chơi

```text id="txbi5r"
attempt_count
matched_end_count
start_without_end_count
matched_end_rate
start_without_end_rate
win_attempt_count
fail_attempt_count
win_rate_among_matched_ends
fail_rate_among_matched_ends
```

Ý nghĩa: đo kết quả theo lượt chơi.

---

### 8.9 Nhóm dữ liệu sạch, loại tutorial

```text id="xzv8bw"
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

Ý nghĩa: dùng để đọc thiết kế level sau khi loại dữ liệu gần tutorial.

---

### 8.10 Nhóm retry, move, duration và fail reason

```text id="egb2vd"
avg_attempt_count_per_reached_user
median_attempt_count_per_reached_user
p75_attempt_count_per_reached_user
p90_attempt_count_per_reached_user
avg_move_used_on_win
avg_move_used_on_fail
avg_move_left_on_win
avg_duration_sec_on_win
avg_duration_sec_on_fail
top_fail_reason_summary
clean_top_fail_reason_summary
```

Ý nghĩa: phục vụ phân tích độ khó, số lần thử, số bước, thời lượng và lý do thua.

---

### 8.11 Nhóm timing

```text id="qkz7zx"
level1_to_step_timing_sample_user_count
avg_seconds_from_level1_to_step
median_seconds_from_level1_to_step
p90_seconds_from_level1_to_step
previous_step_to_step_timing_sample_user_count
avg_seconds_from_previous_step_to_step
median_seconds_from_previous_step_to_step
p90_seconds_from_previous_step_to_step
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

Ý nghĩa: đo thời gian đi tới level và thời gian bên trong level.

---

### 8.12 Nhóm thời điểm tính view

```text id="ym6j01"
first_level1_start_time_utc
last_level1_start_time_utc
funnel_view_query_time_utc
timing_view_query_time_utc
diagnostics_view_query_time_utc
priority_board_view_query_time_utc
```

Ý nghĩa: ghi nhận mốc thời gian dữ liệu và thời điểm tính view.

---

## 9. Cách đọc ưu tiên

### 9.1 Đọc theo `priority_tier`

Thứ tự đọc khuyến nghị:

```text id="p6h3f1"
P0 → P1 → P2 → P3
```

Trong đó:

* `P0`: kiểm tra ngay hoặc sớm nhất;
* `P1`: ưu tiên cao;
* `P2`: theo dõi hoặc kiểm tra sau P0/P1;
* `P3`: mẫu thấp hoặc chưa có tín hiệu mạnh.

---

### 9.2 Đọc theo `sort_priority`

Khi cần sắp xếp chính xác hơn trong cùng một tier, dùng:

```text id="mp9i15"
sort_priority ASC
```

Số càng nhỏ thì càng quan trọng.

---

### 9.3 Đọc theo `recommended_owner`

`recommended_owner` giúp phân loại việc cho team:

| Owner                          | Khi nào nên ưu tiên                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------- |
| `Analytics / Tracking`         | Khi cần kiểm tra tracking hoặc chất lượng dữ liệu trước.                           |
| `Game Design`                  | Khi chỉ số gợi ý vấn đề độ khó hoặc tuning level.                                  |
| `Game Design + UX`             | Khi người chơi thắng rồi nhưng không đi tiếp.                                      |
| `Game Design + UX + Analytics` | Khi vấn đề vừa liên quan độ khó, vừa liên quan chuyển tiếp, vừa có yếu tố dữ liệu. |

---

## 10. Kiểm tra chất lượng dữ liệu khuyến nghị

### 10.1 Kiểm tra số dòng có khớp với view chẩn đoán không

```sql id="xme488"
SELECT
  'vw_f10_design_diag_24h' AS source_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`

UNION ALL

SELECT
  'vw_f10_design_board_24h' AS source_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`;
```

Kỳ vọng:

```text id="ar6c9q"
Hai row_count bằng nhau
```

Vì `vw_f10_design_board_24h` chỉ thêm lớp ưu tiên lên `vw_f10_design_diag_24h`.

---

### 10.2 Kiểm tra phân bố priority tier

```sql id="e1ol95"
SELECT
  priority_tier,
  COUNT(*) AS level_row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

GROUP BY
  priority_tier

ORDER BY
  priority_tier;
```

Truy vấn này giúp xem toàn bộ 10 level đầu đang rơi vào nhóm ưu tiên nào.

---

### 10.3 Kiểm tra phân bố owner đề xuất

```sql id="34jspe"
SELECT
  recommended_owner,
  COUNT(*) AS level_row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

GROUP BY
  recommended_owner

ORDER BY
  level_row_count DESC;
```

Truy vấn này giúp biết vấn đề đang nghiêng về design, UX hay tracking.

---

### 10.4 Kiểm tra các level P0 và P1

```sql id="8epxxb"
SELECT
  priority_tier,
  sort_priority,
  risk_family,

  app_version,
  progression_step,
  level_name,

  reached_user_count,
  clean_matched_end_count,
  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  start_to_next_step_drop_rate,
  win_to_next_step_drop_rate,

  recommended_owner,
  final_metric_signal,
  final_action_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

WHERE
  priority_tier IN ('P0', 'P1')

ORDER BY
  sort_priority,
  app_version,
  progression_step;
```

Đây là truy vấn chính để lấy danh sách level cần kiểm tra trước.

---

### 10.5 Kiểm tra level có rủi ro tracking

```sql id="tw5cif"
SELECT
  priority_tier,
  sort_priority,
  risk_family,

  app_version,
  progression_step,
  level_name,

  matched_end_rate,
  clean_matched_end_rate,
  level_end_event_coverage_rate,
  tutorial_contaminated_attempt_rate,

  recommended_owner,
  final_action_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

WHERE
  recommended_owner = 'Analytics / Tracking'

ORDER BY
  sort_priority,
  matched_end_rate ASC;
```

Truy vấn này giúp tách các vấn đề cần kiểm tra dữ liệu trước khi chỉnh level.

---

## 11. Cách sử dụng khuyến nghị

### 11.1 Bảng ưu tiên chính cho dashboard

```sql id="izt2p1"
SELECT
  priority_tier,
  sort_priority,
  risk_family,

  app_version,
  progression_step,
  level_name,

  reached_user_count,
  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  start_to_next_step_drop_rate,
  win_to_next_step_drop_rate,

  recommended_owner,
  final_metric_signal,
  final_action_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

ORDER BY
  sort_priority,
  app_version,
  progression_step;
```

---

### 11.2 Bảng việc cho Game Design

```sql id="fe3p4t"
SELECT
  priority_tier,
  sort_priority,
  risk_family,

  app_version,
  progression_step,
  level_name,

  clean_win_rate_among_matched_ends,
  avg_attempt_count_per_reached_user,
  p90_attempt_count_per_reached_user,
  clean_top_fail_reason_summary,

  final_action_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

WHERE
  recommended_owner = 'Game Design'

ORDER BY
  sort_priority,
  progression_step;
```

---

### 11.3 Bảng việc cho UX và chuyển tiếp sau thắng

```sql id="y20hi3"
SELECT
  priority_tier,
  sort_priority,
  risk_family,

  app_version,
  progression_step,
  level_name,

  won_user_count,
  next_step_reached_after_win_user_count,
  win_to_next_step_rate,
  win_to_next_step_drop_rate,

  final_action_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`

WHERE
  recommended_owner IN (
    'Game Design + UX',
    'Game Design + UX + Analytics'
  )

ORDER BY
  sort_priority,
  progression_step;
```

---

## 12. Lưu ý khi sử dụng

### 12.1 View này là bảng ưu tiên, không phải bảng nguồn

Nếu cần phân tích chi tiết cách tính từng chỉ số, xem:

```text id="xhyvgo"
vw_f10_design_diag_24h
```

`vw_f10_design_board_24h` chỉ thêm lớp ưu tiên và hành động đề xuất.

---

### 12.2 Không nên ra quyết định chỉ dựa trên `priority_tier`

`priority_tier` giúp sắp xếp công việc, nhưng không thay thế phân tích chi tiết.

Trước khi sửa level, nên kiểm tra thêm:

* sample size;
* clean sample size;
* win rate sạch;
* attempt count;
* fail reason;
* replay hoặc gameplay thực tế nếu có;
* thay đổi thiết kế gần đây;
* khả năng lỗi tracking.

---

### 12.3 Cột hành động hiện đang bằng tiếng Anh

`final_action_note` hiện chứa câu tiếng Anh vì được hard-code trong SQL.

Trong tài liệu này, “hard-code” nghĩa là viết trực tiếp giá trị cố định vào câu truy vấn.

Nếu dashboard phục vụ team nội bộ bằng tiếng Việt, nên tạo thêm một lớp dịch hoặc cột hành động tiếng Việt trong tương lai.

---

### 12.4 P3 không có nghĩa là không quan trọng

`P3` có thể là:

* level chưa đủ mẫu;
* level cần theo dõi;
* level chưa có tín hiệu bất thường theo ngưỡng hiện tại.

Nếu level là level chiến lược hoặc có feedback người chơi, vẫn nên kiểm tra dù đang là P3.

---

## 13. Rủi ro nếu view sai logic

| Lỗi logic                                      | Ảnh hưởng                                       |
| ---------------------------------------------- | ----------------------------------------------- |
| Sai mapping từ diagnostic bucket sang priority | Ưu tiên xử lý bị sai.                           |
| Sai recommended owner                          | Giao nhầm việc cho team.                        |
| Sai final action note                          | Team kiểm tra sai hướng.                        |
| Không phân biệt tracking risk và design risk   | Có thể chỉnh level trong khi lỗi nằm ở dữ liệu. |
| Quá tin vào priority tier                      | Dễ bỏ qua bối cảnh thiết kế thực tế.            |
| Không kiểm tra sample size                     | Có thể sửa level dựa trên dữ liệu quá ít.       |

---

## 14. Mức độ quan trọng

```text id="6xx5t4"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là bảng hành động trực tiếp cho 10 level đầu;
* view này giúp ưu tiên công việc giữa nhiều loại rủi ro;
* view này gom các chỉ số phức tạp thành nhãn dễ xử lý;
* view này có thể được dùng trực tiếp trong dashboard hoặc review meeting;
* nếu logic ưu tiên sai, team có thể tập trung vào sai level hoặc sai loại vấn đề.

---

## 15. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="3qj5zb"
sql/ddl/current_live_analysis/vw_f10_design_board_24h.sql
```

Cấu trúc logic chính của view:

```sql id="0e8o1m"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_board_24h`
AS
WITH diagnostics AS (
  SELECT
    *

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_design_diag_24h`
),

priority_classified AS (
  SELECT
    CASE
      -- sort_priority
    END AS sort_priority,

    CASE
      -- priority_tier
    END AS priority_tier,

    CASE
      -- risk_family
    END AS risk_family,

    CASE
      -- recommended_owner
    END AS recommended_owner,

    CASE
      -- final_action_note
    END AS final_action_note,

    CASE
      -- final_metric_signal
    END AS final_metric_signal,

    diagnostics.*

  FROM
    diagnostics
)

SELECT
  sort_priority,
  priority_tier,
  risk_family,

  app_version,
  progression_step,
  progression_slot_id,
  level_name,
  level_design_id,
  level_type,

  -- các nhãn chẩn đoán
  -- các chỉ số funnel
  -- các chỉ số attempt
  -- các chỉ số clean
  -- các chỉ số timing
  -- các mốc thời gian view

  CURRENT_TIMESTAMP() AS priority_board_view_query_time_utc

FROM
  priority_classified;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: [[vw_f10_design_diag_24h]]

- Downstream: Không có

