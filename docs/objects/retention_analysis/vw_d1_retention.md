ĐẶC TẢ VIEW `vw_d1_retention`
===

# 1. Thông tin chung

## 1.1. Tên view

```
project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_d1_retention
```

## 1.2. Mục đích

View `vw_d1_retention` dùng để phân tích __D1 Retention__ theo từng `app_version`.

View này phục vụ 3 metric chính:

1. D1 App Retention
2. D1 Gameplay Retention
3. D1 Gameplay Activation among Returners

Trong đó:

- D1 App Retention: User quay lại app vào ngày D1 và có ít nhất 1 active event.
- D1 Gameplay Retention: User quay lại app vào ngày D1 và có ít nhất 1 event `Start_level`.
- D1 Gameplay Activation among Returners: Trong nhóm user đã quay lại app vào D1, bao nhiêu % user thật sự bắt đầu chơi level.

---

# 2. Nguồn dữ liệu

## 2.1. Bảng raw GA4

View đọc dữ liệu từ GA4 daily export tables:

```
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Chỉ đọc các bảng daily có `_TABLE_SUFFIX` dạng ngày:

```SQL
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

Điều ngày có nghĩa là view chỉ dùng các bảng: `events_YYYYMMDD`

Không dùng các bảng intraday.

## 2.2. Các raw field được sử dụng

View sử dụng các field chính sau từ GA4:

| Raw field | Ý nghĩa |
|:---|:---|
| `event_timestamp` | Timestamp của event ở dạng microseconds |
| `user_pseudo_id` | ID ẩn danh của user/device trong GA4 |
| `event_name` | Tên event |
| `app_info.version` | App version tại thời điểm event được ghi nhận |

View không sử dụng `event_params` cho logic D1 Retention hiện tại.

---

# 3. Timezone và quy ước ngày

## 3.1. Timezone sử dụng

View dùng timezone: `Asia/Ho_Chi_Minh` để chuyển timestamp UTC sang ngày local.

Cách chuyển:

```SQL
DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Ho_Chi_Minh')
```

## 3.2. Định nghĩa `cohort_date`

`cohort_date` : ngày local mà user có `first_open` đầu tiên.

Ví dụ: User có `first_open` vào 2026-07-01 theo `Asia/Ho_Chi_Minh` → cohort_date = 2026-07-01.

## 3.3. Định nghĩa `d1_date`

`d1_date` = `cohort_date` + 1 day

Ví dụ: `cohort_date` = 2026-07-01 → `d1_date` = 2026-07-02

View sử dụng D1 theo __calendar day__, không dùng rolling 24h window.

---

# 4. Định nghĩa cohort

## 4.1. Cohort user

- Một user được đưa vào cohort nếu user có event: `first_open`

- Đây là `first_open` đầu tiên của user trong toàn bộ dữ liệu được đọc.

- Logic xác định `first_open` đầu tiên:

```SQL
ROW_NUMBER() OVER (
  PARTITION BY user_pseudo_id
  ORDER BY event_time_utc
) = 1
```

## 4.2. App version của cohort

`app_version` : `app_info.version` tại event `first_open` đầu tiên của user.

- Điều này có nghĩa là user được gán vào version theo __version tại lần `first_open` đầu tiên__, không phải version tại ngày D1.

- Ví dụ:

  - User `first_open` ở version `0.4.1`

  - Sau đó D1 user update lên version `0.4.2` và phát sinh event → user vẫn thuộc cohort app_version = 0.4.1

## 4.3. Điều kiện dữ liệu đủ maturity

- View chỉ lấy cohort date từ `release_first_cohort_date` đến `matured_end_cohort_date`

Trong đó:

```
matured_end_cohort_date =
  CURRENT_DATE('Asia/Ho_Chi_Minh') - 2 days
```

- Mục đích là để đảm bảo ngày D1 đã có đủ dữ liệu quan sát.

- Ví dụ nếu query chạy vào ngày local `2026-07-08` thì `matured_end_cohort_date` = 2026-07-06. Cohort ngày `2026-07-06` có D1 là `2026-07-07`, và ngày D1 này đã hoàn tất khi query chạy vào `2026-07-08`.

---

# 5. Định nghĩa D1 activity

## 5.1. D1 App Retention

- Một user được tính là `D1 App Retained` nếu user có ít nhất 1 active event vào `d1_date`.

- Active event được định nghĩa là:

```SQL
event_name NOT IN ('first_open', 'app_remove')
```

- Nghĩa là loại bỏ `first_open` và `app_remove` và tính tất cả event còn lại là active event.

- Công thức user-level: `is_d1_app_retention` = `d1_active_event_count` > 0

- Trong đó `d1_active_event_count` : số event vào `d1_date` của user, loại trừ `first_open`và `app_remove`.

## 5.2. D1 Gameplay Retention

- Một user được tính là `D1 Gameplay Retention` nếu user có ít nhất 1 event `Start_level` vào `d1_date`.

- Công thức user-level: `is_d1_gameplay_retained` = `d1_start_level_event_count` > 0

- Trong đó `d1_start_level_event_count` : số event `Start_level` của user vào `d1_date`.

## 5.3. D1 Gameplay Activation among Returners

- Metric này đo trong nhóm user đã quay lại app vào D1, bao nhiêu user thật sự bắt đầu chơi level.

- Công thức: D1 Gameplay Activation among Returners = D1 Gameplay Retained Users / D1 App Retained Users * 100

---

# 6. Grain của view

View có 2 loại row, phân biệt bằng field: `result_grain`

## 6.1. Daily grain

- `result_grain` = 'daily'

- Mỗi row đại diện cho:
  - 1 `app_version`
  - 1 `cohort_date`
  - 1 `d1_date`
 
- Grain: `app_version` * `cohort_date` * `d1_date`
 
## 6.2. Total grain

- `result_grain` = 'total'

- Mỗi row đại diện cho tổng hợp toàn bộ `cohort_date` đã matured của một `app_version`.

- Grain: `app_version`
  - Với total row: `cohort_date` = NULL và `d1_date` = NULL
 
- Total row không phải trung bình đơn giản của daily retention rate. Total row được tính bằng cách cộng user-level data trước, rồi mới tính tỷ lệ.

- Ví dụ:
  - total D1 App Retention = Tổng `D1_app_retained_users` của tất cả `cohort_date` / Tổng `cohort_users` của tất cả `cohort_date` * 100

---

# 7. Đặc tả từng field trong view

| Field | Type | Ý nghĩa | Đặc tả |
|:---|:---|:---|:---|
| `result_grain` | STRING | Xác định row là daily row hay total row | Value = 'daily' hoặc 'total'<br><br>- 'daily' = row theo từng `cohort_date`<br>- 'total' = row tổng hợp toàn bộ `matured_cohort_date` của `app_version` |
| `app_version` | STRING | App version của cohort user | - Nguồn dữ liệu: `app_info.version` tại event `first_open` đầu tiên của user.<br>- D1 activity có thể xảy ra ở `app_version` khác. Nhưng user vẫn được tính về `app_version` tại `first_open` đầu tiên. |
| `cohort_date` | DATE | Ngày local mà user có `first_open` đầu tiên | - Nguồn dữ liệu: DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Ho_Chi_Minh') |
| `d1_date` | DATE | Ngày D1 tương ứng với `cohort_date` | - Công thức; `d1_date` = `cohort_date` + 1 day<br>- SQL logic: DATE_ADD(cohort_date, INTERVAL 1 DAY) |
| `release_first_cohort_date` | DATE | Ngày cohort đầu tiên có `first_open` của `app_version` đó. | - Nguồn dữ liệu: event `first_open` đầu tiên của từng `app_version`.<br>- Công thức: MIN(DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Ho_Chi_Minh'))<br>- Điều kiện raw:<br>• `event_name` = 'first_open'<br>• `app_info.version` IS NOT NULL<br>• `user_pseudo_id` IS NOT NULL |
| `matured_end_cohort_date` | DATE | Ngày cohort cuối cùng được đưa vào view vì đã đủ dữ liệu để quan sát D1. | - Công thức: DATE_SUB(CURRENT_DATE('Asia/Ho_Chi_Minh'), INTERVAL 2 DAY) |
| `query_local_date` | DATE | Ngày local tại thời điểm query/view được thực thi. | - Công thức: CURRENT_DATE('Asia/Ho_Chi_Minh') |
| `cohort_user_count` | INT64 | Số user thuộc cohort | - Daily row: Số user có `first_open` đầu tiên vào `cohort_date` thuộc `app_version` đó.<br>- Total row: Tổng số user của tất cả `matured_cohort_dat` thuộc `app_version` đó.<br>- Công thức: COUNT(DISTINCT user_pseudo_id) |
| `d1_app_retained_user_count` | INT64 | Số cohort user có ít nhất 1 active event vào ngày D1. | - User-level condition: `is_d1_app_retained` = TRUE trong đó `is_d1_app_retained` = `d1_active_event_count` > 0<br>- Avtive event: `event_name` NOT IN ('first_open', 'app_remove')<br>- Công thức aggregate: COUNT(DISTINCT IF(is_d1_app_retained, user_pseudo_id, NULL)) |
| `d1_app_retention_pct` | FLOAT64 | Tỷ lệ user quay lại app vào D1 | - Công thức: `d1_app_retention_pct` = `d1_app_retained_user_count` / `cohort_user_count` * 100<br>- SQL logic: ROUND( SAFE_DIVIDE( COUNT(DISTINCT IF(is_d1_app_retained, user_pseudo_id, NULL)), COUNT(DISTINCT user_pseudo_id) ) * 100, 2 ) |
| `d1_gameplay_retained_user_count` | INT64 | Số cohort user có ít nhất 1 event `Start_level` vào ngày D1. | - User-level codition: `is_d1_gameplay_retained` = TRUE trong đó `is_d1_gameplay_retained` = `d1_start_level_event_count` > 0<br>- Công thức aggragate: COUNT(DISTINCT IF(is_d1_gameplay_retained, user_pseudo_id, NULL)) |
| `d1_gameplay_retention_pct` | FLOAT64 | Tỷ lệ user quay lại và bắt đầu chơi level vào D1. | - Công thức: `d1_gameplay_retention_pct` = `d1_gameplay_retained_user_count` / `cohort_usr_count` * 100<br>- SQL logic: ROUND( SAFE_DIVIDE( COUNT(DISTINCT IF(is_d1_gameplay_retained, user_pseudo_id, NULL)), COUNT(DISTINCT user_pseudo_id) ) * 100, 2 ) |
| `d1_gameplay_activation_among_returners_pct` | FLOAT64 | Trong nhóm user đã quay lại app vào D1, có bao nhiêu % user bắt đầu chơi level. | - Công thức: `d1_gameplay_activation_among_returners_pct` = `d1_gameplay_retained_user_count` / `d1_app_retained_user_count` * 100<br>- SQL logic: ROUND( SAFE_DIVIDE( COUNT(DISTINCT IF(is_d1_gameplay_retained, user_pseudo_id, NULL)), COUNT(DISTINCT IF(is_d1_app_retained, user_pseudo_id, NULL)) ) * 100, 2 ) |
| `d1_app_retained_without_gameplay_user_count` | INT64 | Số user quay lại app vào D1 nhưng không có `Start_level` vào D1. | - User-level condition: `is_d1_app_retained` = TRUE AND `is_d1_gameplay_retained` = FALSE<br>- Công thức: COUNT(DISTINCT IF( is_d1_app_retained AND NOT is_d1_gameplay_retained, user_pseudo_id, NULL ))<br>- Kiểm tra: `d1_app_retained_without_gameplay_user_count` = `d1_app_retained_user_count` - `d1_gameplay_retained_user_count` |
| `d1_app_retained_without_gameplay_share_pct` | FLOAT64 | Trong nhóm user đã quay lại app vào D1, có bao nhiêu % user không bắt đầu chơi level. | - Công thức: d1_app_retained_without_gameplay_share_pct = d1_app_retained_without_gameplay_user_count / d1_app_retained_user_count × 100<br>- SQL logic: ROUND( SAFE_DIVIDE( COUNT(DISTINCT IF( is_d1_app_retained AND NOT is_d1_gameplay_retained, user_pseudo_id, NULL )), COUNT(DISTINCT IF(is_d1_app_retained, user_pseudo_id, NULL)) ) * 100, 2 ) |
| `avg_seconds_first_open_to_first_d1_active_event` | FLOAT64 | Thời gian trung bình từ `first_open` đến active event đầu tiên vào D1. | - Chỉ tính user có D1 active event.<br>- Công thức user-level: `seconds_first_open_to_first_d1_active_event` = `first_d1_active_event_time_utc` - `first_open_time_utc`<br>- SQL logic: AVG( IF( first_d1_active_event_time_utc IS NOT NULL, TIMESTAMP_DIFF( first_d1_active_event_time_utc, first_open_time_utc, SECOND ), NULL ) ) |
| `median_seconds_first_open_to_first_d1_active_event` | INT64 | Median thời gian từ `first_open` đến active event đầu tiên vào D1. | - CHỉ tính user có D1 active event.<br>- SQL logic: APPROX_QUANTILES( IF( first_d1_active_event_time_utc IS NOT NULL, TIMESTAMP_DIFF( first_d1_active_event_time_utc, first_open_time_utc, SECOND ), NULL ), 100 )[OFFSET(50)] |
| `p90_seconds_first_open_to_first_d1_active_event` | INT64 | P90 thời gian từ `first_open` đến active event đầu tiên vào D1. | - Chỉ tính user có D1 active event.<br>- SQL logic: APPROX_QUANTILES(..., 100)[OFFSET(90)] |
| `avg_seconds_first_open_to_first_d1_start_level` | FLOAT64 | Thời gian trung bình từ `first_open` đến `Start_level` đầu tiên vào D1. | - Chỉ tính user có `Start_level` đầu tiên vào D1.<br>- Công thức user-level: `seconds_first_open_to_first_d1_start_level` = `first_d1_start_level_time_utc` - `first_open_time_utc`<br>- SQL logic: AVG( IF( first_d1_start_level_time_utc IS NOT NULL, TIMESTAMP_DIFF( first_d1_start_level_time_utc, first_open_time_utc, SECOND ), NULL ) ) |
| `median_seconds_first_open_to_first_d1_start_level` | INT64 | Median thời gian từ `first_open` đến `Start_level` đầu tiên vào D1. | - Chỉ tính user có `Start_level` vào D1.<br>- SQL logic: APPROX_QUANTILES( IF( first_d1_start_level_time_utc IS NOT NULL, TIMESTAMP_DIFF( first_d1_start_level_time_utc, first_open_time_utc, SECOND ), NULL ), 100 )[OFFSET(50)] |
| `p90_seconds_ first_open_to-first_d1_start_level` | INT64 | P90 thời gian từ `first_open` đến `Start_level` đầu tiên vào D1. | - Chỉ tính user có `Start_level` vào D1.<br>-SQL logic: APPROX_QUANTILES(..., 100)[OFFSET(90)] |
| `first_open_first_time_utc` | TIMESTAMP | Timestamp UTC sớm nhất của `first_open` trong row đó. | - Daily row: `first_open` đầu tiên trong toàn bộ `matured_cohort-date` của `app_version`.<br>- Total row: `first_open` đầu tiên trong toàn bộ `maturec_cohort_date` của `app_version`.<br>- Công thức: MIN(first_open_time_utc) |
| `first_open_last_time_utc` | TIMESTAMP | Timestamp UTC muộn nhất của `first_open` trong row đó. | - Daily row: `first_open` cuối cùng trong `cohort_date` đó.<br> - Total row: `first_open` cuối cùng trong toàn bộ `matured_cohort_date` của `app_version`.<br>- Công thức: MAX(first_open_time_utc) |
| `query_time_utc` | TIMESTAMP | Thời điểm view/query được thực thi | - Công thức: CURRENT_TIMESTAMP()<br><br>- Lưu ý: Vì đây là view, `query_time_utc` sẽ thay đổi mỗi lần query lại view. |

---

# 8. Xử lý và làm sạch dữ liệu raw

## 8.1. Lọc bảng daily

- View chỉ đọc bảng daily:

```SQL
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

- Mục đích:
  - Loại bỏ intraday tables.
  - Đảm bảo dữ liệu ổn định hơn khi phân tích retention.
 
## 8.2. Loại bỏ `user_pseudp_id` NULL

- View chỉ lấy event có: `user_pseudo_id` IS NOT NULL

- Mục đích: Đảm bảo có thể định danh user để tạo cohort và tính retention.

## 8.3. Loại bỏ `app_version` NULL ở cohort

- Với event `first_open`, view yêu cầu: `app_info.version` IS NOT NULL

- Mục đích: Đảm bảo mỗi cohort user được gán vào một `app_version` cụ thể.

## 8.4. Chuẩn hóa timestamp

- Raw timestamp từ GA4 là microseconds: `event_timestamp`
 
- View chuyển sang TIMESTAMP UTC bằng:
 
```SQL
TIMESTAMP_MICROS(event_timestamp)
```

- Sau đó chuyển sang local date bằng:

```SQL
DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Ho_Chi_Minh')
```

## 8.5. Deduplicate `first_open`

- Nếu một user có nhiều event `first_open`, view chỉ lấy event đầu tiên theo thời gian:

```SQL
ROW_NUMBER() OVER (
  PARTITION BY user_pseudo_id
  ORDER BY event_time_utc
) = 1
```

- Mục đích: Mỗi user chỉ thuộc một cohort duy nhất.

## 8.6. Không clean `event_name`

- View không chuẩn hóa chữ hoa/thường của `event_name`.

- Nếu sau này tracking đổi tên event, view cần được cập nhật.

## 8.7. Không clean `event_params`

View hiện tại không dùng `event_params`, nên không có bước xử lý hoặc clean parameter.

---

# 9. Logic xử lý dữ liệu

## 9.1. Bước 1 — Tìm ngày release đầu tiên của từng `app_version`

- Từ raw event `first_open`, view tính:

`release_first_open_cohort_date` = ngày local đầu tiên có `first_open` của `app_version` đó.

## 9.2. Bước 2 — Xác định ngày cohort cuối cùng đã matured

- View tính:

`matured_end_cohort_date` = `current_local_date` - 2 days

- Chỉ cohort có `cohort_date` <= `matured_end_cohort_date` mới được đưa vào phân tích.

## 9.3. Bước 3 — Xác định `first_open` đầu tiên của từng user

- Với mỗi `user_pseudo_id`, view lấy `first_open` đầu tiên theo thời gian.

- Output user-level:
  - `user_pseudo_id`
  - `app_version` tại `first_open`
  - `first_open_time_utc`
  - `cohort_date`
  - `d1_date`
 
## 9.4. Bước 4 — Join D1 activity

- View join event của cùng user vào ngày `d1_date`.

- Điều kiện join chính:
  - same `user_pseudo_id`
  - `event_local_date` = `d1_date`
  - `event_time_utc` > `first_open_time_utc`
 
- Lưu ý quan trọng: D1 activity không bị ràng buộc phải cùng `app_version` với cohort. Điều này cho phép user được tính retained nếu họ `first_open` ở version A nhưng quay lại D1 ở version B sau khi update.

## 9.5. Bước 5 — Tính user-level flags

- Với mỗi user, view tính:
  - `is_d1_app_retained`
  - `is_d1_gameplay_retained`
 
- Trong đó:
  - `is_d1_app_retained` = `d1_active_event_count` > 0
  - `is_d1_gameplay_retained` = `d1_start_level_event_count` > 0
 
## 9.6. Bước 6 — Aggregate daily rows

- Group by:
  - `app_version`
  - `cohort_date`
  - `d1_date`

- Tạo output daily retention.

## 9.7. Bước 7 — Aggregate total rows

- Group by: `app_version`

- Tạo output total retention cho toàn bộ `matured_cohort_date` của version.

---

## Liên kết liên quan

• Tổng quan repository: [README](../../../README.md)

• Framework: [Framework Pipeline](../../pipeline/framework_pipeline.md)

• Dependency graph: [Mart Dependency Graph](../../graph/mart_dependency_graph.md)

### Upstream

• GA4 raw events: `project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*`

### Downstream

• Hiện chưa có downstream trực tiếp trong `game_analytics_mart`.

### Object cùng nhóm

• Nhóm tài liệu: `docs/objects/retention_analysis/`

• DDL tham chiếu: `../../../sql/ddl/retention_analysis/vw_d1_retention.sql`

### Tài liệu liên quan

• [vw_funnel_first_open_to_level10](../current_live_analysis/vw_funnel_first_open_to_level10.md) — view funnel 24h từ `first_open` đến Level 10, thường được đọc song song với D1 Retention để đánh giá early player experience.
