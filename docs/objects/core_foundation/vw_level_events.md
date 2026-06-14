# vw_level_events

## 1. Mục đích

`vw_level_events` là view làm sạch và chuẩn hóa các sự kiện level chính từ GA4.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này lấy hai loại sự kiện chính:

```text id="6tdent"
Start_level
End_level
```

Mục đích là tạo một nguồn dữ liệu level thống nhất, trong đó các tham số quan trọng đã được trích xuất, ép kiểu và giữ lại cả giá trị gốc để kiểm tra chất lượng dữ liệu.

Nói ngắn gọn, `vw_level_events` là lớp làm sạch đầu tiên cho dữ liệu level.

---

## 2. Vai trò trong hệ thống

`vw_level_events` thuộc nhóm:

```text id="z94sg6"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này là một trong những view quan trọng nhất của toàn bộ hệ thống, vì hầu hết phân tích level đều cần dữ liệu từ `Start_level` và `End_level`.

View này phục vụ các mục đích chính:

* làm sạch sự kiện bắt đầu level;
* làm sạch sự kiện kết thúc level;
* chuẩn hóa các tham số về kiểu dữ liệu;
* giữ lại giá trị gốc để kiểm tra lỗi tracking;
* làm nguồn cho việc ghép lượt chơi level;
* làm nguồn cho giám sát sức khỏe dữ liệu hằng ngày.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu sự kiện `Start_level` theo ngày?
2. Có bao nhiêu sự kiện `End_level` theo ngày?
3. Level nào có nhiều sự kiện nhất?
4. Sự kiện level có thiếu `level_name` không?
5. Sự kiện level có thiếu `attempt_no` không?
6. `End_level` có thiếu `success`, `duration_sec`, `move_used`, hoặc `move_left` không?
7. `duration_sec` có bị sai định dạng không?
8. Phiên bản game nào đang gửi sự kiện level?
9. Có bao nhiêu sự kiện level có dữ liệu booster?
10. Dữ liệu raw và dữ liệu đã ép kiểu có khớp nhau không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_events` đại diện cho một sự kiện level gốc từ GA4 sau khi được trích xuất tham số.

Có thể hiểu đơn giản:

```text id="bi0owi"
Một dòng = một sự kiện Start_level hoặc End_level của một người chơi tại một thời điểm
```

View này chưa ghép `Start_level` với `End_level`.

Nếu muốn phân tích một lượt chơi level hoàn chỉnh, nên dùng:

```text id="t5z34h"
vw_level_attempts
```

Trong tài liệu này, “lượt chơi level” nghĩa là một lần người chơi bắt đầu level, và nếu có dữ liệu hợp lệ thì được ghép với sự kiện kết thúc level tương ứng.

---

## 5. Loại đối tượng

```text id="d0un0v"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính trực tiếp từ dữ liệu GA4 thô mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này đọc trực tiếp từ dữ liệu GA4 thô:

```text id="4ak1sh"
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Trong đó:

* `events_*` là các bảng sự kiện GA4 được chia theo ngày;
* `_TABLE_SUFFIX` là hậu tố ngày của bảng, ví dụ `20260531`;
* view chỉ lấy các bảng có hậu tố đúng dạng 8 chữ số;
* view chỉ giữ lại sự kiện `Start_level` và `End_level`.

---

## 7. Các view đang sử dụng view này

`vw_level_events` được sử dụng bởi các view sau:

| View sử dụng               | Mục đích sử dụng                                                           |
| -------------------------- | -------------------------------------------------------------------------- |
| `vw_level_tutorial_flag`   | Lấy sự kiện `Start_level` để đánh dấu các lần bắt đầu level gần hướng dẫn. |
| `vw_level_attempts`        | Ghép `Start_level` với `End_level` để tạo dữ liệu lượt chơi level.         |
| `vw_f10_timing_24h`        | Đo thời gian và trạng thái kết thúc level trong 10 level đầu.              |
| `vw_pipeline_health_daily` | Kiểm tra chất lượng dữ liệu level hằng ngày.                               |

Do được nhiều view lõi sử dụng, mọi thay đổi trong `vw_level_events` có thể ảnh hưởng trực tiếp đến các phân tích level phía sau.

---

## 8. Danh sách cột

| Cột                  | Kiểu dữ liệu | Ý nghĩa                                                        |
| -------------------- | ------------ | -------------------------------------------------------------- |
| `event_table_suffix` | `STRING`     | Hậu tố ngày của bảng GA4 gốc, ví dụ `20260531`.                |
| `event_date`         | `STRING`     | Ngày sự kiện theo định dạng của GA4.                           |
| `event_time_utc`     | `TIMESTAMP`  | Thời điểm xảy ra sự kiện theo múi giờ UTC.                     |
| `user_pseudo_id`     | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                 |
| `app_version`        | `STRING`     | Phiên bản game tại thời điểm sự kiện xảy ra.                   |
| `event_name`         | `STRING`     | Tên sự kiện. Chỉ gồm `Start_level` hoặc `End_level`.           |
| `level_name`         | `STRING`     | Tên level lấy từ tham số `level_name`.                         |
| `attempt_no`         | `INT64`      | Số lần thử level, được ép kiểu từ giá trị gốc.                 |
| `success`            | `BOOL`       | Trạng thái thắng hoặc thua ở `End_level`.                      |
| `fail_reason`        | `STRING`     | Lý do thua nếu có.                                             |
| `move_used`          | `INT64`      | Số bước đi đã dùng.                                            |
| `move_left`          | `INT64`      | Số bước đi còn lại.                                            |
| `duration_sec`       | `FLOAT64`    | Thời lượng level theo giây.                                    |
| `gold`               | `INT64`      | Số vàng tại thời điểm sự kiện, nếu có tracking.                |
| `lives_left`         | `INT64`      | Số mạng còn lại tại thời điểm sự kiện, nếu có tracking.        |
| `poka_name`          | `STRING`     | Tham số `poka_name` nếu có.                                    |
| `booster_used`       | `STRING`     | Booster được dùng trong level, nếu có.                         |
| `booster_pre_used`   | `STRING`     | Booster được chọn trước khi vào level, nếu có.                 |
| `attempt_no_raw`     | `STRING`     | Giá trị gốc của `attempt_no` trước khi ép kiểu.                |
| `success_raw`        | `STRING`     | Giá trị gốc của `success` trước khi chuyển thành đúng/sai.     |
| `move_used_raw`      | `STRING`     | Giá trị gốc của `move_used` trước khi ép kiểu.                 |
| `move_left_raw`      | `STRING`     | Giá trị gốc của `move_left` trước khi ép kiểu.                 |
| `duration_sec_raw`   | `STRING`     | Giá trị gốc của `duration_sec` trước khi chuyển thành số thực. |
| `gold_raw`           | `STRING`     | Giá trị gốc của `gold` trước khi ép kiểu.                      |
| `lives_left_raw`     | `STRING`     | Giá trị gốc của `lives_left` trước khi ép kiểu.                |

---

## 9. Trường quan trọng

Các trường quan trọng để định danh và phân tích sự kiện level gồm:

```text id="v4vlat"
event_time_utc
user_pseudo_id
app_version
event_name
level_name
attempt_no
```

Trong đó:

* `event_time_utc` xác định thời điểm sự kiện;
* `user_pseudo_id` xác định người chơi;
* `app_version` xác định phiên bản game;
* `event_name` cho biết đây là bắt đầu hay kết thúc level;
* `level_name` xác định level;
* `attempt_no` giúp ghép sự kiện bắt đầu và kết thúc trong `vw_level_attempts`.

---

## 10. Logic xử lý dữ liệu

### 10.1 Lọc bảng GA4 theo hậu tố ngày hợp lệ

View chỉ đọc các bảng có hậu tố ngày đúng định dạng:

```sql id="h7f5hd"
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

Điều này giúp loại bỏ các bảng không phải bảng sự kiện theo ngày.

---

### 10.2 Lọc sự kiện level chính

View chỉ giữ lại hai loại sự kiện:

```sql id="n0q2nx"
event_name IN ('Start_level', 'End_level')
```

Các sự kiện khác không được đưa vào view này.

---

### 10.3 Trích xuất tham số từ GA4

View lấy các tham số sau từ `event_params`:

```text id="zf79h5"
level_name
attempt_no
success
fail_reason
move_used
move_left
duration_sec
gold
lives_left
poka_name
booster_used
booster_pre_used
```

Do GA4 có thể lưu một tham số ở nhiều kiểu dữ liệu khác nhau, view lấy giá trị theo thứ tự:

```text id="r1tifs"
string_value
int_value
float_value
double_value
```

Sau đó chuyển thành dạng chuỗi trung gian để xử lý tiếp.

---

### 10.4 Ép kiểu `attempt_no`

Cột gốc:

```text id="ohrvmv"
attempt_no_raw
```

được ép kiểu sang:

```text id="6k6s6n"
attempt_no
```

bằng kiểu dữ liệu `INT64`.

Nếu giá trị không ép kiểu được, kết quả sẽ là `NULL`.

---

### 10.5 Chuẩn hóa `success`

Cột gốc:

```text id="5tq593"
success_raw
```

được chuyển thành kiểu đúng/sai như sau:

| Giá trị gốc  | Giá trị sau chuẩn hóa |
| ------------ | --------------------- |
| `true`       | `TRUE`                |
| `1`          | `TRUE`                |
| `false`      | `FALSE`               |
| `0`          | `FALSE`               |
| giá trị khác | `NULL`                |

Việc chuẩn hóa này giúp các view phía sau có thể tính số lần thắng và số lần thua một cách nhất quán.

---

### 10.6 Ép kiểu `move_used`

Cột gốc:

```text id="oik952"
move_used_raw
```

được ép kiểu sang số nguyên:

```text id="0od8q1"
move_used
```

Nếu giá trị không hợp lệ, kết quả sẽ là `NULL`.

---

### 10.7 Xử lý `move_left`

Cột gốc:

```text id="6u1sj8"
move_left_raw
```

được xử lý đặc biệt:

```text id="3fmhdq"
Nếu move_left_raw = 'no_data' thì move_left = NULL
Nếu không, ép kiểu move_left_raw sang INT64
```

Điều này tránh việc giá trị văn bản `no_data` làm lỗi ép kiểu số.

---

### 10.8 Xử lý `duration_sec`

Cột gốc:

```text id="yyj6xl"
duration_sec_raw
```

được chuẩn hóa bằng cách thay dấu phẩy bằng dấu chấm trước khi ép kiểu sang số thực.

Ví dụ:

```text id="yd03py"
"1667,064267" → "1667.064267"
```

Sau đó ép kiểu sang:

```text id="vrczbv"
FLOAT64
```

Đây là xử lý quan trọng vì dữ liệu thực tế có thể chứa định dạng số dùng dấu phẩy làm dấu thập phân.

---

### 10.9 Giữ lại giá trị gốc

View giữ lại các cột gốc:

```text id="j08emp"
attempt_no_raw
success_raw
move_used_raw
move_left_raw
duration_sec_raw
gold_raw
lives_left_raw
```

Mục đích là để kiểm tra chất lượng dữ liệu.

Nếu cột đã ép kiểu bị `NULL`, người phân tích có thể xem cột gốc để biết nguyên nhân.

---

## 11. Cách hiểu `Start_level` và `End_level`

### 11.1 `Start_level`

`Start_level` là sự kiện ghi nhận người chơi bắt đầu một level.

Sự kiện này thường có các trường quan trọng:

```text id="l4tr0e"
level_name
attempt_no
gold
lives_left
booster_pre_used
```

Một số trường như `success`, `move_used`, `move_left`, `duration_sec` thường không có ý nghĩa ở `Start_level`.

---

### 11.2 `End_level`

`End_level` là sự kiện ghi nhận người chơi kết thúc một level.

Sự kiện này thường có các trường quan trọng:

```text id="xskai9"
level_name
attempt_no
success
fail_reason
move_used
move_left
duration_sec
booster_used
```

Các phân tích về thắng, thua, thời lượng chơi và số bước đi thường dựa vào `End_level`.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Kiểm tra số lượng sự kiện level theo ngày

```sql id="ycj63e"
SELECT
  event_date,
  app_version,
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

GROUP BY
  event_date,
  app_version,
  event_name

ORDER BY
  event_date,
  app_version,
  event_name;
```

Truy vấn này giúp theo dõi số lượng `Start_level` và `End_level` theo ngày.

---

### 12.2 Kiểm tra thiếu `level_name`

```sql id="eilrya"
SELECT
  event_date,
  app_version,
  event_name,
  COUNT(*) AS missing_level_name_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  level_name IS NULL

GROUP BY
  event_date,
  app_version,
  event_name

ORDER BY
  event_date,
  app_version,
  event_name;
```

Kỳ vọng:

```text id="ov9x1p"
Số lượng thiếu level_name nên bằng 0 hoặc rất thấp
```

Nếu `level_name` bị thiếu, các view phía sau không thể ghép đúng level.

---

### 12.3 Kiểm tra thiếu `attempt_no`

```sql id="bc7axb"
SELECT
  event_date,
  app_version,
  event_name,
  COUNT(*) AS missing_attempt_no_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  attempt_no IS NULL

GROUP BY
  event_date,
  app_version,
  event_name

ORDER BY
  event_date,
  app_version,
  event_name;
```

Nếu `attempt_no` thiếu nhiều, việc ghép `Start_level` và `End_level` trong `vw_level_attempts` có thể bị ảnh hưởng.

---

### 12.4 Kiểm tra chất lượng `End_level`

```sql id="w9pqq0"
SELECT
  event_date,
  app_version,

  COUNT(*) AS end_level_count,

  COUNTIF(success IS NULL)
    AS missing_success_count,

  COUNTIF(duration_sec IS NULL)
    AS missing_duration_sec_count,

  COUNTIF(move_used IS NULL)
    AS missing_move_used_count,

  COUNTIF(move_left IS NULL)
    AS null_move_left_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  event_name = 'End_level'

GROUP BY
  event_date,
  app_version

ORDER BY
  event_date,
  app_version;
```

Truy vấn này giúp kiểm tra các trường quan trọng nhất của sự kiện kết thúc level.

---

### 12.5 Kiểm tra giá trị `duration_sec_raw` có dấu phẩy

```sql id="zgmzh8"
SELECT
  event_date,
  app_version,
  level_name,
  duration_sec_raw,
  duration_sec

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  event_name = 'End_level'
  AND duration_sec_raw LIKE '%,%'

ORDER BY
  event_date,
  app_version,
  level_name

LIMIT 100;
```

Truy vấn này giúp kiểm tra việc chuyển dấu phẩy thành dấu chấm trong `duration_sec`.

---

### 12.6 Kiểm tra giá trị `success_raw` không hợp lệ

```sql id="943lsq"
SELECT
  success_raw,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  event_name = 'End_level'
  AND success IS NULL

GROUP BY
  success_raw

ORDER BY
  row_count DESC;
```

Nếu có nhiều giá trị lạ trong `success_raw`, cần kiểm tra tracking.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Đếm Start và End theo level

```sql id="o4pv2l"
SELECT
  app_version,
  level_name,

  COUNTIF(event_name = 'Start_level') AS start_level_count,
  COUNTIF(event_name = 'End_level') AS end_level_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

GROUP BY
  app_version,
  level_name

ORDER BY
  app_version,
  level_name;
```

### 13.2 Xem các `End_level` thắng/thua theo level

```sql id="a8s0u5"
SELECT
  app_version,
  level_name,

  COUNT(*) AS end_level_count,

  COUNTIF(success IS TRUE) AS win_count,
  COUNTIF(success IS FALSE) AS fail_count,

  ROUND(
    SAFE_DIVIDE(COUNTIF(success IS TRUE), COUNT(*)),
    4
  ) AS win_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  event_name = 'End_level'

GROUP BY
  app_version,
  level_name

ORDER BY
  app_version,
  level_name;
```

Lưu ý: truy vấn này chỉ dựa trên `End_level`, chưa kiểm tra việc ghép với `Start_level`. Để phân tích chính thức theo lượt chơi, nên dùng `vw_level_attempts`.

---

### 13.3 Kiểm tra thông tin booster

```sql id="6ir61s"
SELECT
  app_version,
  level_name,

  COUNT(*) AS end_level_count,

  COUNTIF(booster_used IS NOT NULL) AS booster_used_count,
  COUNTIF(booster_pre_used IS NOT NULL) AS booster_pre_used_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

WHERE
  event_name = 'End_level'

GROUP BY
  app_version,
  level_name

ORDER BY
  booster_used_count DESC;
```

---

## 14. Lưu ý khi sử dụng

### 14.1 View này chưa ghép lượt chơi

`vw_level_events` chỉ là dữ liệu sự kiện.

Một lần chơi level có thể gồm:

```text id="49re15"
Start_level
  ↓
End_level
```

Nhưng view này chưa ghép hai sự kiện đó thành một dòng.

Muốn phân tích lượt chơi level, dùng:

```text id="r9ekhh"
vw_level_attempts
```

---

### 14.2 Một số cột chỉ có ý nghĩa ở `End_level`

Các cột như sau chủ yếu có ý nghĩa ở `End_level`:

```text id="lpykd6"
success
fail_reason
move_used
move_left
duration_sec
booster_used
```

Nếu truy vấn cả `Start_level` và `End_level` cùng lúc, cần cẩn thận khi tính trung bình hoặc tỷ lệ.

---

### 14.3 Cột raw rất quan trọng cho kiểm tra tracking

Các cột raw không nên xóa khỏi view, vì chúng giúp kiểm tra nguyên nhân lỗi ép kiểu.

Ví dụ:

```text id="2px6fc"
duration_sec = NULL
```

chưa chắc nghĩa là game không gửi duration. Có thể game gửi giá trị lạ trong:

```text id="j1n4yh"
duration_sec_raw
```

---

### 14.4 View đọc toàn bộ bảng ngày hợp lệ

View đọc từ `events_*` và chỉ lọc hậu tố ngày hợp lệ, chưa giới hạn khoảng ngày cụ thể.

Khi dữ liệu lớn lên, truy vấn view này có thể tốn chi phí nếu được gọi nhiều lần.

Nếu chi phí tăng cao, có thể cân nhắc chuyển view này thành bảng được cập nhật theo lịch.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                                     | Ảnh hưởng                                                             |
| --------------------------------------------- | --------------------------------------------------------------------- |
| Lọc thiếu `Start_level`                       | Thiếu dữ liệu bắt đầu level.                                          |
| Lọc thiếu `End_level`                         | Thiếu dữ liệu kết thúc level, sai tỷ lệ thắng/thua.                   |
| Sai `attempt_no`                              | Ghép lượt chơi trong `vw_level_attempts` bị sai hoặc không ghép được. |
| Sai `success`                                 | Tỷ lệ thắng và tỷ lệ thua bị sai.                                     |
| Sai `duration_sec`                            | Phân tích thời lượng level bị sai.                                    |
| Không xử lý dấu phẩy trong `duration_sec_raw` | Nhiều dòng duration có thể bị `NULL`.                                 |
| Không xử lý `move_left = no_data`             | Có thể gây lỗi ép kiểu hoặc dữ liệu sai.                              |
| Xóa cột raw                                   | Khó điều tra lỗi tracking.                                            |

---

## 16. Mức độ quan trọng

```text id="eb8wrr"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là nguồn làm sạch chính cho `Start_level` và `End_level`;
* view này là đầu vào của `vw_level_attempts`;
* view này ảnh hưởng trực tiếp đến phân tích độ khó level;
* view này ảnh hưởng trực tiếp đến kiểm tra sức khỏe dữ liệu;
* nếu view này sai, nhiều view phía sau sẽ sai theo.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="ru4i2t"
sql/ddl/core_foundation/vw_level_events.sql
```

Cấu trúc logic chính của view:

```sql id="72nctg"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`
AS
WITH base AS (
  SELECT
    event_table_suffix,
    event_date,
    event_time_utc,
    user_pseudo_id,
    app_version,
    event_name,

    level_name,
    attempt_no_raw,
    success_raw,
    fail_reason,
    move_used_raw,
    move_left_raw,
    duration_sec_raw,
    gold_raw,
    lives_left_raw,
    poka_name,
    booster_used,
    booster_pre_used

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*`

  WHERE
    event_name IN ('Start_level', 'End_level')
)

SELECT
  *,
  SAFE_CAST(attempt_no_raw AS INT64) AS attempt_no,
  -- success_raw được chuẩn hóa thành TRUE / FALSE
  -- duration_sec_raw được thay dấu phẩy bằng dấu chấm rồi ép kiểu FLOAT64
FROM
  base;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Nguồn ngoài mart: `analytics_524104373.events_*`

- Upstream: Không có

- Downstream:
	- [[vw_level_tutorial_flag]]
	- [[vw_level_attempts]]
	- [[vw_f10_timing_24h]]
	- [[vw_pipeline_health_daily]]

