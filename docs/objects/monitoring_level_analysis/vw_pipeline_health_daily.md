# vw_pipeline_health_daily

## 1. Mục đích

`vw_pipeline_health_daily` là view kiểm tra sức khỏe kỹ thuật hằng ngày của pipeline phân tích gameplay.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

“Pipeline” nghĩa là luồng xử lý dữ liệu, bắt đầu từ dữ liệu sự kiện GA4 thô, đi qua các view làm sạch, view ghép lượt chơi, view phân tích độ khó, view progression và cuối cùng là các view phục vụ báo cáo.

Mục đích chính của view này là trả lời câu hỏi:

```text id="u9y3sd"
Mỗi ngày, pipeline dữ liệu gameplay có dấu hiệu lỗi tracking, lỗi schema, lỗi join, lỗi progression hoặc lỗi dữ liệu quan trọng không?
```

View này không dùng để đánh giá gameplay hay độ khó trực tiếp. Nó dùng để kiểm tra chất lượng dữ liệu và độ ổn định của các view trong mart.

---

## 2. Vai trò trong hệ thống

`vw_pipeline_health_daily` thuộc nhóm:

```text id="kmsy4e"
monitoring_level_analysis
```

Tức là nhóm theo dõi và phân tích level.

Đây là view giám sát kỹ thuật cuối pipeline.

View này gom tín hiệu từ nhiều lớp:

```text id="scmvq4"
vw_level_events
vw_tutorial_events
vw_level_tutorial_flag
vw_level_attempts
vw_level_difficulty_daily
vw_user_level_reach
vw_level_progression_version
```

Vai trò chính:

* phát hiện ngày không có core event;
* phát hiện thiếu `app_version`;
* phát hiện thiếu tham số quan trọng ở `End_level`;
* phát hiện tỷ lệ ghép `Start_level` với `End_level` thấp;
* phát hiện attempt cross-version;
* phát hiện lỗi grain ở `vw_user_level_reach`;
* phát hiện metric bất hợp lệ trong progression mart;
* tạo trạng thái sức khỏe pipeline hằng ngày;
* tạo ghi chú tiếng Việt để người phân tích biết nên kiểm tra gì.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Ngày nào không có core gameplay hoặc tutorial event?
2. Ngày nào có `app_version` bị thiếu?
3. Ngày nào `Start_level` thiếu `level_name` hoặc `attempt_no`?
4. Ngày nào `End_level` thiếu `success`, `duration_sec`, hoặc `move_used`?
5. Normal N-level có thiếu tham số outcome không?
6. Tutorial event có thiếu `level_name` hoặc `tutorial_name` không?
7. Tỷ lệ `Start_level` gần tutorial là bao nhiêu?
8. Có bao nhiêu attempt ghép được `End_level`?
9. Tỷ lệ ghép attempt với `End_level` có thấp không?
10. Có attempt nào start ở một app version nhưng end ở app version khác không?
11. `vw_user_level_reach` có duplicate key không?
12. `vw_level_progression_version` có metric bất hợp lệ không?
13. Ngày nào pipeline có trạng thái cảnh báo?
14. Nếu có cảnh báo, cần kiểm tra phần nào trước?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_pipeline_health_daily` đại diện cho một ngày dữ liệu.

Có thể hiểu đơn giản:

```text id="esq8mc"
Một dòng = một event_date
```

View dùng thêm:

```text id="zk4x42"
event_table_suffix
```

để liên kết với ngày của bảng GA4 gốc.

---

## 5. Loại đối tượng

```text id="x97b82"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào nhiều nguồn trong mart:

| Nguồn                          | Vai trò kiểm tra                                                            |
| ------------------------------ | --------------------------------------------------------------------------- |
| `vw_level_events`              | Kiểm tra số lượng `Start_level`, `End_level`, tham số level và app version. |
| `vw_tutorial_events`           | Kiểm tra số lượng tutorial event và tham số tutorial.                       |
| `vw_level_tutorial_flag`       | Kiểm tra tỷ lệ `Start_level` gần tutorial và missing attempt.               |
| `vw_level_attempts`            | Kiểm tra matched end, unmatched start, win/fail, app version mismatch.      |
| `vw_level_difficulty_daily`    | Kiểm tra số row difficulty theo ngày và difficulty band.                    |
| `vw_user_level_reach`          | Kiểm tra user-level reach, duplicate key, level number, first win.          |
| `vw_level_progression_version` | Kiểm tra progression mart ở cấp global.                                     |

Trong tài liệu này, “global” nghĩa là chỉ số được tính trên toàn bộ view, không chia theo ngày.

---

## 7. Cách tạo danh sách ngày

View tạo danh sách ngày bằng cách lấy union từ nhiều nguồn daily khác nhau:

```text id="9f3cff"
level_events_daily
tutorial_events_daily
start_tutorial_flag_daily
attempts_daily
difficulty_daily
user_level_first_reach_daily
```

Điều này giúp một ngày vẫn xuất hiện nếu có dữ liệu ở một số lớp nhưng thiếu ở lớp khác.

---

## 8. Nhóm cột đầu ra

View có 86 cột, được chia thành các nhóm sau.

### 8.1 Nhóm ngày

```text id="se9z1p"
event_table_suffix
event_date
```

Ý nghĩa: xác định ngày dữ liệu.

---

### 8.2 Nhóm level events

```text id="u6l07u"
level_event_count
start_level_event_count
end_level_event_count
level_event_null_app_version_count
start_level_null_app_version_count
end_level_null_app_version_count
start_level_missing_level_name_count
start_level_missing_attempt_no_count
end_level_missing_level_name_count
end_level_missing_attempt_no_count
end_level_missing_success_count
end_level_missing_duration_sec_count
end_level_missing_move_used_count
end_level_null_move_left_count
app_version_count
```

Ý nghĩa: kiểm tra chất lượng dữ liệu từ `vw_level_events`.

---

### 8.3 Nhóm normal N-level End_level

```text id="zqil5n"
normal_n_end_level_count
normal_n_end_level_missing_success_count
normal_n_end_level_missing_duration_sec_count
normal_n_end_level_missing_move_used_count
normal_n_end_level_null_move_left_count
normal_n_end_level_null_app_version_count
```

Ý nghĩa: kiểm tra riêng các `End_level` của normal level dạng `Nxxx`.

Đây là nhóm quan trọng vì các tham số này ảnh hưởng trực tiếp đến difficulty mart.

---

### 8.4 Nhóm tutorial events

```text id="x7m66m"
tutorial_event_count
tutorial_start_event_count
tutorial_step_start_event_count
tutorial_end_event_count
tutorial_event_missing_level_name_count
tutorial_event_missing_tutorial_name_count
```

Ý nghĩa: kiểm tra dữ liệu tutorial.

---

### 8.5 Nhóm start tutorial flag

```text id="ktk53s"
start_level_with_level_name_count
start_level_flag_null_start_app_version_count
near_tutorial_start_count
non_tutorial_start_count
near_tutorial_start_rate
start_level_flag_missing_attempt_no_count
```

Ý nghĩa: kiểm tra kết quả từ `vw_level_tutorial_flag`.

---

### 8.6 Nhóm attempts

```text id="zbtnip"
attempt_count
attempt_user_count
matched_end_count
start_without_end_count
matched_end_rate
attempt_null_start_app_version_count
matched_attempt_null_end_app_version_count
cross_app_version_attempt_count
attempt_win_count
attempt_fail_count
normal_n_attempt_count
normal_n_non_tutorial_attempt_count
attempt_near_tutorial_start_count
normal_n_start_without_end_count
```

Ý nghĩa: kiểm tra dữ liệu lượt chơi từ `vw_level_attempts`.

---

### 8.7 Nhóm difficulty daily

```text id="b6yoba"
difficulty_daily_row_count
difficulty_daily_sufficient_sample_row_count
difficulty_daily_low_sample_row_count
difficulty_daily_tracking_risk_row_count
difficulty_daily_very_hard_row_count
difficulty_daily_hard_row_count
difficulty_daily_medium_row_count
difficulty_daily_easy_row_count
difficulty_daily_very_easy_row_count
```

Ý nghĩa: kiểm tra output của `vw_level_difficulty_daily` theo ngày.

---

### 8.8 Nhóm user level reach

```text id="84986d"
user_level_first_reach_row_count
user_level_first_reach_user_count
user_level_first_reach_level_count
user_level_first_reach_null_first_start_app_version_count
user_level_first_reach_null_level_number_count
user_level_first_reach_won_level_count
user_level_first_reach_first_win_time_count
user_level_first_reach_matured_24h_count
user_level_first_reach_not_matured_24h_count
user_level_first_reach_near_tutorial_count
user_level_first_reach_non_tutorial_count
user_level_first_reach_duplicate_key_count
```

Ý nghĩa: kiểm tra output của `vw_user_level_reach`.

---

### 8.9 Nhóm progression mart global

```text id="6o58r9"
progression_mart_row_count
progression_mart_sufficient_sample_row_count
progression_mart_low_sample_row_count
progression_mart_null_app_version_count
progression_mart_null_level_number_count
progression_mart_null_level_name_count
progression_mart_null_next_level_name_count
progression_mart_invalid_rate_count
progression_mart_start_rate_sum_mismatch_count
progression_mart_win_rate_sum_mismatch_count
progression_mart_invalid_count_relationship_count
progression_mart_sufficient_sample_full_drop_row_count
```

Ý nghĩa: kiểm tra `vw_level_progression_version`.

Lưu ý quan trọng: các cột progression này là chỉ số global và được lặp lại trên mọi dòng ngày, vì trong DDL chúng được `CROSS JOIN` vào bảng daily.

Không nên diễn giải các cột progression global như chỉ số riêng của từng ngày.

---

### 8.10 Nhóm trạng thái sức khỏe

```text id="dh26gx"
progression_sequence_check_status
progression_sequence_check_note
pipeline_health_status
pipeline_health_note
```

Ý nghĩa: trạng thái và ghi chú cuối cùng để người phân tích biết cần kiểm tra gì.

---

## 9. Logic `progression_sequence_check_status`

Cột:

```text id="trt4zn"
progression_sequence_check_status
```

có hai giá trị:

| Giá trị                   | Điều kiện                                                                      | Ý nghĩa                                      |
| ------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------- |
| `requires_sequence_check` | Có dòng progression đủ sample nhưng drop 100% cả start-to-next và win-to-next. | Cần kiểm tra giả định next level là `N + 1`. |
| `ok`                      | Không có cảnh báo trên.                                                        | Sequence check tổng quan ổn.                 |

Cảnh báo này là cảnh báo global, không phải cảnh báo riêng của từng ngày.

Nó thường liên quan đến giả định:

```text id="c5jfml"
next_level_name = N + 1
```

Nếu progression thực tế không đi theo chuỗi này, cần kiểm tra lại actual level sequence trước khi kết luận gameplay drop thật.

---

## 10. Logic `pipeline_health_status`

Cột:

```text id="wgrdzy"
pipeline_health_status
```

được tính theo thứ tự ưu tiên sau:

| Thứ tự | Trạng thái                                              | Điều kiện chính                                                               |
| -----: | ------------------------------------------------------- | ----------------------------------------------------------------------------- |
|      1 | `warning_no_core_events`                                | Không có `Start_level`, `End_level`, và tutorial event trong ngày.            |
|      2 | `warning_app_version_missing`                           | Có `app_version` bị NULL ở level hoặc progression pipeline.                   |
|      3 | `warning_end_app_version_missing_on_matched_attempt`    | Có matched attempt nhưng `end_app_version` bị NULL.                           |
|      4 | `warning_cross_app_version_attempt`                     | Có attempt start ở một app version nhưng end ở app version khác.              |
|      5 | `warning_normal_n_end_level_required_parameter_missing` | Normal N-level `End_level` thiếu `success`, `duration_sec`, hoặc `move_used`. |
|      6 | `warning_user_level_first_reach_duplicate_key`          | `vw_user_level_reach` có duplicate `user_pseudo_id + level_name`.             |
|      7 | `warning_user_level_first_reach_invalid_level_number`   | `vw_user_level_reach` có `level_number` NULL.                                 |
|      8 | `warning_progression_mart_invalid_metric`               | `vw_level_progression_version` có metric hoặc dimension bất hợp lệ.           |
|      9 | `warning_low_attempt_match_rate`                        | Có ít nhất 30 attempts nhưng `matched_end_rate < 0.5`.                        |
|     10 | `ok`                                                    | Không có cảnh báo kỹ thuật nào ở trên.                                        |

Thứ tự này rất quan trọng: nếu một ngày thỏa nhiều điều kiện cảnh báo, view chỉ trả cảnh báo đầu tiên theo thứ tự ưu tiên.

---

## 11. Logic `pipeline_health_note`

Cột:

```text id="licg6o"
pipeline_health_note
```

là ghi chú tiếng Việt tương ứng với `pipeline_health_status`.

Ví dụ:

| Status                                                  | Ý nghĩa ghi chú                                                          |
| ------------------------------------------------------- | ------------------------------------------------------------------------ |
| `warning_no_core_events`                                | Cần kiểm tra GA4 export, event tracking hoặc date range.                 |
| `warning_app_version_missing`                           | Cần kiểm tra `app_info.version` và propagation qua các view.             |
| `warning_cross_app_version_attempt`                     | Không nhất thiết là lỗi, nhưng cần chú ý khi attribution theo version.   |
| `warning_normal_n_end_level_required_parameter_missing` | Có thể ảnh hưởng trực tiếp tới difficulty mart.                          |
| `warning_low_attempt_match_rate`                        | Cần kiểm tra `End_level`, `attempt_no`, `level_name`, hoặc session flow. |
| `ok`                                                    | Daily technical pipeline health check OK.                                |

Trong tài liệu này, “attribution theo version” nghĩa là gán một hành vi hoặc kết quả cho phiên bản game nào.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Xem trạng thái pipeline theo ngày

```sql id="2ilzop"
SELECT
  event_date,
  pipeline_health_status,
  pipeline_health_note,
  progression_sequence_check_status,
  progression_sequence_check_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

ORDER BY
  event_date DESC;
```

Đây là truy vấn chính để theo dõi sức khỏe pipeline hằng ngày.

---

### 12.2 Lọc các ngày có cảnh báo

```sql id="xupr4d"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

WHERE
  pipeline_health_status != 'ok'
  OR progression_sequence_check_status != 'ok'

ORDER BY
  event_date DESC;
```

---

### 12.3 Kiểm tra thiếu tham số `End_level` của normal N-level

```sql id="8m7kqb"
SELECT
  event_date,
  normal_n_end_level_count,
  normal_n_end_level_missing_success_count,
  normal_n_end_level_missing_duration_sec_count,
  normal_n_end_level_missing_move_used_count,
  normal_n_end_level_null_move_left_count,
  pipeline_health_status,
  pipeline_health_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

WHERE
  normal_n_end_level_missing_success_count > 0
  OR normal_n_end_level_missing_duration_sec_count > 0
  OR normal_n_end_level_missing_move_used_count > 0

ORDER BY
  event_date DESC;
```

---

### 12.4 Kiểm tra matched end rate theo ngày

```sql id="196vny"
SELECT
  event_date,
  attempt_count,
  matched_end_count,
  start_without_end_count,
  matched_end_rate,
  pipeline_health_status,
  pipeline_health_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

ORDER BY
  event_date DESC;
```

Nếu `attempt_count >= 30` và `matched_end_rate < 0.5`, view sẽ cảnh báo `warning_low_attempt_match_rate`.

---

### 12.5 Kiểm tra app version bị thiếu

```sql id="8n352y"
SELECT
  event_date,

  level_event_null_app_version_count,
  start_level_null_app_version_count,
  end_level_null_app_version_count,
  start_level_flag_null_start_app_version_count,
  attempt_null_start_app_version_count,
  user_level_first_reach_null_first_start_app_version_count,
  progression_mart_null_app_version_count,

  pipeline_health_status,
  pipeline_health_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

WHERE
  level_event_null_app_version_count > 0
  OR start_level_null_app_version_count > 0
  OR end_level_null_app_version_count > 0
  OR start_level_flag_null_start_app_version_count > 0
  OR attempt_null_start_app_version_count > 0
  OR user_level_first_reach_null_first_start_app_version_count > 0
  OR progression_mart_null_app_version_count > 0

ORDER BY
  event_date DESC;
```

---

### 12.6 Kiểm tra progression mart global

```sql id="gwbxh1"
SELECT DISTINCT
  progression_mart_row_count,
  progression_mart_sufficient_sample_row_count,
  progression_mart_low_sample_row_count,
  progression_mart_null_app_version_count,
  progression_mart_null_level_number_count,
  progression_mart_null_level_name_count,
  progression_mart_null_next_level_name_count,
  progression_mart_invalid_rate_count,
  progression_mart_start_rate_sum_mismatch_count,
  progression_mart_win_rate_sum_mismatch_count,
  progression_mart_invalid_count_relationship_count,
  progression_mart_sufficient_sample_full_drop_row_count,
  progression_sequence_check_status,
  progression_sequence_check_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`;
```

Dùng `DISTINCT` vì các chỉ số progression global được lặp lại trên mọi ngày.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Daily health dashboard

```sql id="enxsh2"
SELECT
  event_date,
  level_event_count,
  start_level_event_count,
  end_level_event_count,
  tutorial_event_count,
  attempt_count,
  matched_end_rate,
  difficulty_daily_row_count,
  user_level_first_reach_row_count,
  pipeline_health_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

ORDER BY
  event_date DESC;
```

---

### 13.2 Checklist kỹ thuật mỗi ngày

```sql id="gxcjza"
SELECT
  event_date,

  CASE
    WHEN start_level_event_count = 0 THEN 'Check Start_level'
    ELSE 'OK'
  END AS start_level_check,

  CASE
    WHEN end_level_event_count = 0 THEN 'Check End_level'
    ELSE 'OK'
  END AS end_level_check,

  CASE
    WHEN tutorial_event_count = 0 THEN 'Check Tutorial events'
    ELSE 'OK'
  END AS tutorial_event_check,

  CASE
    WHEN matched_end_rate < 0.5 AND attempt_count >= 30 THEN 'Check attempt matching'
    ELSE 'OK'
  END AS attempt_matching_check,

  pipeline_health_status,
  pipeline_health_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

ORDER BY
  event_date DESC;
```

---

### 13.3 Theo dõi ảnh hưởng tutorial

```sql id="8ycqcj"
SELECT
  event_date,
  start_level_with_level_name_count,
  near_tutorial_start_count,
  non_tutorial_start_count,
  near_tutorial_start_rate,
  attempt_near_tutorial_start_count,
  normal_n_non_tutorial_attempt_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`

ORDER BY
  event_date DESC;
```

---

## 14. Lưu ý khi sử dụng

### 14.1 Đây là view kiểm tra kỹ thuật, không phải view đánh giá gameplay

`pipeline_health_status = ok` không có nghĩa là gameplay tốt.

Nó chỉ nghĩa là các kiểm tra kỹ thuật chính không phát hiện lỗi rõ ràng.

---

### 14.2 Cảnh báo progression sequence là cảnh báo global

Các cột:

```text id="k7xtw0"
progression_sequence_check_status
progression_sequence_check_note
progression_mart_*
```

không phải chỉ số riêng cho từng ngày. Chúng được tính trên toàn bộ `vw_level_progression_version` và lặp lại trên các dòng ngày.

---

### 14.3 Một ngày có thể có nhiều vấn đề nhưng chỉ hiện một status chính

`pipeline_health_status` dùng logic ưu tiên. Nếu một ngày có nhiều lỗi, view trả về lỗi đầu tiên theo thứ tự ưu tiên.

Khi cần điều tra sâu, nên xem các cột count chi tiết thay vì chỉ đọc status.

---

### 14.4 `cross_app_version_attempt_count` không nhất thiết là lỗi

Nếu người chơi cập nhật app giữa lúc bắt đầu và kết thúc level, start app version và end app version có thể khác nhau.

Tuy nhiên, chỉ số này cần được theo dõi vì nó ảnh hưởng đến attribution theo version.

---

### 14.5 `move_left` NULL không luôn là lỗi nghiêm trọng

View có đếm:

```text id="zfiopd"
end_level_null_move_left_count
normal_n_end_level_null_move_left_count
```

Nhưng warning mạnh chỉ dùng cho thiếu:

```text id="uo47ay"
success
duration_sec
move_used
```

Vì `move_left` có thể không phải tham số bắt buộc trong mọi trường hợp, tùy logic tracking.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                                         | Ảnh hưởng                                          |
| ------------------------------------------------- | -------------------------------------------------- |
| Không lấy đủ nguồn daily                          | Bỏ sót ngày có vấn đề.                             |
| Sai điều kiện warning                             | Cảnh báo sai hoặc bỏ sót lỗi.                      |
| Không phân biệt daily và global                   | Diễn giải sai progression warning.                 |
| Dựa vào status mà không xem count chi tiết        | Có thể bỏ qua lỗi phụ.                             |
| Sai matched_end_rate threshold                    | Cảnh báo attempt matching không phù hợp.           |
| Không kiểm tra required params của normal N-level | Difficulty mart có thể sai mà không được cảnh báo. |
| Không kiểm tra duplicate user-level key           | Progression mart có thể sai grain.                 |

Trong tài liệu này, “grain” nghĩa là độ chi tiết của mỗi dòng dữ liệu.

---

## 16. Mức độ quan trọng

```text id="utqx7i"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là lớp giám sát kỹ thuật cuối pipeline;
* view này giúp phát hiện lỗi tracking trước khi team đọc dashboard;
* view này kiểm tra nhiều view quan trọng cùng lúc;
* view này giúp phân biệt lỗi dữ liệu với vấn đề gameplay;
* view này nên được xem thường xuyên khi có build mới hoặc thay đổi tracking.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="23lfze"
sql/ddl/monitoring_level_analysis/vw_pipeline_health_daily.sql
```

Cấu trúc logic chính của view:

```sql id="l03nbt"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_pipeline_health_daily`
AS
WITH level_events_daily AS (...),

normal_n_end_level_daily AS (...),

tutorial_events_daily AS (...),

start_tutorial_flag_daily AS (...),

attempts_daily AS (...),

difficulty_daily AS (...),

user_level_first_reach_daily AS (...),

user_level_first_reach_duplicate_daily AS (...),

progression_mart_health AS (...),

all_dates AS (
  -- union tất cả ngày có dữ liệu từ các nguồn daily
),

final_metrics AS (
  SELECT
    all_dates.event_table_suffix,
    all_dates.event_date,

    -- level events metrics
    -- tutorial metrics
    -- start tutorial flag metrics
    -- attempt metrics
    -- difficulty daily metrics
    -- user level reach metrics
    -- progression global metrics

  FROM
    all_dates

  LEFT JOIN level_events_daily
  LEFT JOIN normal_n_end_level_daily
  LEFT JOIN tutorial_events_daily
  LEFT JOIN start_tutorial_flag_daily
  LEFT JOIN attempts_daily
  LEFT JOIN difficulty_daily
  LEFT JOIN user_level_first_reach_daily
  LEFT JOIN user_level_first_reach_duplicate_daily

  CROSS JOIN progression_mart_health
)

SELECT
  *,

  CASE
    -- progression_sequence_check_status
  END AS progression_sequence_check_status,

  CASE
    -- progression_sequence_check_note
  END AS progression_sequence_check_note,

  CASE
    -- pipeline_health_status
  END AS pipeline_health_status,

  CASE
    -- pipeline_health_note
  END AS pipeline_health_note

FROM
  final_metrics;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[vw_level_events]]
	- [[vw_tutorial_events]]
	- [[vw_level_tutorial_flag]]
	- [[vw_level_attempts]]
	- [[vw_user_level_reach]]
	- [[vw_level_difficulty_daily]]
	- [[vw_level_progression_version]]

- Downstream: Không có

