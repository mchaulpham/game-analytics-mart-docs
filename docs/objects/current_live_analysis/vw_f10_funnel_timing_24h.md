# vw_f10_funnel_timing_24h

## 1. Mục đích

`vw_f10_funnel_timing_24h` là view kết hợp phễu 10 level đầu với các chỉ số thời gian tương ứng.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này kết hợp hai nhóm thông tin:

```text id="3lx6f4"
1. Chỉ số phễu từ vw_f10_funnel_24h
2. Chỉ số thời gian từ vw_f10_timing_24h
```

Mục tiêu chính là tạo một bảng phân tích đầy đủ cho 10 level đầu, trong đó mỗi bước có cả:

* số người tới level;
* tỷ lệ chuyển đổi;
* tỷ lệ rơi;
* thời gian đi từ Level 1 đến bước hiện tại;
* thời gian đi từ bước trước đến bước hiện tại;
* thời gian trong level;
* độ phủ dữ liệu `End_level`;
* độ phủ dữ liệu `duration_sec`.

---

## 2. Vai trò trong hệ thống

`vw_f10_funnel_timing_24h` thuộc nhóm:

```text id="d38bzw"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là lớp kết hợp, không tự tính lại toàn bộ logic phễu và thời gian.

Luồng phụ thuộc chính:

```text id="dq4t90"
vw_f10_funnel_24h
vw_f10_timing_24h
  └── vw_f10_funnel_timing_24h
```

View này phù hợp để dùng trực tiếp cho dashboard hoặc báo cáo vì nó gom các chỉ số quan trọng nhất của 10 level đầu vào cùng một bảng.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu người chơi tới từng level trong 10 level đầu?
2. Tỷ lệ rơi từ level trước sang level hiện tại là bao nhiêu?
3. Tỷ lệ chuyển đổi cộng dồn từ Level 1 đến từng level là bao nhiêu?
4. Người chơi mất bao lâu để đi từ Level 1 đến từng level?
5. Người chơi mất bao lâu để đi từ level trước đến level hiện tại?
6. Level nào vừa có tỷ lệ rơi cao vừa có thời gian đi tới cao?
7. Level nào có thời gian chơi trong level cao?
8. Level nào có tỷ lệ có `End_level` thấp?
9. Level nào có tỷ lệ có `duration_sec` thấp?
10. Level nào nên được ưu tiên kiểm tra về thiết kế hoặc tracking?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_f10_funnel_timing_24h` đại diện cho một bước trong 10 level đầu của một phiên bản game.

Có thể hiểu đơn giản:

```text id="0gpw14"
Một dòng = một app_version + một progression_step + một level_name
```

View này giữ cùng độ chi tiết với `vw_f10_funnel_24h` và `vw_f10_timing_24h`.

---

## 5. Loại đối tượng

```text id="ptnff6"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính từ hai view nguồn mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào hai view:

| Nguồn               | Vai trò                                                      |
| ------------------- | ------------------------------------------------------------ |
| `vw_f10_funnel_24h` | Cung cấp chỉ số phễu, số người, tỷ lệ chuyển đổi, tỷ lệ rơi. |
| `vw_f10_timing_24h` | Cung cấp chỉ số thời gian và độ phủ dữ liệu thời lượng.      |

View lấy `vw_f10_funnel_24h` làm bảng chính, sau đó `LEFT JOIN` sang `vw_f10_timing_24h`.

Điều này có nghĩa là nếu một bước có trong phễu nhưng không có dữ liệu timing tương ứng, dòng phễu vẫn được giữ lại.

---

## 7. Điều kiện nối dữ liệu

View nối hai nguồn theo ba trường:

```text id="7nq7t0"
app_version
progression_step
level_name
```

Logic nối:

```sql id="54hda2"
f.app_version = t.app_version
AND f.progression_step = t.progression_step
AND f.level_name = t.level_name
```

Điều này đảm bảo chỉ số phễu và chỉ số thời gian được ghép đúng theo cùng phiên bản game, cùng bước progression và cùng tên level.

---

## 8. Danh sách cột

| Cột                                              | Kiểu dữ liệu | Ý nghĩa                                                                            |
| ------------------------------------------------ | ------------ | ---------------------------------------------------------------------------------- |
| `app_version`                                    | `STRING`     | Phiên bản game.                                                                    |
| `progression_step`                               | `INT64`      | Bước trong 10 level đầu.                                                           |
| `progression_slot_id`                            | `STRING`     | Mã vị trí của bước trong tiến trình.                                               |
| `level_name`                                     | `STRING`     | Tên level.                                                                         |
| `level_design_id`                                | `STRING`     | Mã thiết kế level.                                                                 |
| `level_type`                                     | `STRING`     | Loại level.                                                                        |
| `previous_progression_step`                      | `INT64`      | Bước trước đó trong tiến trình.                                                    |
| `next_progression_step`                          | `INT64`      | Bước tiếp theo trong tiến trình.                                                   |
| `mapping_confidence_status`                      | `STRING`     | Mức độ tin cậy của mapping.                                                        |
| `mapping_source`                                 | `STRING`     | Nguồn tạo mapping.                                                                 |
| `level1_cohort_user_count`                       | `INT64`      | Tổng số người chơi có bắt đầu Level 1.                                             |
| `matured_level1_cohort_user_count`               | `INT64`      | Số người chơi đã đủ 24 giờ kể từ Level 1.                                          |
| `not_matured_level1_cohort_user_count`           | `INT64`      | Số người chơi chưa đủ 24 giờ kể từ Level 1.                                        |
| `user_count`                                     | `INT64`      | Số người chơi đã tới bước hiện tại theo đúng thứ tự.                               |
| `previous_step_user_count`                       | `INT64`      | Số người chơi ở bước trước.                                                        |
| `drop_from_previous_step_user_count`             | `INT64`      | Số người rơi từ bước trước sang bước hiện tại.                                     |
| `conversion_rate_from_previous_step`             | `FLOAT64`    | Tỷ lệ chuyển đổi từ bước trước sang bước hiện tại.                                 |
| `drop_rate_from_previous_step`                   | `FLOAT64`    | Tỷ lệ rơi từ bước trước sang bước hiện tại.                                        |
| `funnel_start_user_count`                        | `INT64`      | Số người chơi ở bước đầu phễu.                                                     |
| `cumulative_drop_user_count`                     | `INT64`      | Số người rơi cộng dồn từ bước đầu đến bước hiện tại.                               |
| `cumulative_conversion_rate`                     | `FLOAT64`    | Tỷ lệ chuyển đổi cộng dồn từ Level 1 đến bước hiện tại.                            |
| `cumulative_drop_rate`                           | `FLOAT64`    | Tỷ lệ rơi cộng dồn từ Level 1 đến bước hiện tại.                                   |
| `level1_to_step_timing_sample_user_count`        | `INT64`      | Số người có đủ dữ liệu để tính thời gian từ Level 1 đến bước hiện tại.             |
| `avg_seconds_from_level1_to_step`                | `FLOAT64`    | Thời gian trung bình từ Level 1 đến bước hiện tại.                                 |
| `median_seconds_from_level1_to_step`             | `INT64`      | Trung vị thời gian từ Level 1 đến bước hiện tại.                                   |
| `p90_seconds_from_level1_to_step`                | `INT64`      | Mốc 90% thời gian từ Level 1 đến bước hiện tại.                                    |
| `previous_step_to_step_timing_sample_user_count` | `INT64`      | Số người có đủ dữ liệu để tính thời gian từ bước trước đến bước hiện tại.          |
| `avg_seconds_from_previous_step_to_step`         | `FLOAT64`    | Thời gian trung bình từ bước trước đến bước hiện tại.                              |
| `median_seconds_from_previous_step_to_step`      | `INT64`      | Trung vị thời gian từ bước trước đến bước hiện tại.                                |
| `p90_seconds_from_previous_step_to_step`         | `INT64`      | Mốc 90% thời gian từ bước trước đến bước hiện tại.                                 |
| `level_end_event_sample_user_count`              | `INT64`      | Số người có `End_level` phù hợp trong vòng 6 giờ sau khi bắt đầu level.            |
| `level_end_event_coverage_rate`                  | `FLOAT64`    | Tỷ lệ người có `End_level` phù hợp trên tổng số người ở bước đó.                   |
| `level_end_success_user_count`                   | `INT64`      | Số người có `End_level` đầu tiên với `success = TRUE`.                             |
| `avg_wallclock_seconds_spent_inside_level`       | `FLOAT64`    | Thời gian trung bình trong level tính theo chênh lệch timestamp giữa start và end. |
| `median_wallclock_seconds_spent_inside_level`    | `INT64`      | Trung vị thời gian trong level theo timestamp.                                     |
| `p90_wallclock_seconds_spent_inside_level`       | `INT64`      | Mốc 90% thời gian trong level theo timestamp.                                      |
| `reported_duration_sample_user_count`            | `INT64`      | Số người có `duration_sec` hợp lệ.                                                 |
| `reported_duration_coverage_rate`                | `FLOAT64`    | Tỷ lệ người có `duration_sec` hợp lệ trên tổng số người ở bước đó.                 |
| `avg_reported_duration_sec_inside_level`         | `FLOAT64`    | Thời lượng trung bình trong level theo `duration_sec`.                             |
| `median_reported_duration_sec_inside_level`      | `FLOAT64`    | Trung vị thời lượng trong level theo `duration_sec`.                               |
| `p90_reported_duration_sec_inside_level`         | `FLOAT64`    | Mốc 90% thời lượng trong level theo `duration_sec`.                                |
| `first_level1_start_time_utc`                    | `TIMESTAMP`  | Thời điểm Level 1 sớm nhất trong phiên bản game.                                   |
| `last_level1_start_time_utc`                     | `TIMESTAMP`  | Thời điểm Level 1 muộn nhất trong phiên bản game.                                  |
| `funnel_view_query_time_utc`                     | `TIMESTAMP`  | Thời điểm tính view phễu.                                                          |
| `timing_view_query_time_utc`                     | `TIMESTAMP`  | Thời điểm tính view timing.                                                        |

---

## 9. Nhóm chỉ số chính

### 9.1 Nhóm mapping

Các cột:

```text id="gc229s"
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

Nhóm này mô tả level trong tiến trình 10 level đầu.

---

### 9.2 Nhóm cohort

Các cột:

```text id="3vwv3k"
level1_cohort_user_count
matured_level1_cohort_user_count
not_matured_level1_cohort_user_count
```

Trong tài liệu này, “cohort” nghĩa là nhóm người chơi cùng thỏa một điều kiện. Ở đây là nhóm người chơi đã bắt đầu Level 1.

---

### 9.3 Nhóm phễu

Các cột:

```text id="6jpf1q"
user_count
previous_step_user_count
drop_from_previous_step_user_count
conversion_rate_from_previous_step
drop_rate_from_previous_step
funnel_start_user_count
cumulative_drop_user_count
cumulative_conversion_rate
cumulative_drop_rate
```

Nhóm này trả lời câu hỏi:

```text id="ayrgvc"
Có bao nhiêu người chơi tới từng bước và rơi ở đâu?
```

---

### 9.4 Nhóm thời gian đi tới level

Các cột:

```text id="d8j10n"
level1_to_step_timing_sample_user_count
avg_seconds_from_level1_to_step
median_seconds_from_level1_to_step
p90_seconds_from_level1_to_step
previous_step_to_step_timing_sample_user_count
avg_seconds_from_previous_step_to_step
median_seconds_from_previous_step_to_step
p90_seconds_from_previous_step_to_step
```

Nhóm này trả lời câu hỏi:

```text id="tsj5ta"
Người chơi mất bao lâu để đi tới bước hiện tại?
```

---

### 9.5 Nhóm thời gian trong level

Các cột:

```text id="g9yizn"
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

Nhóm này trả lời câu hỏi:

```text id="k5f7x1"
Người chơi mất bao lâu ở bên trong level và dữ liệu End_level có đủ không?
```

---

## 10. Kiểm tra chất lượng dữ liệu khuyến nghị

### 10.1 Kiểm tra số dòng có khớp với `vw_f10_funnel_24h`

```sql id="82lpqs"
SELECT
  'vw_f10_funnel_24h' AS source_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

UNION ALL

SELECT
  'vw_f10_funnel_timing_24h' AS source_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`;
```

Kỳ vọng:

```text id="mept9q"
Hai row_count bằng nhau
```

Vì `vw_f10_funnel_timing_24h` lấy `vw_f10_funnel_24h` làm bảng chính.

---

### 10.2 Kiểm tra dòng bị thiếu dữ liệu timing

```sql id="7g45mo"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,

  level1_to_step_timing_sample_user_count,
  previous_step_to_step_timing_sample_user_count,
  level_end_event_sample_user_count,
  reported_duration_sample_user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

WHERE
  level1_to_step_timing_sample_user_count IS NULL
  AND user_count > 0

ORDER BY
  app_version,
  progression_step;
```

Nếu có kết quả, cần kiểm tra việc nối giữa `vw_f10_funnel_24h` và `vw_f10_timing_24h`.

---

### 10.3 Kiểm tra thời điểm truy vấn giữa hai view nguồn

```sql id="mocap9"
SELECT
  app_version,
  progression_step,
  level_name,
  funnel_view_query_time_utc,
  timing_view_query_time_utc,

  TIMESTAMP_DIFF(
    timing_view_query_time_utc,
    funnel_view_query_time_utc,
    SECOND
  ) AS query_time_gap_sec

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

ORDER BY
  ABS(query_time_gap_sec) DESC;
```

Thông thường, khi cùng một truy vấn gọi view này, hai thời điểm nên rất gần nhau.

---

### 10.4 Kiểm tra sample timing không vượt quá `user_count`

```sql id="ctq9hs"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

WHERE
  level1_to_step_timing_sample_user_count > user_count
  OR previous_step_to_step_timing_sample_user_count > user_count
  OR level_end_event_sample_user_count > user_count
  OR reported_duration_sample_user_count > user_count

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="ga114g"
Không trả ra dòng nào
```

Nếu có kết quả, có thể có lỗi trong logic tổng hợp hoặc join.

---

### 10.5 Kiểm tra tỷ lệ trong khoảng 0 đến 1

```sql id="e2g9so"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

WHERE
  conversion_rate_from_previous_step < 0
  OR conversion_rate_from_previous_step > 1
  OR drop_rate_from_previous_step < 0
  OR drop_rate_from_previous_step > 1
  OR cumulative_conversion_rate < 0
  OR cumulative_conversion_rate > 1
  OR cumulative_drop_rate < 0
  OR cumulative_drop_rate > 1
  OR level_end_event_coverage_rate < 0
  OR level_end_event_coverage_rate > 1
  OR reported_duration_coverage_rate < 0
  OR reported_duration_coverage_rate > 1

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="ntn6qe"
Không trả ra dòng nào
```

---

## 11. Cách sử dụng khuyến nghị

### 11.1 Xem bảng tổng hợp phễu và thời gian

```sql id="p6t08e"
SELECT
  app_version,
  progression_step,
  level_name,

  user_count,
  previous_step_user_count,
  drop_rate_from_previous_step,
  cumulative_conversion_rate,

  avg_seconds_from_previous_step_to_step,
  p90_seconds_from_previous_step_to_step,

  level_end_event_coverage_rate,
  avg_wallclock_seconds_spent_inside_level,
  reported_duration_coverage_rate,
  avg_reported_duration_sec_inside_level

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

---

### 11.2 Tìm level vừa rơi cao vừa chậm

```sql id="7hq93a"
SELECT
  app_version,
  progression_step,
  level_name,

  previous_step_user_count,
  user_count,
  drop_rate_from_previous_step,

  p90_seconds_from_previous_step_to_step,

  CASE
    WHEN previous_step_user_count < 30 THEN 'Chưa đủ mẫu'
    WHEN drop_rate_from_previous_step >= 0.4
      AND p90_seconds_from_previous_step_to_step >= 300
      THEN 'Ưu tiên kiểm tra cao'
    WHEN drop_rate_from_previous_step >= 0.25
      OR p90_seconds_from_previous_step_to_step >= 300
      THEN 'Cần theo dõi'
    ELSE 'Tạm ổn'
  END AS review_priority

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

WHERE
  progression_step > 1

ORDER BY
  review_priority,
  drop_rate_from_previous_step DESC,
  p90_seconds_from_previous_step_to_step DESC;
```

Ngưỡng trong truy vấn này chỉ là ví dụ. Team nên điều chỉnh theo chuẩn nội bộ.

---

### 11.3 Kiểm tra level có dữ liệu thời lượng yếu

```sql id="8g3odk"
SELECT
  app_version,
  progression_step,
  level_name,

  user_count,
  level_end_event_coverage_rate,
  reported_duration_coverage_rate,

  CASE
    WHEN user_count < 30 THEN 'Chưa đủ mẫu'
    WHEN level_end_event_coverage_rate < 0.7 THEN 'Thiếu End_level nhiều'
    WHEN reported_duration_coverage_rate < 0.7 THEN 'Thiếu duration_sec nhiều'
    ELSE 'Tạm ổn'
  END AS timing_data_quality_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

---

## 12. Lưu ý khi sử dụng

### 12.1 View này là view kết hợp, không phải nguồn gốc logic

Nếu cần điều tra sâu về phễu, xem:

```text id="lxzrjq"
vw_f10_funnel_24h
```

Nếu cần điều tra sâu về timing, xem:

```text id="l7nqxf"
vw_f10_timing_24h
```

`vw_f10_funnel_timing_24h` chỉ ghép hai nguồn trên.

---

### 12.2 Dòng phễu được giữ lại kể cả khi timing thiếu

View dùng `LEFT JOIN` từ `vw_f10_funnel_24h` sang `vw_f10_timing_24h`.

Vì vậy, dòng từ phễu sẽ vẫn tồn tại nếu timing bị thiếu.

Đây là hành vi đúng cho báo cáo tổng hợp, vì không nên mất dòng phễu chỉ vì thiếu dữ liệu thời gian.

---

### 12.3 Cần phân biệt tỷ lệ rơi và thời gian

Một level có thể:

* tỷ lệ rơi cao nhưng thời gian không cao;
* thời gian cao nhưng tỷ lệ rơi thấp;
* cả hai đều cao.

Cần đọc kết hợp nhiều chỉ số thay vì chỉ nhìn một cột.

---

### 12.4 Không nên kết luận từ timing nếu sample nhỏ

Nếu các cột sample thấp, ví dụ:

```text id="y9yl6v"
level1_to_step_timing_sample_user_count
previous_step_to_step_timing_sample_user_count
level_end_event_sample_user_count
reported_duration_sample_user_count
```

thì các chỉ số trung bình, trung vị và p90 có thể không ổn định.

---

## 13. Rủi ro nếu view sai logic

| Lỗi logic                                          | Ảnh hưởng                                                 |
| -------------------------------------------------- | --------------------------------------------------------- |
| Join sai giữa funnel và timing                     | Chỉ số phễu và thời gian bị gắn nhầm level.               |
| Không giữ dòng phễu khi timing thiếu               | Dashboard bị thiếu level.                                 |
| Không kiểm tra sample timing                       | Có thể kết luận sai từ dữ liệu quá ít.                    |
| Không phân biệt tỷ lệ rơi và thời gian             | Dễ ưu tiên sai level cần sửa.                             |
| Mapping level sai ở view nguồn                     | Toàn bộ kết quả kết hợp bị sai.                           |
| Timing view và funnel view không cùng logic cohort | Có thể tạo chênh lệch giữa `user_count` và sample timing. |

---

## 14. Mức độ quan trọng

```text id="zsh5oi"
Mức độ quan trọng: Cao
```

Lý do:

* view này là bảng tổng hợp thuận tiện nhất cho dashboard 10 level đầu;
* view này kết hợp cả số người, tỷ lệ rơi và thời gian;
* view này là đầu vào quan trọng cho chẩn đoán thiết kế level;
* view này giúp ưu tiên level cần kiểm tra nhanh hơn so với đọc riêng từng view.

---

## 15. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="srlepy"
sql/ddl/current_live_analysis/vw_f10_funnel_timing_24h.sql
```

Cấu trúc logic chính của view:

```sql id="njqozm"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_timing_24h`
AS
SELECT
  f.app_version,
  f.progression_step,
  f.progression_slot_id,
  f.level_name,
  f.level_design_id,
  f.level_type,
  f.previous_progression_step,
  f.next_progression_step,
  f.mapping_confidence_status,
  f.mapping_source,

  -- cohort
  f.level1_cohort_user_count,
  f.matured_level1_cohort_user_count,
  f.not_matured_level1_cohort_user_count,

  -- funnel
  f.user_count,
  f.previous_step_user_count,
  f.drop_from_previous_step_user_count,
  f.conversion_rate_from_previous_step,
  f.drop_rate_from_previous_step,
  f.funnel_start_user_count,
  f.cumulative_drop_user_count,
  f.cumulative_conversion_rate,
  f.cumulative_drop_rate,

  -- timing
  t.level1_to_step_timing_sample_user_count,
  t.avg_seconds_from_level1_to_step,
  t.median_seconds_from_level1_to_step,
  t.p90_seconds_from_level1_to_step,

  t.previous_step_to_step_timing_sample_user_count,
  t.avg_seconds_from_previous_step_to_step,
  t.median_seconds_from_previous_step_to_step,
  t.p90_seconds_from_previous_step_to_step,

  t.level_end_event_sample_user_count,
  t.level_end_event_coverage_rate,
  t.level_end_success_user_count,
  t.avg_wallclock_seconds_spent_inside_level,
  t.median_wallclock_seconds_spent_inside_level,
  t.p90_wallclock_seconds_spent_inside_level,

  t.reported_duration_sample_user_count,
  t.reported_duration_coverage_rate,
  t.avg_reported_duration_sec_inside_level,
  t.median_reported_duration_sec_inside_level,
  t.p90_reported_duration_sec_inside_level,

  f.first_level1_start_time_utc,
  f.last_level1_start_time_utc,
  f.mart_updated_at AS funnel_view_query_time_utc,
  t.timing_view_query_time_utc

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h` AS f

LEFT JOIN
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h` AS t
ON
  f.app_version = t.app_version
  AND f.progression_step = t.progression_step
  AND f.level_name = t.level_name;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[vw_f10_funnel_24h]]
	- [[vw_f10_timing_24h]]

- Downstream: [[vw_f10_design_diag_24h]]

