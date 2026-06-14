# vw_tutorial_events

## 1. Mục đích

`vw_tutorial_events` là view chuẩn hóa các sự kiện hướng dẫn trong game.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này lấy ba loại sự kiện hướng dẫn:

```text id="l45sa8"
Tutorial_start
Tutorial_step_start
Tutorial_end
```

Mục đích chính là tạo một nguồn dữ liệu riêng cho các sự kiện hướng dẫn, để các view phía sau có thể:

* biết người chơi đang đi qua hướng dẫn nào;
* biết hướng dẫn xảy ra gần level nào;
* đánh dấu các lần bắt đầu level bị ảnh hưởng bởi hướng dẫn;
* kiểm tra chất lượng tracking của các sự kiện hướng dẫn.

---

## 2. Vai trò trong hệ thống

`vw_tutorial_events` thuộc nhóm:

```text id="f951qc"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này không phải là view phân tích kết quả cuối cùng. Nó là view làm sạch dữ liệu đầu vào cho nhóm sự kiện hướng dẫn.

Vai trò quan trọng nhất của view này là cung cấp dữ liệu cho:

```text id="7tv0fv"
vw_level_tutorial_flag
```

`vw_level_tutorial_flag` dùng dữ liệu từ `vw_tutorial_events` để đánh dấu các lần bắt đầu level xảy ra gần hướng dẫn.

Điều này giúp tách riêng:

* dữ liệu chơi level bình thường;
* dữ liệu chơi level có thể bị ảnh hưởng bởi hướng dẫn.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu sự kiện `Tutorial_start` theo ngày?
2. Có bao nhiêu sự kiện `Tutorial_step_start` theo ngày?
3. Có bao nhiêu sự kiện `Tutorial_end` theo ngày?
4. Hướng dẫn nào xuất hiện nhiều nhất?
5. Hướng dẫn nào gắn với level nào?
6. Có sự kiện hướng dẫn nào thiếu `level_name` không?
7. Có sự kiện hướng dẫn nào thiếu `tutorial_name` không?
8. Có sự kiện hướng dẫn nào thiếu `tutorial_value` không?
9. Có level nào bị ảnh hưởng nhiều bởi hướng dẫn không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_tutorial_events` đại diện cho một sự kiện hướng dẫn từ GA4.

Có thể hiểu đơn giản:

```text id="uzq2by"
Một dòng = một sự kiện Tutorial_start, Tutorial_step_start hoặc Tutorial_end của một người chơi tại một thời điểm
```

Một người chơi có thể có nhiều dòng trong view này nếu người đó đi qua nhiều bước hướng dẫn.

---

## 5. Loại đối tượng

```text id="2mgtjc"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính trực tiếp từ dữ liệu GA4 thô mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này đọc trực tiếp từ dữ liệu GA4 thô:

```text id="6wdui8"
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Trong đó:

* `events_*` là các bảng sự kiện GA4 được chia theo ngày;
* `_TABLE_SUFFIX` là hậu tố ngày của bảng, ví dụ `20260531`;
* view chỉ lấy các bảng có hậu tố đúng dạng 8 chữ số;
* view chỉ lấy các sự kiện liên quan đến hướng dẫn.

---

## 7. Các view đang sử dụng view này

`vw_tutorial_events` được sử dụng bởi các view sau:

| View sử dụng               | Mục đích sử dụng                                                      |
| -------------------------- | --------------------------------------------------------------------- |
| `vw_level_tutorial_flag`   | Đánh dấu các lần bắt đầu level xảy ra gần sự kiện bắt đầu hướng dẫn.  |
| `vw_pipeline_health_daily` | Kiểm tra số lượng sự kiện hướng dẫn và các trường bị thiếu theo ngày. |

Do view này là nguồn chính cho dữ liệu hướng dẫn, nếu view lọc sai sự kiện hoặc thiếu trường, các phân tích về ảnh hưởng của hướng dẫn sẽ bị sai.

---

## 8. Danh sách cột

| Cột                  | Kiểu dữ liệu | Ý nghĩa                                                        |
| -------------------- | ------------ | -------------------------------------------------------------- |
| `event_table_suffix` | `STRING`     | Hậu tố ngày của bảng GA4 gốc, ví dụ `20260531`.                |
| `event_date`         | `STRING`     | Ngày sự kiện theo định dạng của GA4.                           |
| `event_time_utc`     | `TIMESTAMP`  | Thời điểm xảy ra sự kiện theo múi giờ UTC.                     |
| `user_pseudo_id`     | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                 |
| `event_name`         | `STRING`     | Tên sự kiện hướng dẫn, ví dụ `Tutorial_start`.                 |
| `level_name`         | `STRING`     | Tên level gắn với sự kiện hướng dẫn.                           |
| `tutorial_name`      | `STRING`     | Tên hướng dẫn, lấy từ tham số `name` trong GA4.                |
| `tutorial_value`     | `STRING`     | Giá trị hoặc bước hướng dẫn, lấy từ tham số `value` trong GA4. |

---

## 9. Trường quan trọng

Các trường quan trọng nhất trong view này gồm:

```text id="7jp6zs"
event_time_utc
user_pseudo_id
event_name
level_name
tutorial_name
tutorial_value
```

Trong đó:

* `event_time_utc` dùng để so sánh thời gian giữa hướng dẫn và bắt đầu level;
* `user_pseudo_id` dùng để nối sự kiện hướng dẫn với sự kiện level của cùng người chơi;
* `event_name` cho biết loại sự kiện hướng dẫn;
* `level_name` giúp xác định hướng dẫn đang liên quan đến level nào;
* `tutorial_name` giúp xác định loại hướng dẫn;
* `tutorial_value` giúp xác định bước hoặc giá trị chi tiết của hướng dẫn.

---

## 10. Logic xử lý dữ liệu

### 10.1 Lọc bảng GA4 theo hậu tố ngày hợp lệ

View chỉ đọc các bảng có hậu tố ngày đúng định dạng:

```sql id="lwnsml"
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

Điều này giúp loại bỏ các bảng không phải bảng sự kiện theo ngày.

---

### 10.2 Lọc sự kiện hướng dẫn

View chỉ giữ lại ba loại sự kiện:

```sql id="mp648c"
event_name IN (
  'Tutorial_start',
  'Tutorial_step_start',
  'Tutorial_end'
)
```

Các sự kiện khác không được đưa vào view này.

Tên sự kiện có phân biệt chữ hoa và chữ thường, nên khi kiểm tra tracking cần dùng đúng tên.

---

### 10.3 Trích xuất `level_name`

View lấy tham số:

```text id="2u084b"
level_name
```

từ `event_params` của GA4.

Cột này được đưa ra dưới tên:

```text id="7g2sws"
level_name
```

Đây là level liên quan đến sự kiện hướng dẫn.

---

### 10.4 Trích xuất `tutorial_name`

View lấy tham số:

```text id="q7umnt"
name
```

từ `event_params` của GA4.

Cột này được đưa ra dưới tên:

```text id="biopsf"
tutorial_name
```

Ví dụ giá trị có thể là tên của một loại hướng dẫn như `Match`, `Nectar`, `Box`, hoặc các tên hướng dẫn khác tùy tracking.

---

### 10.5 Trích xuất `tutorial_value`

View lấy tham số:

```text id="a92zko"
value
```

từ `event_params` của GA4.

Cột này được đưa ra dưới tên:

```text id="td5j5d"
tutorial_value
```

Cột này có thể dùng để mô tả bước hướng dẫn hoặc giá trị chi tiết của sự kiện hướng dẫn.

---

### 10.6 Cách lấy giá trị từ GA4

Với mỗi tham số, view lấy giá trị theo thứ tự:

```text id="w3js4h"
string_value
int_value
float_value
double_value
```

Sau đó chuyển về dạng chuỗi.

Cách này giúp view xử lý được trường hợp cùng một tham số nhưng GA4 ghi nhận ở nhiều kiểu dữ liệu khác nhau.

---

## 11. Lưu ý về trường `app_version`

`vw_tutorial_events` hiện không có cột:

```text id="lf1jbc"
app_version
```

Điều này có nghĩa là khi phân tích sự kiện hướng dẫn theo phiên bản game, không thể dùng trực tiếp view này nếu không nối thêm dữ liệu khác.

Trong các trường hợp cần phân tích hướng dẫn theo phiên bản, cần cân nhắc:

* thêm `app_version` vào view trong tương lai;
* hoặc nối với dữ liệu level gần nhất nếu có quy tắc hợp lý;
* hoặc dùng view khác đã có `app_version` nếu phù hợp.

Đây là điểm cần lưu ý khi mở rộng phân tích tutorial.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Kiểm tra số lượng sự kiện hướng dẫn theo ngày

```sql id="u7xvng"
SELECT
  event_date,
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

GROUP BY
  event_date,
  event_name

ORDER BY
  event_date,
  event_name;
```

Truy vấn này giúp theo dõi số lượng từng loại sự kiện hướng dẫn theo ngày.

---

### 12.2 Kiểm tra số lượng theo tên hướng dẫn

```sql id="jrgwxk"
SELECT
  event_date,
  tutorial_name,
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

GROUP BY
  event_date,
  tutorial_name,
  event_name

ORDER BY
  event_date,
  tutorial_name,
  event_name;
```

Truy vấn này giúp xem hướng dẫn nào xuất hiện nhiều hoặc ít bất thường.

---

### 12.3 Kiểm tra thiếu `level_name`

```sql id="3wi40i"
SELECT
  event_date,
  event_name,
  tutorial_name,
  COUNT(*) AS missing_level_name_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

WHERE
  level_name IS NULL

GROUP BY
  event_date,
  event_name,
  tutorial_name

ORDER BY
  event_date,
  event_name,
  tutorial_name;
```

Nếu `level_name` thiếu nhiều, việc xác định hướng dẫn liên quan đến level nào sẽ bị giảm độ tin cậy.

---

### 12.4 Kiểm tra thiếu `tutorial_name`

```sql id="48xr3j"
SELECT
  event_date,
  event_name,
  level_name,
  COUNT(*) AS missing_tutorial_name_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

WHERE
  tutorial_name IS NULL

GROUP BY
  event_date,
  event_name,
  level_name

ORDER BY
  event_date,
  event_name,
  level_name;
```

Nếu `tutorial_name` thiếu nhiều, sẽ khó biết người chơi đang đi qua loại hướng dẫn nào.

---

### 12.5 Kiểm tra thiếu `tutorial_value`

```sql id="4x0i6g"
SELECT
  event_date,
  event_name,
  tutorial_name,
  level_name,
  COUNT(*) AS missing_tutorial_value_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

WHERE
  tutorial_value IS NULL

GROUP BY
  event_date,
  event_name,
  tutorial_name,
  level_name

ORDER BY
  event_date,
  event_name,
  tutorial_name,
  level_name;
```

Kết quả của truy vấn này cần được đọc theo từng loại sự kiện. Không phải mọi loại tutorial event đều bắt buộc có `tutorial_value`.

---

### 12.6 Kiểm tra người chơi có `Tutorial_start` nhưng không có `Tutorial_end`

```sql id="j7zxf3"
WITH tutorial_start AS (
  SELECT
    user_pseudo_id,
    level_name,
    tutorial_name,
    MIN(event_time_utc) AS tutorial_start_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

  WHERE
    event_name = 'Tutorial_start'

  GROUP BY
    user_pseudo_id,
    level_name,
    tutorial_name
),

tutorial_end AS (
  SELECT
    user_pseudo_id,
    level_name,
    tutorial_name,
    MIN(event_time_utc) AS tutorial_end_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

  WHERE
    event_name = 'Tutorial_end'

  GROUP BY
    user_pseudo_id,
    level_name,
    tutorial_name
)

SELECT
  tutorial_start.level_name,
  tutorial_start.tutorial_name,

  COUNT(DISTINCT tutorial_start.user_pseudo_id)
    AS tutorial_start_user_count,

  COUNT(DISTINCT tutorial_end.user_pseudo_id)
    AS tutorial_end_user_count,

  COUNT(DISTINCT IF(
    tutorial_end.user_pseudo_id IS NULL,
    tutorial_start.user_pseudo_id,
    NULL
  )) AS tutorial_start_without_end_user_count

FROM
  tutorial_start

LEFT JOIN
  tutorial_end
ON
  tutorial_start.user_pseudo_id = tutorial_end.user_pseudo_id
  AND tutorial_start.level_name = tutorial_end.level_name
  AND tutorial_start.tutorial_name = tutorial_end.tutorial_name
  AND tutorial_end.tutorial_end_time_utc >= tutorial_start.tutorial_start_time_utc

GROUP BY
  tutorial_start.level_name,
  tutorial_start.tutorial_name

ORDER BY
  tutorial_start_without_end_user_count DESC;
```

Truy vấn này giúp phát hiện các hướng dẫn bắt đầu nhưng không có sự kiện kết thúc tương ứng.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Xem các hướng dẫn xuất hiện trong dữ liệu

```sql id="qet4fh"
SELECT
  tutorial_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count,
  MIN(event_date) AS first_seen_event_date,
  MAX(event_date) AS last_seen_event_date

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

GROUP BY
  tutorial_name

ORDER BY
  event_count DESC;
```

---

### 13.2 Xem hướng dẫn theo level

```sql id="h6hav4"
SELECT
  level_name,
  tutorial_name,
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

GROUP BY
  level_name,
  tutorial_name,
  event_name

ORDER BY
  level_name,
  tutorial_name,
  event_name;
```

---

### 13.3 Lấy thời điểm bắt đầu hướng dẫn đầu tiên của mỗi người chơi theo level

```sql id="703nv4"
SELECT
  user_pseudo_id,
  level_name,
  tutorial_name,
  MIN(event_time_utc) AS first_tutorial_start_time_utc

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

WHERE
  event_name = 'Tutorial_start'

GROUP BY
  user_pseudo_id,
  level_name,
  tutorial_name;
```

Truy vấn này có thể dùng làm nền tảng để so sánh với thời điểm bắt đầu level.

---

## 14. Lưu ý khi sử dụng

### 14.1 Đây là view sự kiện, chưa phải view phân tích hoàn chỉnh

`vw_tutorial_events` chỉ chuẩn hóa sự kiện hướng dẫn.

Nếu muốn biết một lần bắt đầu level có bị ảnh hưởng bởi hướng dẫn hay không, nên dùng:

```text id="k7iqqg"
vw_level_tutorial_flag
```

---

### 14.2 Không nên tự suy luận app version từ view này

View hiện không có `app_version`.

Nếu cần phân tích theo phiên bản game, nên bổ sung logic rõ ràng thay vì tự suy luận không kiểm chứng.

---

### 14.3 `tutorial_value` không phải lúc nào cũng bắt buộc

Một số loại sự kiện hướng dẫn có thể không cần `tutorial_value`.

Vì vậy, khi thấy `tutorial_value IS NULL`, cần đọc theo từng `event_name` và từng `tutorial_name`, không nên kết luận ngay là lỗi tracking.

---

### 14.4 Tên hướng dẫn cần được quản lý nhất quán

Nếu cùng một hướng dẫn nhưng được gửi lên với nhiều tên khác nhau, ví dụ viết hoa khác nhau hoặc sai chính tả, phân tích sẽ bị phân mảnh.

Nên có quy ước đặt tên rõ cho `tutorial_name`.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                                    | Ảnh hưởng                                                |
| -------------------------------------------- | -------------------------------------------------------- |
| Lọc thiếu `Tutorial_start`                   | Không đánh dấu đúng các lần bắt đầu level gần hướng dẫn. |
| Lọc thiếu `Tutorial_end`                     | Khó biết người chơi có hoàn tất hướng dẫn hay không.     |
| Sai `level_name`                             | Không biết hướng dẫn gắn với level nào.                  |
| Sai `tutorial_name`                          | Không phân biệt được các loại hướng dẫn.                 |
| Thiếu `tutorial_value` nhưng không phát hiện | Có thể mất thông tin về bước hướng dẫn.                  |
| Không có `app_version`                       | Hạn chế phân tích hướng dẫn theo phiên bản game.         |
| Tên hướng dẫn không nhất quán                | Chỉ số theo từng hướng dẫn bị tách thành nhiều nhóm nhỏ. |

---

## 16. Mức độ quan trọng

```text id="7rmy02"
Mức độ quan trọng: Cao
```

Lý do:

* view này là nguồn dữ liệu hướng dẫn chuẩn;
* view này ảnh hưởng trực tiếp đến `vw_level_tutorial_flag`;
* view này giúp phân biệt level chơi bình thường và level bị ảnh hưởng bởi hướng dẫn;
* nếu view này sai, các phân tích độ khó level có thể bị nhiễu bởi tutorial.

Trong tài liệu này, “bị nhiễu bởi tutorial” nghĩa là dữ liệu chơi level không còn phản ánh hành vi chơi tự nhiên, vì người chơi đang được hướng dẫn hoặc điều khiển bởi luồng hướng dẫn.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="jqtx2p"
sql/ddl/core_foundation/vw_tutorial_events.sql
```

Cấu trúc logic chính của view:

```sql id="al1lbk"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`
AS
WITH tutorial_events_raw AS (
  SELECT
    _TABLE_SUFFIX AS event_table_suffix,
    event_date,
    TIMESTAMP_MICROS(event_timestamp) AS event_time_utc,
    user_pseudo_id,
    event_name,

    -- Lấy tham số level_name
    -- Lấy tham số name làm tutorial_name
    -- Lấy tham số value làm tutorial_value

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*`

  WHERE
    REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
    AND event_name IN (
      'Tutorial_start',
      'Tutorial_step_start',
      'Tutorial_end'
    )
)

SELECT
  event_table_suffix,
  event_date,
  event_time_utc,
  user_pseudo_id,
  event_name,
  level_name_raw AS level_name,
  tutorial_name_raw AS tutorial_name,
  tutorial_value_raw AS tutorial_value

FROM
  tutorial_events_raw;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Nguồn ngoài mart: `analytics_524104373.events_*`

- Upstream: Không có

- Downstream:
	- [[vw_level_tutorial_flag]]
	- [[vw_pipeline_health_daily]]

