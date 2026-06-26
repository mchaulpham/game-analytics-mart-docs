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

`end_level_user_count` = `win_only_user_cont` + `fail_only_user_count` + `win_after_fail_or_mixed_user_count`


