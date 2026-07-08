ĐẶC TẢ VIEW
===
`vw_funnel_first_open_to_level10`
---

# 1. Thông tin tổng quan

## 1.1. Tên View

```
project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_funnel_first_open_to_level10
```

## 1.2. Mục tiêu của view

View này dùng để phân tích funnel từ lần mở app đầu tiên đến Level 10:

1. first_open
2. Open_first
3. Level 1
4. Level 2
5. Level 3
6. Level 4
7. Level 5
8. Level 6
9. Level 7
10. Level 8
11. Level 9
12. Level 10

View được thiết kế để làm nguồn dữ liệu chính thức cho các phân tích:

- onboarding funnel
- early progression funnel
- drop-of theo từng step
- conversion theo từng step
- conversion theo first_open
- conversion từ Level 1
- level completion
- win/fail/no-end
- difficulty/friction
- timing giữa các step
- moves used

## 1.3. Grain của view

Mỗi row trong view đại diện cho: 1 app_version & 1 funnel step

## 1.4. Cohort gốc

Cohort gốc là: user có event first_open

User được đưa vào cohort nếu:

```
first_open_time_utc <= CURRENT_TIMESTAMP() - 24 hours
```

> Tức là, chỉ lấy user đã đủ ít nhất 24 giờ quan sát sau `first_open`.

## 1.5. App version assignment

`app_version` của user được gán theo: `app_info.version` tại `first_open` đầu tiên của user.

Nếu user có nhiều event về sau với version khác, view vẫn gán user vào `app_version` tại `first_open`.

## 1.6. Observaton window

Mỗi user được quan sát trong windows:

```
[first_open_time_utc, first_open_time_utc + 24hours]
```

Các event ngoài windows 24h không được tính vào funnel này.

## 1.7. User identifier

View sử dụng `user_pseudo_id` làm định danh user.

---

# 2. Nguồn dữ liệu

## 2.1. Raw GA4 source

- Nguồn chính:

```
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

- View chỉ đọc các bảng có suffix dạng ngày:

```SQL
_TABLE_SUFFIX = YYYYMMĐ
```

- Điều kiện:

```SQL
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

> Điều này loại bỏ các bảng không phải daily table, ví dụ bảng intraday nếu có.

## 2.2. Mapping source

- Nguồn mapping Level 1 → Level 10

```
project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map
```

- Dùng các field:
  - `app_version`
  - `progression_step`
  - `progression_slot_id`
  - `level_name`
  - `level_design_id`
  - `level_type`

- Chỉ dùng:
```
progression_step BETWEEN 1 AND 10
```

## 2.3. Event chính sử dụng

| Event | Vai trò |
|:---|:---|
| `first_open` | Tạo cohort gốc |
| `Open_first` | Step onboarding sau `first_open` |
| `Start_level` | Raw level start |
| `Tutorial_start` | Dùng để tạo pseudo level start trong một số case |
| `End_level` | Xác định user end level, win/fail |
| `Move` | Tính move count trong level window |

---

# 3. Logic tạo cohort

## 3.1. Chọn `first_open` đầu tiên

Nếu một user có nhiều event `first_open`, chỉ lấy event đầu tiên:

```SQL
ROW_NUMBER() OVER ( PARTITION BY user_pseudo_id ORDER BY event_time_utc ) = 1
```

- Output:
  - `user_pseudo_id`
  - `app_version`
  - `first_open_time_utc`

 ## 3.2. Maturity 24h

 Chỉ lấy user đã đủ 24 giờ quan sát:

 ```SQL
first_open_time_utc <= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR) 
```

## 3.3. Follow-up Window

Với mỗi user:

```
followup_end_time_utc = first_open_time_utc + 24 hours
```

---

# 4. Logic `Open_first`

`Open_first` được tính là event `Open_first` đầu tiên của user trong window:

```
first_open_time_utc <= Open_first time <= followup_end_time_utc
```

Đồng thời, event phải cùng `app_version` với `app_version` tại `first_open`.

- Công thức:
```
MIN(event_time_utc)
```

Với:

```
event_name = 'Open_first'
```

---

# 5. Logic effective level start

View sử dụng `effective_level_starts`, gồm 2 nhóm:

1. Raw `Start_level`
2. Pseudo `Start_level`

## 5.1. Raw `Start_level`

- `event_name = 'Start_level'`
- `event_params = 'level_name'`

## 5.2. Pseudo `Start_level`

Một só level trong version `0.3.61` không có event `Start_level` nhưng có event `Tutorial_start` có thể dùng để thay thế.

| Level ID | `level_name` | `name` |
|:---|:---|:---|
| N001 | 1 | Match |
| N010 | 10 | Nectar |

## 5.3. Điều kiện để event `Start_level` của một level hợp lệ

Một event `Start_level` của một level được xem là candidate nếu:

- `user_pseudo_id` khớp
- `app_version` khớp `app_version` tại `first_open`
- `level_name` khớp mapping
- `Open_first` đã tồn tại
- `level_start_time_utc` >= `open_first_time_utc`
- `level_start_time_utc` <= `followup_end_time_utc`

---

# 6. Logic sequential progression

View không chỉ dếm user từng thấy level. View yếu cầu user đi theo thứ tự sequential.

Với mỗi user và mỗi mapped level: `raw_level_start_time_utc` = first start time của level đó trong 24h window.

- Một level được xem là valid reach nếu:

  - `raw_level_start_time_utc` IS NOT NULL

  - Tất cả previous progression steps đều có start

  - Không có order violation.

- Order violation xảy ra nếu:

current level start time < max start time của các previous steps.

Nếu một previous step bị thiếu hoặc sai thứ tự, các step sau không được tính là reached hợp lệ trong funnel.

---

# 7. Logic level window

Với mỗi user-level hợp lệ: `level_start_time_utc` = thời điểm start level hiện tại.

- `level_window_end_time_utc` được xác định như sau:

  - Nếu có next level start trong 24h: `level_window_end_time_utc` = `next_level_start_time_utc`
 
  - Nếu không có next level start: `level_window_end_time_utc` = `followup_end_time_utc`

> Tức là mọi event `End_level`, `Move` của một level chỉ được tính trong windows:

```
[
  level_start_time_utc,
  level_window_end_time_utc
]
```

---

# 8. Logic `End_level`, Win, Fail

## 8.1. `End_level` event

- `event_name = End_level`

- Điều kiện match với level windows:

  - same: `user_pseudo_id`, `app_version`, `level_name`
 
  - `end_time_utc` >= `level_start_time_utc`
 
  - `end_time_utc` <= `level_window_end_time_utc`
 
## 8.2. Win/Fail param

- Event param = `success`

- Mapping:

  - `success = 1` → Win
 
  - `success = 0` → Fail
 
- Logic

```SQL
CASE
  WHEN success_value = '1'  THEN TRUE
  WHEN success_value = '0' THEN FALSE
  ELSE NULL
END AS is_win
```

## 8.3. User-level outcome

Với mỗi user-level:

- `has_end`: có ít nhất 1 `End_level` event

- `has_win`: có ít nhất 1 `End_level` event với `success = 1`

- `has_fail`: có ít nhất 1 `End_level` event với `success = 0`

## 8.4. Outcome groups

Mỗi user-level có thể thuộc các nhóm:

| Nhóm | Điều kiện |
|:---|:---|
| `win_only_user` | `has_win = TRUE` AND `has_fail = FALSE` |
| `fail_only_user` | `has_win = FALSE` AND `has_fail = TRUE` |
| `win_after_fail_or_mixed_user` | `has_win = TRUE` AND `has_fail = TRUE` |
| `start_without_end_user` | `has_end = FALSE` |

## 8.5. Quan hệ giữa các nhóm

Với level rows:

```
end_level_user_count =
  win_only_user_count +
  fail_only_user_count +
  win_after_fail_or_mixed_user_count
```

```
user_count =
  end_level_user_count +
  start_without_end_user_count
```

```
win_user_count =
  end_level_user_count +
  start_without_end_user_count
```

```
win_user_count =
  user_with_any_win_count =
  win_only_user_count +
  win_after_fail_or_mixed_user_count
```

```
user_with_any_fail_count =
  fail_only_user_count +
  win_after_fail_or_mixed_user_count
```

```
end_no_win_user_count =
  fail_only_user_count
```

---

# 9. Logic Move

## 9.1. Move event

- event_name = 'Move'

Một Move được tính trong level nếu same:

- `user_pseudo_id`
- `app_version`
- `move_time_utc` >= `level_start_time_utc`
- `move_time_utc` <= `level_window_end_time_utc`

## 9.2. Move used to win

Với user-level có Win:

```
move_count_to_win = Số Move từ level_start_time_utc đến first_win_time_utc
```

## 9.3. Move before no win

Với user-level không có Win:

```
move_count_before_no_win = Số Move từ level_start_time_utc đến level_windows_end_time_utc
```

---

# 10. Làm sạch và chuẩn hóa dữ liệu raw

## 10.1. Chuẩn hóa event timestamp

- Raw GA4 có: `event_timestamp`

- View chuyển sang UTC timestamp:

```SQL
TIMESTAMP_MICROS(event_timestamp) AS event_time_utc
```

## 10.2. Extract `level_name`

- `level_name` được lấy từ `event_params`:

- `event_params.key` = `level_name`

- View dùng `COALESCE` để lấy value theo các kiểu:

  - `string_value`
  - `int_value`
  - `float_value`
  - `double_value`
 
- Ouput cuối là: `string`

## 10.3. Extract tutorial name

- Tutorial name lấy từ: `event_params.key` = 'name'

- Dùng cho pseudo start:
  - Match: N001
  - Nectar: N010

## 10.4. Extract success

- Success lấy từ: `event_params.key` = 'success'

- Output dạng string: '0' hoặc '1'

- Sau đó map sang boolean `is_win`

- ## 10.5. Clean `duration_sec`

- Raw `duration_sec` có thể chứa dấu phẩy thập phân.

- Ví dụ: "12,34"

- View xử lý:

```SQL
REPLACE(value, ',', '.')
```

- Sau đó:

```SQL
SAFE_CAST(... AS FLOAT64)
```

Nếu không cast được, output là `NULL`.

## 10.6. Clean `attempt_no`

- `attempt_no` được lấy từ `event_params`, sau đó:

```SQL
SAFE_CAST(... AS FLOAT64)
```

- Nếu không cast được, output là `NULL`.

## 10.7. Loại intraday / non-daily tables

- View chỉ đọc bảng có suffix đúng format ngày:

```
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

- Do đó các bảng không phải daily export sẽ không được tính.

---

# 11. Đặc tả từng field trong output

## 11.1. Dimension fields

| Field | Type | Đặc tả |
|:---|:---|:---| 
| `app_version` | STRING | - App version tại `first_open` đầu tiên của user.<br>- Cách lấy:<br>`app-info.version` từ event `first_open` đầu tiên của user. |
| `funnel_step_order` | INT64 | Thứ tự của strp trong funnel. |
| `funnel_step_type` | STRING | Phân loại step là `onboarding` hay `level`. |
| `funnel_step_name` | STRING | Tên hiển thị của funnel step<br>- Ví dụ: 01_first_open<br>- Công thức cho level rows:<br>CONCAT( LPAD(CAST(progression_step + 2 AS STRING), 2, '0'), '_Level_', LPAD(CAST(progression_step AS STRING), 2, '0'), '_', level_name ) |
| `progression_step` | INT64 | Thứ tự level |
| `progression_slot_id` | STRING | Slot là định danh progression step trong mapping table.<br>- Nguồn: dim_f10_level_map.progression_slot_id |
| `level_name` | STRING | Tên level được map vào progression step.<br>- Nguồn: dim_f10_level_map.level_name |
| `level_design_id` | STRING | ID thiết kế level nếu có trong mapping table.<br>- Nguồn: dim_f10_level_map.level_design-id |
| `level_type` | STRING | Loại level nếu có trong mapping table><br>- Nguồn: dim_f10_level_map.level_type |

---

## 11.2. Funnel count fields

| Field | Type | Đặc tả |
|:---|:---|:---|
| `user_count` | INT64 | - Với `first_open`: COUNT(DISTINCT user_pseudo_id trong cohort first_open)<br>- Với `Open_first`: COUNT(DISTINCT user_pseudo_id có Open_first trong 24h window)<br>- Với level_rows: COUNT(DISTINCT user_pseudo_id có valid sequential level_start_time_utc cho level đó) |
| `previous_user_count` | INT64 | - Số user ở step trước đó. |
| `drop_from_previous_step_user_count` | INT64 | - Số user rơi từ step trước sang step hiện tại.<br>- Công thức: `previous_user_count` - `user_count` |
| `drop_rate_from_previous_step_pct` | FLOAT64 | - Tỷ lệ user rơi từ step trước sang step hiện tại<br>- Công thức: `drop_from_previous_step_user_count` / `previous_user_cont` * 100<br>- SQL: ROUND( SAFE_DIVIDE( previous_user_count - user_count, previous_user_count ) * 100, 2 ) |
| `conversion_rate_from_previous_step_pct` | FLOAT64 | Tỷ lệ user chuyển tiếp từ step trước sang step hiện tại.<br>- Công thức: `user_count` / `previous_user_count` * 100<brr>- SQL: ROUND( SAFE_DIVIDE(user_count, previous_user_count) * 100, 2 ) |
| `drop_from_level1_user_count` | INT64 | - Số user rơi so với Level 1 cohort.<br>- Công thức: `level1_user_count` - `user_count` |
| `drop_rate_from_level1_pct` | FLOAT64 | - Tỷ lệ user rơi so với Level 1 cohort.<br>- Công thức: (`level1_user_count` - `user_count`) / `level1_user_count` × 100 |
| `cumulative_conversion_rate_from_level1_pct` | FLOAT64 | Conversion tích lũy từ Level 1 tới level hiện tại.<br>- Công thức: `user_count` / `level1_user_count` * 100 |
| `cumulative_drop_rate_from_level1_pct` | FLOAT64 | - Drop tích lũy từ Level 1 tới level hiện tại.<br>- Công thức: 100 - `cumulative_conversion_rate_from_level1_pct` hoặc<br>1 - `user_count` / `level1_user_count` |
| `conversion_rate_from_first_open_pct` | FLOAT64 | - Conversion từ `first_open` tới step hiện tại. <br>- Công thức: `user_count` / `first_open_user_count` * 100 |
| `drop_rate_from_first_open_pct` | FLOAT64 | Drop từ `first_open` tới step hiện tại.<br>- Công thức: 100 - `conversion_rate_from_first_open_pct` hoặc<br>1 - `user_count` / `first_open_user_count` |

---

## 11.3. End / Win / Fail fields
| Field | Type | Đặc tả |
|:---|:---|:---|
| `end_level_user_count` | INT64 | Số user có ít nhất 1 `End_level` trong level window.<br>- Công thức: COUNT(DISTINCT user_pseudo_id WHERE has_end = TRUE) |
| `win_user_count` | INT64 | Số user có ít nhất 1 `End_level` success = 1 trong level window.<br>- Công thức: COUNT(DISTINCT user_pseudo_id WHERE has_win = TRUE) |
| `end_no_win_user_count` | INT64 | Số user có `End_level` nhưng không có Win trong level window.<br>`end_no_win_user_count` = `fail_only_user_count` |
| `start_without_end_user_count` | INT64 | Số user start level nhưng không có `End_level` trong level window<br>- Công thức: COUNT(DISTINCT user_pseudo_id WHERE has_end = FALSE) |
| `user_witth_any_win_count` | INT64 | - Số user có ít nhất 1 win event trong level window.<br>- Công thức: COUNT(DISTINCT user_pseudo_id WHERE has_win = TRUE) |
| `user_with_any_fail_count` | INT64 | - Số user có ít nhất 1 fail event trong level window.<br>- Công thức: COUNT(DISTINCT user_pseudo_id WHERE has_fail = TRUE) |
| `win_only_user_count` | INT64 | Số user có win và không có fail trong level window. |
| `fail_only_user_count` | INT64 | Số user có fail và không có win trong level window. |
| `win_after_fail_or_mixed_user_count` | INT64 | Số user vừa có fail vừa có win trong cùng level window. |
| `win_end_event_count` | INT64 | Tổng số `End_level` events có success = 1 |
| `fail_end_event_count` | INT64 | - Tổng số `End_level` events có success = 0.<br>- Đây là event count, không phải user count. |
| `total_end_event_count` | INT64 | - Tổng số `End_level` events trong level window.<br>- Công thức: `win_end_event_count` + `fail_end_event_count` |

---

## 11.4. Rate fields for End / Win / Fail

| Field | Đặc tả |
|:---|:---|
| `win_user_rate_pct` | - Tỷ lệ user start level và có ít nhất 1 win.<br>- Công thức: `win_user_count` / `user_count` * 100 |
| `end_no_win_user_rate_pct` | - Tỷ lệ user start level, có `End_level`, nhưng không có win.<br>- Công thức: `end_no_win_user_count` / `user_count` * 100 |
| `end_level_converage_rate_pct` | - Tỷ lệ user start leveel và có `End_level`<br>- Công thức: `end_level_user_count` / `user_count` * 100 |
| `start_without_end_rate_pct` | - Tỷ lệ user start level và từng fail ít nhất 1 lần.<br>- Công thức: `user_with_any_fail_count` / `user_count` * 100 |
| `user_with_any_fail_rate_pct` | - Tỷ lệ user start level và từng fail ít nhất 1 lần.<br>- Công thức: `user_with_any_fail_count` / `user_count` * 100 |
| `faild_only_user_rate_pct` | Tỷ lệ user start level, fail, và không win.<br>- Công thức: `fail_only_user_count` / `user_count` * 100 |
| `win_after_fail_or_mixed_user_rate_pct` | - Tỷ lệ user start level, từng fail, nhưng cũng có win.<br>- Công thức: `win_after_fail_or_mixed_user_count` / `user_count` * 100 |

---

## 11.5. Timing fields

| Field | Type | Đặc tả |
|:---|:---|:---|
| `avg_seconds_from_previous_step` | FLOAT64 | - Thời gian trung bình từ step trước sang step hiện tại.<br>- Công thức: AVG(seconds_from_previous_step) |
| `median_seconds_from_previous_step` | INT64 | - Median thời gian từ step trước sang step hiện tại.<br>- Công thức: APPROX_QUANTILES(seconds_from_previous_step, 100)[OFFSET(50)] |
| `p90_seconds_from_previous_step` | INT64 | P90 thời gian từ step trước sang step hiện tại.<br>- Công thức: APPROX_QUANTILES(seconds_from_previous_step, 100)[OFFSET(90)] |
| `avg_seconds_from_first_open` | FLOAT64 | - Thời gian trung bình từ first_open tới step hiện tại.<br>- Công thức:AVG(current_step_time - first_open_time_utc) |
| `median_seconds_from_first_open` | INT64 | Median thời gian từ first_open tới step hiện tại. |
| `p90_seconds_from_first_open` | INT64 | P90 thời gian từ `first_open` tới step hiện tại. |

---

## 11.6. Move fields

| Field | Type | Đặc tả |
|:---|:---|:---|
| `avg_move_used_to_win` | FLOAT64 | Số move trung bnhf user dùng để win level (chỉ tnhs user có win).<br>- Công thức user-level: move_count_to_win = số Move từ `level_start_time_utc` tới `first_win_time_utc`<br>- Công thức aggregate: AVG(move_count_to_win) trên các user có has_win = TRUE |
| `avg_move_used_before_drop_no_win` | FLOAT64 | - Số move trung bình của user không win (chỉ tính user không có win)<br>- Công thức user-level: `move_count_before_no_win` = Số Move từ `level_start_time_utc` tới `level_window_end_time_utc`<br>- Công thức aggragate: AVG(move_count_before_no_win) trên các user has_win = FALSE |

---

## 11.7. Duration / attempt fields

| Field | Type | Đặc tả |
|:---|:---|:---|
| `avg_reported_duration_sec_on_win` | FLOAT64 | Thời gian `duration_sec` trung bình của user có win.<br>- Nguồn: `End_level.duration_sec`<br>- Raw `duration_sec` được làm sạch bằng: replace comma decimal → dot decimal SAFE_CAST to FLOAT64<br>- Lưu ý: Nếu `duration_sec` coverage thấp hoặc tracking lỗi, field này cần đọc thận trọng. |
| `avg_last_attempt_no` | FLOAT64 | Attempt number trung bình ở `End-level` cuối cùng trong level window.<br>- Nguồn: `End_level.attempt_no`<br>- Raw `attmpt_no` được cast bằng: SAFE_CASE(... AS INT64) |

---

## 11.8. Time boundary fields

| Field | Type | Đặc tả |
|:---|:---|:---|
| `first_time_utc` | TIMESTAMP | - Thời điểm đầu tiên của chính funnel step đó. |
| `last_time_utc` | TIMESTAMP | - Thời điểm cuối cùng của chính funnel đó. |
| `query_time_utc` | TIMESTAMP | - Thời điểm view được query.<br>- Công thức: CURRENT_TIMESTAMP() |

---

## 12. Output behavior theo row type

## 12.1. Onboarding rows

- `01_first_open`
- `02_Open_first`

## 12.2. Level rows

- `03_Level_01_...`
- ...
- `12_Level_10_..`

---

# 13. Validation rules đã pass

View được xem là hợp lệ nếu các điều kiện sau đúng:

- `row_count` = 12
- `min_step_order` = 1
- `max_step_order` = 12
- `distinct_step_order_count` = 12
- `first_open_row_count` = 1
- `open_first_row_count` = 1

Funnel count:

- `user_count` không tăng ngược
- `drop_from_previous_step_user_count` = `previous_user_count` - `user_count`
- `drop_rate_from_previous_step_pct` đúng công thức
- `conversion_rate_from_previous_step_pct` đúng công thức

Outcome

- `user_count` = `end_level_user_count` + `start_without_end_user_count`
- `win_user_count` = `user_with_any_win_count`
- `end_no_win_user_count` = `fail_only_user_count`
- `user_with_any_win_count` = `win_only_user_count` + `win_after_fail_or_mixed_user_count`
- `user_with_any_fail_count` = `fail_only_user_count` + `win_after_fail_or_mixed_user_count`
- `end_level_user_count` = `win_only_user_count` + `fail_only_user_count` + `win_after_fail_or_mixed_user_count`
- `total_end_event_count` = `win_end_event_count` + `fail_end_event_count`

Timing: `median_seconds_from_previous_step` <= `p90_seconds_from_previous_step`

---

# 14. Diễn giải metric quan trọng

## 14.1. Đọc progression drop

- `drop_from_previous_step_user_count`
- `drop_rate_from_previvous_step_pct`

## 14.2. Đọc Difficulty

- `user_with_any_fail_count`
- `user_with_any_fail_rate_ptc`
- `fail_end_event_count`
- `win_after_fail_or_mixed_user_count`
- `avg_move_used_to_win`

## 14.3. Đọc abandon / no-end

- `start_without_end_user_count`
- `start_without_end_rate_ptc`

## 14.4. Đọc timing

- `median_seconds_from_previous_step`
- `p90_seconds_from_previous_step`

Không nên chỉ nhìn average (mean) vì dễ bị outlier kéo cao.

---

# 15. Know limitations

## 15.1. View dùng `CURRENT_TIMESTAMP()`

Do view dùng meturity logic:

```
first_open_time_utc <= CURRENT_TIMESTAMP() - 24 hours
```

Nên kết quả có thể thay đổi theo thời gian khi cohort mới đủ maturity.

## 15.2. Không tính intraday table

View chỉ đọc daily tables có suffix `YYYYMMDD`.

Nếu cần realtime/intraday analysis, cần tạo viw hoặc query riêng.

## 15.3. App version cố định theo first_open

Nếu user `first_open` ở vesion A nhưng chơi tiếp trong version B trong 24h, view vẫn gán user vào version A và chỉ match event cùng `app_version`.

Điều này phù hợp với cohort analysis theo verson tại `first_open`, nhưng không phù hợp nếu muốn phân tích theo version tại từng level start.

## 15.4. Sequential funnel nghiêm ngặt

User chỉ được tính ở Level N nếu đã có đầy đủ các previous levels theo đúng thứ tự.

Điều này giúp funnel sạch, nhưng có thể loại các user có dữ liệu tracking thiếu ở step trước.

## 15.6. Win/Fail phụ param `success`

Nếu tracking thay đổi key hoặc value của success, view cần được cập nhật.

---

# Cập nhật 01: Bổ sung 2 metric mới

## 1. `first_end_is_win_user_count`

### 1.1. Ý nghĩa

Là số user có `End_level` đầu tiên trong level window là Win.

Nói cách khác, với mỗi user trong một level cụ thể, hệ thống nhìn vào event `End_level` đầu tiên của user đó trong `level_window`. Nếu event đầu tiên đó có: `success` = 1 thì user được tính vào `first_end_is_win_user_count`.

Metric này trả lời câu hỏi;

> Trong số user đã Start_level, bao nhiêu user có kết quả kết thúc level đầu tiên là Win?

### 1.2. Nguồn dữ liệu

- Nguồn raw event: `analytics_524104373.events_*`
- Event sử dụng: `End_level`
- Params:
  - `level_name`
  - `success`
    - `success` = 1 → Win
    - `success` = 0 → Fail
   
### 1.3. Điều kiện event được tính

Một `End_level` event chỉ được xét cho metric này nếu thỏa mãn same:
- `user_pseudo_id`
- `app_version`
- `level_name`
- `end_time_utc` >= `level_start_time_utc`
- `end_time_utc` <= `level_window_end_time_utc`

> Tức là event phải nằm trong đúng level window của user-level đó.

### 1.4. Logic user-level

Với mỗi user-level, lấy `End_level` đầu tiên theo thời gian:

```SQL
ARRAY_AGG(
  IF(end_events.end_time_utc IS NOT NULL, end_events.is_win, NULL)
  IGNORE NULLS
  ORDER BY end_events.end_time_utc
  LIMIT 1
)[SAFE_OFFSET(0)] AS first_end_is_win
```

Trong đó:

```SQL
CASE
  WHEN success_value = '1' THEN TRUE
  WHEN success_value = '0' THEN FALSE
  ELSE NULL
END AS is_win
```

Nếu `first_end_is_win` = TRUE, user được tính vào metric.

### 1.5. Công thức aggregate

```SQL
COUNT(DISTINCT IF(first_end_is_win = TRUE, user_pseudo_id, NULL)) AS first_end_is_win_user_count
```

### 1.6. Output

- Kiểu dữ liệu: `INT64`
- Chỉ áp dụng cho level rows

### 1.7. Khác gì so với `win_user_count`?

- `win_user_count` đếm: User có ít nhất 1 win trong level window.
- `first_end_is_win_user_coutn` đếm: User có `End_level` đầu tiên là win.

- Vì vậy: `first_end_is_win_user_count` <= `win_user_count`

- Trường hợp user fail trước rồi win sau, user này được tính vào:
  - `win_user_count`
  - `user_with_any_fail_count`
  - `win_after_fail_or_mixed_user_count`
 
- Nhưng không được tính vào: `first_end_is_win_user_count`.

### 1.9. Có phải First-Try-Win không?

Không hoàn toàn, `firt_end_is_win_user_count` phản ánh: `End_level` đầu tiên được ghi nhận là win. Nó không phụ thuộc vào `attempt_no`.

Nếu tracking `End_level` chính xác, metric này là proxy tốt cho "first-result win". Tuy nhiên, nếu `End_level` bị thiếu ở một số attempt trước đó, thì nó không thể chứng minh tuyệt đối user win ngay attempt đầu tiên.

---

## 2. `attempt_1_win_user_count`

### 2.1. Ý nghĩa

Là số user có ít nhất 1 `End_level` success = 1 với `attempt_no` = 1.

Metric này trả lời câu hỏi: Bao nhiêu user được ghi nhận là win ở attempt 1?

### 2.2. Nguồn dữ liệu

- Nguồn raw event: `analytics_524104373.event_*`
- Event sử dụng: `End_level`
- Params:
  - `level_name`
  - `success`
  - `attempt_no`
- Trong đó
  - success = 1 → Win
  - `attempt_no` = 1 → attempt đầu tiên theo tracking
 
### 2.3. Làm sạch / chuẩn hóa dữ liệu raw

- `success` được lấy từ `event_params` với key: success
  - Output được chuẩn hóa về string: '1', '0'
  - Sau đó map sang boolean:

```SQL
CASE
  WHEN success_value = '1' THEN TRUE
  WHEN success_value = '0' THEN FALSE
  ELSE NULL
END AS is_win
```

- `attempt_no` được lấy từ `event_params` với key: attempt_no.
- Sau đó được cast sang `INT64`

```SQL
SAFE_CAST(
  (
    SELECT COALESCE(
      ep.value.string_value,
      CAST(ep.value.int_value AS STRING),
      CAST(ep.value.float_value AS STRING),
      CAST(ep.value.double_value AS STRING)
    )
    FROM UNNEST(event_params) ep
    WHERE ep.key = 'attempt_no'
    LIMIT 1
  ) AS INT64
) AS attempt_no
```

- Nếu `attempt_no` không tồn tại hoặc không cast được sang `INT64`, output là: NULL.
- Các event có `attempt_no` = NULL sẽ không được tính vào `attempt_1_win_user_count`.

### 2.4. Điều kiện event được tính

Một event được tính là attempt-1 win nếu:
- `event_name` = End_level
- `success` = 1
- `attempt_no` = 1
- Same `user_pseudo-id`
- Same `level_name`
- Event nằm trong đúng level window

### 2.5. Logic user-level

Với mỗi user-level:

```SQL
COUNTIF(
  end_events.is_win = TRUE
  AND end_events.attempt_no = 1
) > 0 AS has_attempt_1_win
```

Nếu `has_attempt_1_win` = TRUE, user được tính vào metric.

### 2.6. Công thức aggregate

```
COUNT(DISTINCT IF(has_attempt_1_win = TRUE, user_pseudo_id, NULL)) AS attempt_1_win_user_count
```

### 2.7. Output

- Kiểu dữ liệu: `INT64`
- Chỉ áp dụng cho level rows

### 2.8. Khác gì với `first_end_is_win_user_count`?

- `first_end_is_win_user_count` dựa trên: Thứ tự `End_level` theo thời gian.
- `attempt_1_win_user_count` dựa trên: Giá trị `attempt_no` được tracking trong `End_level`.
- Vì vậy hai metric có thể khác nhau vì:
  - `attempt_no` bị thiếu
  - `attempt_no` không bắt đầu từ 1
  - `attempt_no` được tracking không đồng nhất
  - `End_level` đầu tiên là Win nhưng `attempt_no` không bằng 1.
 
---

## 3. Quan hệ logic với các field hiện có

Với level rows, các quan hệ logic cần đúng:

- `first_end_is_win_user_count` <= `win_user_count`
- `first_end_is_win_user_count` <= `end_level_user_count`
- `attempt_1_win_user_count` <= `win_user_count`
- `attempt_1_win_user_count` <= `end_level_user_count`

Trong đó:

`win_user_count` = `user_with_any_win_count`

và:

`end_level_user_count` = `win_only_user_count` +
                     `fail_only_user_count` +
                     `win_after_fail_or_mixed_user_count`

`first_end_is_win_user_count` có thể lớn hơn hoặc nhỏ hơn `attempt_1_win_user_count`, tùy chất lượng tracking `attempt_no`, nhưng thông thường:

`attempt_1_win_user_count` <= `first_end_is_win_user_count`

Tuy nhiên không nên xem đây là rule bắt buộc tuyệt đối nếu tracking `attempt_no` có lỗi hoặc event ordering bất thường.

---

## 4. Validation status

Hai metric mới đã được validate bằng 2 nhóm kiểm tra;

### 4.1. Internal validation

Kiểm tra:

- onboarding rows phải NULL
- level rows không NULL
- `first_end_is_win_user_count` <= `win_user_count`
- `first_end_is-win_user_count` <= `end_level_user_count`
- `attempt_1_win_user_count` <= `win_user_count`
- `attempt_1_win_user_count` <= `end_level_user_count`

Kết quả: PASS

### 4.2. Raw recompute validation

Hai metric được recompute trực tiếp từ raw GA4 theo cùng logic:

- `first_open` cohort
- 24h observation window
- `Open_first` requirement
- sequential Level 1 → Level 10
- level window
- `End_level` success
- `attempt_no`

Kết quả:

- Tất cả Level 1 → Level 10 đều PASS 
- `first_end_is_win_diff` = 0 
- `attempt_1_win_diff` = 0

Do đó, hai metric mới có thể được xem là hợp lệ để dùng trong dashboard và phân tích chính thức.

---

# Liên kết liên quan

## Framework và tài liệu tổng quan

- README tổng quan: [README](../../../README.md)
- Framework pipeline: [Framework Pipeline](../../pipeline/framework_pipeline.md)
- Dependency graph: [Mart Dependency Graph](../../graph/mart_dependency_graph.md)

## Nhóm tài liệu

- Nhóm object: `current_live_analysis`
- Vai trò trong mart: view phân tích funnel tổng hợp từ `first_open` đến Level 10 trong cửa sổ 24 giờ.
- Grain: 1 `app_version` × 1 `funnel_step`.

## Upstream

- GA4 raw source: `project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*`
- Mapping first 10 levels: [dim_f10_level_map](../core_foundation/dim_f10_level_map.md)

## Downstream

- Hiện chưa có downstream trực tiếp trong mart dependency graph.
- View này hiện phù hợp để dùng trực tiếp cho dashboard hoặc phân tích ad-hoc về onboarding, early progression, drop-off, win/fail/no-end, timing và move usage.

## DDL tham chiếu

- DDL: [vw_funnel_first_open_to_level10.sql](../../../sql/ddl/current_live_analysis/vw_funnel_first_open_to_level10.sql)

## Lưu ý khi đọc cùng các view khác

- View này dùng cửa sổ quan sát 24 giờ kể từ `first_open_time_utc`, khác với `vw_d1_retention`, vốn dùng D1 theo calendar day local.
- View này gán `app_version` theo event `first_open` đầu tiên và yêu cầu nhiều event follow-up khớp với `app_version` đó.
- View này dùng sequential funnel nghiêm ngặt, nên nếu một previous step bị thiếu tracking hoặc sai thứ tự, các step sau có thể không được tính là reached hợp lệ.
- Khi dùng view này để kết luận về difficulty hoặc progression, nên đọc cùng `vw_pipeline_health_daily` để kiểm tra rủi ro tracking.
