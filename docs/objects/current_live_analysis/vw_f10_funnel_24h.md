# vw_f10_funnel_24h

## 1. Mục đích

`vw_f10_funnel_24h` là view tổng hợp phễu 10 level đầu trong vòng 24 giờ kể từ lần đầu người chơi bắt đầu Level 1.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

“Phễu” nghĩa là chuỗi bước mà người chơi cần đi qua. Với view này, phễu chính là chuỗi 10 level đầu:

```text id="dxd4u8"
Level 1
  ↓
Level 2
  ↓
Level 3
  ↓
...
  ↓
Level 10
```

Mục tiêu chính của view là đo xem người chơi có đi qua từng bước trong 10 level đầu hay không, trong cửa sổ 24 giờ sau lần bắt đầu Level 1.

---

## 2. Vai trò trong hệ thống

`vw_f10_funnel_24h` thuộc nhóm:

```text id="g2yiti"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là đầu ra chính để theo dõi tiến trình người chơi qua 10 level đầu.

View này không đo thắng hay thua level. Nó đo việc người chơi có bắt đầu từng level trong chuỗi 10 level đầu hay không.

Luồng phụ thuộc chính:

```text id="bjdl53"
dim_f10_level_map
vw_level_start_eff
  └── vw_f10_funnel_24h
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu người chơi bắt đầu Level 1?
2. Có bao nhiêu người chơi đã đủ 24 giờ để quan sát sau khi bắt đầu Level 1?
3. Có bao nhiêu người chơi đi tới từng level trong 10 level đầu?
4. Tỷ lệ chuyển đổi từ level trước sang level sau là bao nhiêu?
5. Tỷ lệ rơi từ level trước sang level sau là bao nhiêu?
6. Tỷ lệ chuyển đổi cộng dồn từ Level 1 đến từng level là bao nhiêu?
7. Level nào trong 10 level đầu làm người chơi rơi nhiều nhất?
8. Có app version nào có phễu 10 level đầu kém hơn phiên bản khác không?
9. Mapping 10 level đầu có đang đủ và đúng không?
10. Người chơi có đi qua các bước theo đúng thứ tự hay không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_f10_funnel_24h` đại diện cho một bước trong 10 level đầu của một phiên bản game.

Có thể hiểu đơn giản:

```text id="t7bn8f"
Một dòng = một app_version + một progression_step trong 10 level đầu
```

Ví dụ:

```text id="v9joqz"
app_version = 0.3.61
progression_step = 3
level_name = N003
```

nghĩa là dòng đó mô tả kết quả phễu ở bước 3 của phiên bản `0.3.61`.

---

## 5. Loại đối tượng

```text id="ird2qt"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

View dùng `CURRENT_TIMESTAMP()` để xác định người chơi đã đủ 24 giờ kể từ Level 1 hay chưa. Vì vậy, kết quả có thể thay đổi theo thời điểm truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào hai nguồn chính:

| Nguồn                | Vai trò                                         |
| -------------------- | ----------------------------------------------- |
| `dim_f10_level_map`  | Xác định 10 level đầu theo từng phiên bản game. |
| `vw_level_start_eff` | Cung cấp các lần bắt đầu level có hiệu lực.     |

Trong đó, `vw_level_start_eff` bao gồm cả:

* `Start_level` thật;
* một số trường hợp bắt đầu level được suy luận từ `Tutorial_start`.

Điều này giúp phễu không bỏ sót một số trường hợp đặc biệt trong tutorial.

---

## 7. Cách xác định 10 level đầu

View lấy 10 level đầu từ:

```text id="hc7qe9"
dim_f10_level_map
```

với điều kiện:

```sql id="q69q45"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step BETWEEN 1 AND 10
```

Các trường mapping được giữ lại trong kết quả gồm:

```text id="wp76sa"
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

Điều này giúp mỗi dòng phễu vừa có chỉ số phân tích, vừa có thông tin thiết kế level đi kèm.

---

## 8. Cách xác định nhóm Level 1

View xác định Level 1 bằng:

```sql id="kiobxf"
progression_step = 1
```

trong `dim_f10_level_map`.

Sau đó, view tìm lần bắt đầu Level 1 đầu tiên của mỗi người chơi từ:

```text id="c8ctl7"
vw_level_start_eff
```

Mỗi người chơi trong một phiên bản game chỉ có một thời điểm Level 1 đầu tiên:

```text id="t3wh0v"
level1_start_time_utc
```

Đây là mốc bắt đầu để tính cửa sổ quan sát 24 giờ.

---

## 9. Logic 24 giờ

Với mỗi người chơi đã bắt đầu Level 1, view tạo cửa sổ quan sát:

```text id="kwccml"
level1_start_time_utc
đến
level1_start_time_utc + 24 giờ
```

Một người chơi được xem là đã đủ thời gian quan sát nếu:

```text id="fp7f24"
level1_start_time_utc <= thời điểm chạy view - 24 giờ
```

Các chỉ số phễu theo từng bước chỉ tính trên nhóm người chơi đã đủ 24 giờ.

Trong tài liệu này, “đã đủ thời gian quan sát” nghĩa là người chơi đã qua đủ 24 giờ kể từ mốc bắt đầu Level 1, nên có thể đánh giá đầy đủ hành vi trong 24 giờ đầu sau Level 1.

---

## 10. Logic đếm người chơi theo từng bước

View không chỉ đếm người chơi có bắt đầu một level bất kỳ. View còn kiểm tra chuỗi thứ tự.

Một người chơi được tính là đã tới bước hiện tại nếu:

```text id="sfy4ns"
1. Người chơi đã bắt đầu level của bước hiện tại trong 24 giờ sau Level 1
2. Không thiếu bất kỳ bước nào trước đó
3. Không có bước nào bị đi ngược thứ tự thời gian
```

Nói cách khác, đây là phễu có kiểm tra thứ tự bước.

Ví dụ, nếu một người chơi có dữ liệu:

```text id="7wn05y"
Level 1
Level 2
Level 4
```

nhưng thiếu Level 3, người chơi đó không được tính là đã đi tới bước 4 trong phễu tuần tự.

Trong tài liệu này, “phễu tuần tự” nghĩa là người chơi phải đi qua các bước theo đúng thứ tự đã định nghĩa.

---

## 11. Danh sách cột

| Cột                                    | Kiểu dữ liệu | Ý nghĩa                                                              |
| -------------------------------------- | ------------ | -------------------------------------------------------------------- |
| `app_version`                          | `STRING`     | Phiên bản game.                                                      |
| `progression_step`                     | `INT64`      | Bước trong 10 level đầu.                                             |
| `progression_slot_id`                  | `STRING`     | Mã vị trí của bước trong tiến trình.                                 |
| `level_name`                           | `STRING`     | Tên level của bước đó.                                               |
| `level_design_id`                      | `STRING`     | Mã thiết kế level.                                                   |
| `level_type`                           | `STRING`     | Loại level.                                                          |
| `previous_progression_step`            | `INT64`      | Bước trước đó trong tiến trình.                                      |
| `next_progression_step`                | `INT64`      | Bước tiếp theo trong tiến trình.                                     |
| `mapping_confidence_status`            | `STRING`     | Mức độ tin cậy của mapping.                                          |
| `mapping_source`                       | `STRING`     | Nguồn tạo mapping.                                                   |
| `level1_cohort_user_count`             | `INT64`      | Tổng số người chơi có bắt đầu Level 1.                               |
| `matured_level1_cohort_user_count`     | `INT64`      | Số người chơi đã đủ 24 giờ kể từ Level 1.                            |
| `not_matured_level1_cohort_user_count` | `INT64`      | Số người chơi chưa đủ 24 giờ kể từ Level 1.                          |
| `user_count`                           | `INT64`      | Số người chơi đã đi tới bước hiện tại theo đúng thứ tự trong 24 giờ. |
| `previous_step_user_count`             | `INT64`      | Số người chơi ở bước trước.                                          |
| `drop_from_previous_step_user_count`   | `INT64`      | Số người rơi từ bước trước sang bước hiện tại.                       |
| `conversion_rate_from_previous_step`   | `FLOAT64`    | Tỷ lệ chuyển đổi từ bước trước sang bước hiện tại.                   |
| `drop_rate_from_previous_step`         | `FLOAT64`    | Tỷ lệ rơi từ bước trước sang bước hiện tại.                          |
| `funnel_start_user_count`              | `INT64`      | Số người chơi ở bước đầu của phễu.                                   |
| `cumulative_drop_user_count`           | `INT64`      | Số người rơi cộng dồn từ bước đầu đến bước hiện tại.                 |
| `cumulative_conversion_rate`           | `FLOAT64`    | Tỷ lệ chuyển đổi cộng dồn từ bước đầu đến bước hiện tại.             |
| `cumulative_drop_rate`                 | `FLOAT64`    | Tỷ lệ rơi cộng dồn từ bước đầu đến bước hiện tại.                    |
| `first_level1_start_time_utc`          | `TIMESTAMP`  | Thời điểm Level 1 sớm nhất trong phiên bản đó.                       |
| `last_level1_start_time_utc`           | `TIMESTAMP`  | Thời điểm Level 1 muộn nhất trong phiên bản đó.                      |
| `mart_updated_at`                      | `TIMESTAMP`  | Thời điểm view được truy vấn.                                        |

---

## 12. Công thức chỉ số chính

### 12.1 Số người rơi từ bước trước

```text id="rn7kje"
drop_from_previous_step_user_count
=
previous_step_user_count - user_count
```

Ý nghĩa: số người đã tới bước trước nhưng không tới bước hiện tại.

---

### 12.2 Tỷ lệ chuyển đổi từ bước trước

```text id="z35957"
conversion_rate_from_previous_step
=
user_count
/
previous_step_user_count
```

Ý nghĩa: trong số người đã tới bước trước, bao nhiêu phần trăm đi tiếp tới bước hiện tại.

---

### 12.3 Tỷ lệ rơi từ bước trước

```text id="910mpv"
drop_rate_from_previous_step
=
(previous_step_user_count - user_count)
/
previous_step_user_count
```

Ý nghĩa: trong số người đã tới bước trước, bao nhiêu phần trăm không tới bước hiện tại.

---

### 12.4 Tỷ lệ chuyển đổi cộng dồn

```text id="8zllhq"
cumulative_conversion_rate
=
user_count
/
funnel_start_user_count
```

Ý nghĩa: trong số người bắt đầu phễu ở Level 1, bao nhiêu phần trăm đi tới bước hiện tại.

---

### 12.5 Tỷ lệ rơi cộng dồn

```text id="em9eb9"
cumulative_drop_rate
=
(funnel_start_user_count - user_count)
/
funnel_start_user_count
```

Ý nghĩa: trong số người bắt đầu phễu ở Level 1, bao nhiêu phần trăm đã rơi trước hoặc tại bước hiện tại.

---

## 13. Kiểm tra chất lượng dữ liệu khuyến nghị

### 13.1 Kiểm tra mỗi app version có đủ 10 bước không

```sql id="6dp6m8"
SELECT
  app_version,
  COUNT(*) AS step_count,
  MIN(progression_step) AS min_progression_step,
  MAX(progression_step) AS max_progression_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

GROUP BY
  app_version

ORDER BY
  app_version;
```

Kỳ vọng thông thường:

```text id="iysrfl"
step_count = 10
min_progression_step = 1
max_progression_step = 10
```

Nếu thiếu bước, cần kiểm tra `dim_f10_level_map`.

---

### 13.2 Kiểm tra số người ở bước 1 có bằng nhóm đã đủ 24 giờ không

```sql id="isbp2h"
SELECT
  app_version,
  matured_level1_cohort_user_count,
  user_count AS step1_user_count,
  matured_level1_cohort_user_count - user_count AS gap

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

WHERE
  progression_step = 1

ORDER BY
  app_version;
```

Kỳ vọng:

```text id="k7gcby"
gap = 0
```

Nếu khác 0, cần kiểm tra logic Level 1 hoặc `vw_level_start_eff`.

---

### 13.3 Kiểm tra số người có giảm dần theo bước không

```sql id="962n5d"
SELECT
  app_version,
  progression_step,
  user_count,
  previous_step_user_count,

  CASE
    WHEN previous_step_user_count IS NOT NULL
      AND user_count > previous_step_user_count
      THEN 'Bất thường: user_count lớn hơn bước trước'
    ELSE 'OK'
  END AS check_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

ORDER BY
  app_version,
  progression_step;
```

Trong phễu tuần tự, `user_count` không nên lớn hơn `previous_step_user_count`.

---

### 13.4 Kiểm tra tỷ lệ nằm trong khoảng 0 đến 1

```sql id="loebuu"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

WHERE
  conversion_rate_from_previous_step < 0
  OR conversion_rate_from_previous_step > 1
  OR drop_rate_from_previous_step < 0
  OR drop_rate_from_previous_step > 1
  OR cumulative_conversion_rate < 0
  OR cumulative_conversion_rate > 1
  OR cumulative_drop_rate < 0
  OR cumulative_drop_rate > 1

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="v1ztuj"
Không trả ra dòng nào
```

---

### 13.5 Kiểm tra phiên bản có nhiều người chưa đủ 24 giờ

```sql id="d0wukr"
SELECT
  app_version,
  level1_cohort_user_count,
  matured_level1_cohort_user_count,
  not_matured_level1_cohort_user_count,

  ROUND(
    SAFE_DIVIDE(
      matured_level1_cohort_user_count,
      level1_cohort_user_count
    ),
    4
  ) AS matured_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

WHERE
  progression_step = 1

ORDER BY
  app_version;
```

Nếu `matured_rate` thấp, kết quả 24 giờ chưa ổn định.

---

## 14. Cách sử dụng khuyến nghị

### 14.1 Xem phễu 10 level đầu

```sql id="pum7wt"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  previous_step_user_count,
  conversion_rate_from_previous_step,
  drop_rate_from_previous_step,
  cumulative_conversion_rate,
  cumulative_drop_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

ORDER BY
  app_version,
  progression_step;
```

---

### 14.2 Tìm bước rơi mạnh nhất

```sql id="rxsa4t"
SELECT
  app_version,
  progression_step,
  level_name,
  previous_step_user_count,
  user_count,
  drop_from_previous_step_user_count,
  drop_rate_from_previous_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

WHERE
  previous_step_user_count IS NOT NULL

ORDER BY
  drop_rate_from_previous_step DESC,
  drop_from_previous_step_user_count DESC;
```

Truy vấn này giúp xác định bước nào trong 10 level đầu làm người chơi rơi nhiều nhất.

---

### 14.3 So sánh phễu theo phiên bản game

```sql id="4al2dr"
SELECT
  app_version,
  progression_step,
  level_name,
  matured_level1_cohort_user_count,
  user_count,
  cumulative_conversion_rate,
  cumulative_drop_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`

ORDER BY
  progression_step,
  app_version;
```

Truy vấn này phù hợp để so sánh cùng một bước giữa nhiều phiên bản game.

---

## 15. Lưu ý khi sử dụng

### 15.1 View này đo việc bắt đầu level, không đo việc thắng level

`user_count` trong view này nghĩa là người chơi đã bắt đầu level của bước đó theo đúng thứ tự.

Nó không có nghĩa là người chơi đã thắng level đó.

Nếu muốn phân tích thắng, thua, số lần thử, nên dùng:

```text id="e8s8m9"
vw_level_attempts
```

hoặc các view chẩn đoán thiết kế như:

```text id="c1e5vv"
vw_f10_design_diag_24h
```

---

### 15.2 View dùng dữ liệu bắt đầu level có hiệu lực

Nguồn bắt đầu level là:

```text id="d2ni1y"
vw_level_start_eff
```

Nguồn này có thể bao gồm các dòng bắt đầu level được suy luận từ tutorial.

Điều này giúp phễu đầy đủ hơn, nhưng cũng cần đọc cùng với logic tutorial khi phân tích sâu.

---

### 15.3 View chỉ tính người chơi đã đủ 24 giờ cho số người từng bước

Các chỉ số `user_count` theo bước chỉ tính nhóm:

```text id="ricf0s"
matured_level1_cohort
```

Tức là người chơi đã qua đủ 24 giờ kể từ Level 1.

Không nên lấy `level1_cohort_user_count` làm mẫu số chính khi phân tích phễu 24 giờ.

---

### 15.4 Mapping sai sẽ làm phễu sai

Nếu `dim_f10_level_map` sai:

* sai tên level;
* thiếu bước;
* trùng bước;
* sai app version;
* sai `is_active` hoặc `is_required`;

thì kết quả phễu trong view này sẽ sai.

---

### 15.5 Phễu có kiểm tra thứ tự

View này không chỉ đếm người chơi có từng level, mà yêu cầu không thiếu bước trước và không đi ngược thứ tự.

Vì vậy, kết quả có thể thấp hơn so với cách đếm đơn giản “người chơi từng bắt đầu level X”.

---

## 16. Rủi ro nếu view sai logic

| Lỗi logic                              | Ảnh hưởng                                             |
| -------------------------------------- | ----------------------------------------------------- |
| Sai mapping 10 level đầu               | Toàn bộ phễu bị sai.                                  |
| Sai Level 1 cohort                     | Mẫu đầu vào phễu bị sai.                              |
| Sai logic 24 giờ                       | Tính cả người chơi chưa đủ thời gian quan sát.        |
| Không kiểm tra thứ tự bước             | Có thể tính nhầm người chơi nhảy cóc qua level.       |
| Không giữ mapping row có 0 user        | Khó phát hiện level không có người tới.               |
| Nhầm giữa bắt đầu level và thắng level | Diễn giải sai chỉ số `user_count`.                    |
| Không hiểu nguồn `vw_level_start_eff`  | Có thể bỏ qua ảnh hưởng của pseudo start từ tutorial. |

---

## 17. Mức độ quan trọng

```text id="w5s9e0"
Mức độ quan trọng: Cao
```

Lý do:

* view này là báo cáo chính cho phễu 10 level đầu;
* view này giúp tìm bước rơi mạnh trong giai đoạn đầu;
* view này là đầu vào cho `vw_f10_funnel_timing_24h`;
* view này cũng được dùng trong chẩn đoán thiết kế level;
* view này phụ thuộc mạnh vào `dim_f10_level_map`, nên cần kiểm tra sau mỗi thay đổi progression.

---

## 18. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="j2rzfc"
sql/ddl/current_live_analysis/vw_f10_funnel_24h.sql
```

Cấu trúc logic chính của view:

```sql id="8dt9yb"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_funnel_24h`
AS
WITH analysis_params AS (
  SELECT
    CURRENT_TIMESTAMP() AS analysis_time_utc
),

first10_mapping AS (
  SELECT
    app_version,
    progression_step,
    progression_slot_id,
    level_name,
    level_design_id,
    level_type,
    previous_progression_step,
    next_progression_step,
    mapping_source,
    confidence_status

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

  WHERE
    is_active IS TRUE
    AND is_required IS TRUE
    AND progression_step BETWEEN 1 AND 10
),

level1_mapping AS (...),

level1_first_start_by_user AS (...),

level1_cohort AS (...),

matured_level1_cohort AS (...),

user_step_reaches AS (...),

user_step_sequence_check AS (...),

step_counts AS (...),

step_counts_with_mapping AS (...),

step_counts_with_rates AS (...)

SELECT
  app_version,
  progression_step,
  progression_slot_id,
  level_name,
  level_design_id,
  level_type,
  previous_progression_step,
  next_progression_step,
  mapping_confidence_status,
  mapping_source,
  level1_cohort_user_count,
  matured_level1_cohort_user_count,
  not_matured_level1_cohort_user_count,
  user_count,
  previous_step_user_count,
  drop_from_previous_step_user_count,
  conversion_rate_from_previous_step,
  drop_rate_from_previous_step,
  funnel_start_user_count,
  cumulative_drop_user_count,
  cumulative_conversion_rate,
  cumulative_drop_rate,
  first_level1_start_time_utc,
  last_level1_start_time_utc,
  mart_updated_at

FROM
  step_counts_with_rates;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[dim_f10_level_map]]
	- [[vw_level_start_eff]]

- Downstream: [[vw_f10_funnel_timing_24h]]

