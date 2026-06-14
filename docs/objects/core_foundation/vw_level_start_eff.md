# vw_level_start_eff

## 1. Mục đích

`vw_level_start_eff` là view chuẩn hóa sự kiện bắt đầu level.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

Mục đích chính của `vw_level_start_eff` là tạo ra một nguồn dữ liệu thống nhất cho “lần bắt đầu level có hiệu lực”.

Nguồn này gồm hai loại:

1. sự kiện bắt đầu level thật từ `Start_level`;
2. sự kiện bắt đầu level được suy luận từ một số trường hợp `Tutorial_start`.

Lý do cần view này là vì trong dữ liệu game, một số thời điểm người chơi bắt đầu level có thể không được ghi nhận hoàn toàn bằng `Start_level`, mà được thể hiện qua sự kiện hướng dẫn. View này giúp các phân tích phía sau không bỏ sót những trường hợp đặc biệt đó.

---

## 2. Vai trò trong hệ thống

`vw_level_start_eff` thuộc nhóm:

```text id="w7uol2"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này là một trong các view đầu vào quan trọng nhất cho phân tích phễu người chơi mới và phễu 10 level đầu.

View này giúp chuẩn hóa khái niệm:

```text id="y2tyvy"
Người chơi đã bắt đầu một level
```

thành một định nghĩa rõ ràng và có thể tái sử dụng.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Người chơi bắt đầu level nào?
2. Thời điểm bắt đầu level là khi nào?
3. Sự kiện bắt đầu level đến từ `Start_level` thật hay được suy luận từ `Tutorial_start`?
4. Level bắt đầu thuộc phiên bản game nào?
5. Lần bắt đầu level có phải trường hợp giả lập từ hướng dẫn hay không?
6. Quy tắc nào được dùng để tạo lần bắt đầu level giả lập?
7. Có bao nhiêu lần bắt đầu level đến từ dữ liệu thật và bao nhiêu lần đến từ dữ liệu suy luận?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_start_eff` đại diện cho một lần bắt đầu level có hiệu lực của một người chơi.

Có thể hiểu đơn giản:

```text id="vvbksd"
Một dòng = một người chơi bắt đầu một level tại một thời điểm
```

View này không phải là dữ liệu theo ngày, không phải dữ liệu tổng hợp, và không phải dữ liệu theo người chơi duy nhất. Một người chơi có thể xuất hiện nhiều dòng nếu người đó bắt đầu nhiều level hoặc bắt đầu cùng một level nhiều lần.

---

## 5. Loại đối tượng

```text id="t13b6x"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính trực tiếp từ dữ liệu GA4 thô mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này đọc trực tiếp từ dữ liệu GA4 thô:

```text id="167ewx"
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Trong đó:

* `events_*` là các bảng sự kiện GA4 được chia theo ngày;
* `_TABLE_SUFFIX` là hậu tố ngày của bảng, ví dụ `20260531`;
* view chỉ lấy các bảng có hậu tố đúng dạng 8 chữ số.

View này chỉ lấy các sự kiện:

```text id="x21j8a"
Start_level
Tutorial_start
```

---

## 7. Các view đang sử dụng view này

`vw_level_start_eff` được sử dụng bởi nhiều view phân tích phía sau:

| View sử dụng               | Mục đích sử dụng                                                           |
| -------------------------- | -------------------------------------------------------------------------- |
| `vw_onboard_funnel_24h`    | Xác định người chơi đã bắt đầu Level 1 trong vòng 24 giờ sau `first_open`. |
| `vw_onboard_drop_diag_24h` | Chẩn đoán nhóm người chơi rơi trước hoặc sau khi bắt đầu Level 1.          |
| `vw_f10_funnel_24h`        | Đo phễu 10 level đầu dựa trên lần bắt đầu từng level.                      |
| `vw_f10_timing_24h`        | Đo thời gian từ Level 1 đến các level sau.                                 |
| `vw_f10_design_diag_24h`   | Hỗ trợ chẩn đoán thiết kế level dựa trên số người tới từng level.          |

Do được nhiều view quan trọng sử dụng, mọi thay đổi trong logic của `vw_level_start_eff` có thể ảnh hưởng lớn đến kết quả phân tích phía sau.

---

## 8. Danh sách cột

| Cột                        | Kiểu dữ liệu | Ý nghĩa                                                                                                     |
| -------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| `effective_start_event_id` | `STRING`     | Mã định danh ổn định cho một lần bắt đầu level có hiệu lực. Được tạo bằng hàm băm từ các trường quan trọng. |
| `event_table_suffix`       | `STRING`     | Hậu tố ngày của bảng GA4 gốc, ví dụ `20260531`.                                                             |
| `event_date`               | `STRING`     | Ngày sự kiện theo định dạng của GA4.                                                                        |
| `start_time_utc`           | `TIMESTAMP`  | Thời điểm bắt đầu level theo múi giờ UTC.                                                                   |
| `user_pseudo_id`           | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                                                              |
| `app_version`              | `STRING`     | Phiên bản game tại thời điểm sự kiện xảy ra.                                                                |
| `level_name`               | `STRING`     | Tên level đã được chuẩn hóa để phân tích, ví dụ `N001`, `N010`.                                             |
| `attempt_no`               | `INT64`      | Số lần thử level. Với các dòng suy luận từ hướng dẫn, trường này có thể bị trống.                           |
| `source_event_name`        | `STRING`     | Tên sự kiện gốc dùng để tạo dòng dữ liệu. Có thể là `Start_level` hoặc `Tutorial_start`.                    |
| `raw_level_name`           | `STRING`     | Giá trị level gốc lấy từ tham số `level_name` trong GA4.                                                    |
| `tutorial_name`            | `STRING`     | Tên hướng dẫn nếu dòng được tạo từ sự kiện hướng dẫn.                                                       |
| `is_pseudo_start`          | `BOOL`       | Cho biết đây có phải là lần bắt đầu level được suy luận hay không.                                          |
| `effective_start_source`   | `STRING`     | Nguồn tạo ra lần bắt đầu level có hiệu lực.                                                                 |
| `pseudo_start_rule`        | `STRING`     | Tên quy tắc dùng để tạo dòng bắt đầu level giả lập. Chỉ có giá trị với các dòng được suy luận từ hướng dẫn. |

---

## 9. Trường định danh quan trọng

Trường định danh chính của view là:

```text id="c36n7r"
effective_start_event_id
```

Trường này được tạo bằng cách băm các thông tin chính của sự kiện.

Trong tài liệu này, “băm” nghĩa là biến một chuỗi thông tin đầu vào thành một mã định danh dạng chuỗi. Mã này giúp nhận diện một dòng dữ liệu mà không cần có sẵn mã sự kiện gốc từ GA4.

Với sự kiện `Start_level` thật, mã này được tạo từ:

```text id="ctzk9f"
user_pseudo_id
start_time_utc
app_version
Start_level
raw_level_name
attempt_no
```

Với sự kiện bắt đầu level được suy luận từ `Tutorial_start`, mã này được tạo từ:

```text id="bt9d19"
user_pseudo_id
start_time_utc
app_version
Tutorial_start_as_Start_level
raw_level_name
tutorial_name
level_name được suy luận
```

---

## 10. Logic xử lý dữ liệu

### 10.1 Lọc dữ liệu đầu vào

View chỉ đọc các bảng GA4 có hậu tố ngày hợp lệ:

```sql id="10rj9n"
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

View cũng yêu cầu:

```sql id="2cvps6"
user_pseudo_id IS NOT NULL
```

Điều này đảm bảo mỗi dòng dữ liệu có thể gắn với một người chơi ẩn danh.

View chỉ lấy hai loại sự kiện:

```sql id="28bfku"
event_name IN (
  'Start_level',
  'Tutorial_start'
)
```

---

### 10.2 Trích xuất tham số từ GA4

View trích xuất các tham số sau từ `event_params`:

| Tham số GA4  | Cột trung gian      | Ý nghĩa         |
| ------------ | ------------------- | --------------- |
| `level_name` | `level_name_raw`    | Tên level gốc.  |
| `attempt_no` | `attempt_no_raw`    | Số lần thử gốc. |
| `name`       | `tutorial_name_raw` | Tên hướng dẫn.  |

Do GA4 có thể lưu giá trị ở nhiều kiểu dữ liệu khác nhau, view dùng logic lấy giá trị theo thứ tự:

```text id="a08mvk"
string_value
int_value
float_value
double_value
```

Sau đó ép về dạng chuỗi để xử lý tiếp.

---

### 10.3 Làm sạch tên level

View làm sạch `level_name_raw` bằng cách:

* loại bỏ khoảng trắng đầu và cuối;
* bỏ hậu tố `.0` nếu có.

Ví dụ:

```text id="tj8xli"
"1.0"  → "1"
"N001" → "N001"
```

Cột sau khi làm sạch được đặt là:

```text id="qij3g2"
raw_level_name
```

---

### 10.4 Làm sạch số lần thử

View làm sạch `attempt_no_raw` bằng cách:

* loại bỏ khoảng trắng;
* bỏ hậu tố `.0` nếu có;
* ép kiểu sang số nguyên.

Ví dụ:

```text id="m3n9bi"
"1.0" → 1
"2"   → 2
```

Nếu không thể ép kiểu, giá trị sẽ thành `NULL`.

---

### 10.5 Tạo dòng từ sự kiện `Start_level`

Với sự kiện `Start_level`, view tạo dòng bắt đầu level thật.

Điều kiện:

```sql id="c8yson"
event_name = 'Start_level'
AND raw_level_name IS NOT NULL
```

Các dòng này có:

```text id="h8oh4n"
is_pseudo_start = FALSE
effective_start_source = raw_start_level
pseudo_start_rule = NULL
```

Trong trường hợp này:

```text id="6cczuq"
level_name = raw_level_name
```

---

### 10.6 Tạo dòng suy luận từ `Tutorial_start`

Một số dòng được tạo từ `Tutorial_start` để bù cho các trường hợp bắt đầu level không có `Start_level` rõ ràng.

Hiện tại view chỉ suy luận trong phiên bản:

```text id="z4rxu9"
app_version = 0.3.61
```

Có hai quy tắc đang được dùng:

| Điều kiện                                             | Level suy luận | Quy tắc                                          |
| ----------------------------------------------------- | -------------- | ------------------------------------------------ |
| `raw_level_name = '1'` và `tutorial_name = 'Match'`   | `N001`         | `0.3.61__tutorial_start_level_1_match_to_N001`   |
| `raw_level_name = '10'` và `tutorial_name = 'Nectar'` | `N010`         | `0.3.61__tutorial_start_level_10_nectar_to_N010` |

Các dòng này có:

```text id="bm5x7j"
is_pseudo_start = TRUE
effective_start_source = pseudo_tutorial_start
attempt_no = NULL
source_event_name = Tutorial_start
```

---

## 11. Cách hiểu các loại nguồn bắt đầu level

Cột `effective_start_source` hiện có hai nhóm giá trị chính:

| Giá trị                 | Ý nghĩa                                           |
| ----------------------- | ------------------------------------------------- |
| `raw_start_level`       | Dòng được tạo trực tiếp từ sự kiện `Start_level`. |
| `pseudo_tutorial_start` | Dòng được suy luận từ sự kiện `Tutorial_start`.   |

Cột `is_pseudo_start` giúp phân biệt nhanh:

```text id="pdu8sq"
FALSE = bắt đầu level thật từ Start_level
TRUE  = bắt đầu level được suy luận từ Tutorial_start
```

---

## 12. Lưu ý về dữ liệu

### 12.1 `attempt_no` có thể trống

Với các dòng được suy luận từ `Tutorial_start`, `attempt_no` được đặt là `NULL`.

Điều này cần được lưu ý khi nối với các bảng cần số lần thử level.

### 12.2 Logic suy luận hiện đang giới hạn cho một phiên bản

Các quy tắc suy luận hiện chỉ áp dụng cho:

```text id="lac3qv"
app_version = 0.3.61
```

Nếu phiên bản mới có logic hướng dẫn khác, cần cập nhật view hoặc tạo bảng cấu hình riêng để tránh hard-code quá nhiều.

Trong tài liệu này, “hard-code” nghĩa là viết trực tiếp giá trị cố định vào câu truy vấn, thay vì lấy từ bảng cấu hình.

### 12.3 View đọc toàn bộ bảng ngày hợp lệ

View đọc từ `events_*` và chỉ lọc hậu tố ngày hợp lệ, chưa giới hạn khoảng ngày cụ thể.

Khi dữ liệu lớn lên, truy vấn view này có thể tốn chi phí nếu được gọi thường xuyên từ nhiều view khác.

Nếu chi phí tăng cao, nên cân nhắc chuyển view này thành bảng được cập nhật theo lịch.

---

## 13. Kiểm tra chất lượng dữ liệu khuyến nghị

### 13.1 Kiểm tra số lượng dòng theo nguồn bắt đầu level

```sql id="in0zt3"
SELECT
  app_version,
  effective_start_source,
  is_pseudo_start,
  COUNT(*) AS start_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

GROUP BY
  app_version,
  effective_start_source,
  is_pseudo_start

ORDER BY
  app_version,
  effective_start_source;
```

Truy vấn này giúp kiểm tra số lượng dòng bắt đầu level thật và dòng bắt đầu level được suy luận.

---

### 13.2 Kiểm tra dòng bắt đầu level được suy luận

```sql id="mnpbez"
SELECT
  app_version,
  raw_level_name,
  tutorial_name,
  level_name,
  pseudo_start_rule,
  COUNT(*) AS pseudo_start_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

WHERE
  is_pseudo_start IS TRUE

GROUP BY
  app_version,
  raw_level_name,
  tutorial_name,
  level_name,
  pseudo_start_rule

ORDER BY
  pseudo_start_count DESC;
```

Kỳ vọng hiện tại là chỉ có các quy tắc đã định nghĩa rõ.

---

### 13.3 Kiểm tra trùng `effective_start_event_id`

```sql id="sfxvwu"
SELECT
  effective_start_event_id,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

GROUP BY
  effective_start_event_id

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="zexyvc"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là mã định danh không đủ duy nhất hoặc có dữ liệu trùng ở nguồn.

---

### 13.4 Kiểm tra level bị thiếu

```sql id="6eiem2"
SELECT
  source_event_name,
  effective_start_source,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

WHERE
  level_name IS NULL

GROUP BY
  source_event_name,
  effective_start_source

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="pu4zhb"
Không có dòng level_name bị thiếu
```

---

## 14. Cách sử dụng khuyến nghị

### 14.1 Đếm số lượt bắt đầu level theo ngày và level

```sql id="mu0lxo"
SELECT
  event_date,
  app_version,
  level_name,
  COUNT(*) AS effective_start_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

GROUP BY
  event_date,
  app_version,
  level_name

ORDER BY
  event_date,
  app_version,
  level_name;
```

### 14.2 Tách bắt đầu level thật và bắt đầu level suy luận

```sql id="r1pjvx"
SELECT
  event_date,
  app_version,
  level_name,

  COUNTIF(is_pseudo_start IS FALSE) AS raw_start_count,
  COUNTIF(is_pseudo_start IS TRUE) AS pseudo_start_count,
  COUNT(*) AS total_effective_start_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

GROUP BY
  event_date,
  app_version,
  level_name

ORDER BY
  event_date,
  app_version,
  level_name;
```

### 14.3 Lấy lần bắt đầu Level 1 đầu tiên của mỗi người chơi

```sql id="wfhb36"
SELECT
  app_version,
  user_pseudo_id,
  MIN(start_time_utc) AS first_level1_start_time_utc

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`

WHERE
  level_name = 'N001'

GROUP BY
  app_version,
  user_pseudo_id;
```

Lưu ý: truy vấn này chỉ đúng nếu `N001` là Level 1 trong phiên bản game cần phân tích. Trong các view chính, Level 1 nên được lấy từ `dim_f10_level_map` thay vì hard-code `N001`.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                                 | Ảnh hưởng                                                        |
| ----------------------------------------- | ---------------------------------------------------------------- |
| Bỏ sót `Start_level` thật                 | Phễu level sẽ bị thiếu người chơi.                               |
| Suy luận sai `Tutorial_start` thành level | Có thể tính nhầm người chơi đã bắt đầu level.                    |
| Sai `level_name`                          | Các view phía sau không nối đúng với mapping level.              |
| Sai `effective_start_event_id`            | Có thể gây trùng hoặc mất khả năng nhận diện dòng bắt đầu level. |
| Không kiểm soát `is_pseudo_start`         | Khó phân biệt dữ liệu thật và dữ liệu suy luận.                  |
| Hard-code quy tắc suy luận quá nhiều      | Dễ sai khi app version mới thay đổi logic hướng dẫn.             |

---

## 16. Mức độ quan trọng

```text id="x57ysr"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là nguồn bắt đầu level chuẩn cho nhiều phân tích chính;
* view này ảnh hưởng trực tiếp đến phễu người chơi mới;
* view này ảnh hưởng trực tiếp đến phễu 10 level đầu;
* view này có logic suy luận từ tutorial, nếu sai sẽ làm sai số lượng người chơi bắt đầu level;
* view này là nền tảng cho nhiều view phía sau.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="nyobrs"
sql/ddl/core_foundation/vw_level_start_eff.sql
```

Cấu trúc logic chính của view:

```sql id="i8jyz9"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_start_eff`
AS
WITH base AS (...),
cleaned AS (...),
raw_start_level_events AS (...),
pseudo_start_level_events AS (...)

SELECT ... FROM raw_start_level_events

UNION ALL

SELECT ... FROM pseudo_start_level_events;
```

View gồm hai nhánh dữ liệu:

```text id="h4wzmf"
1. raw_start_level_events
   - tạo từ Start_level thật

2. pseudo_start_level_events
   - tạo từ một số Tutorial_start được suy luận thành bắt đầu level
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Nguồn ngoài mart: `analytics_524104373.events_*`

- Upstream: Không có

- Downstream:
	- [[vw_onboard_funnel_24h]]
	- [[vw_onboard_drop_diag_24h]]
	- [[vw_f10_funnel_24h]]
	- [[vw_f10_timing_24h]]
	- [[vw_f10_design_diag_24h]]
