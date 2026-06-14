# vw_level_tutorial_flag

## 1. Mục đích

`vw_level_tutorial_flag` là view đánh dấu các lần bắt đầu level có xảy ra gần sự kiện bắt đầu hướng dẫn.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

Mục đích chính của view này là xác định xem một sự kiện `Start_level` có bị ảnh hưởng bởi tutorial hay không.

Trong tài liệu này, “tutorial” được hiểu là phần hướng dẫn trong game. Các sự kiện liên quan đến tutorial được lấy từ `vw_tutorial_events`.

Nếu một `Start_level` xảy ra gần `Tutorial_start`, view sẽ đánh dấu:

```text id="yl62c8"
is_near_tutorial_start = TRUE
```

Điều này giúp các phân tích phía sau tách riêng:

* lượt chơi level bình thường;
* lượt chơi level có thể bị ảnh hưởng bởi hướng dẫn.

---

## 2. Vai trò trong hệ thống

`vw_level_tutorial_flag` thuộc nhóm:

```text id="1qqbu2"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này nằm giữa dữ liệu sự kiện thô đã làm sạch và dữ liệu lượt chơi level.

Luồng phụ thuộc chính:

```text id="6hh2ju"
vw_level_events
vw_tutorial_events
  └── vw_level_tutorial_flag
        └── vw_level_attempts
```

Nói cách khác:

* `vw_level_events` cung cấp sự kiện `Start_level`;
* `vw_tutorial_events` cung cấp sự kiện `Tutorial_start`;
* `vw_level_tutorial_flag` đánh dấu `Start_level` nào gần `Tutorial_start`;
* `vw_level_attempts` dùng kết quả này để đưa cờ tutorial vào dữ liệu lượt chơi level.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu lần bắt đầu level xảy ra gần tutorial?
2. Level nào có tỷ lệ bắt đầu gần tutorial cao?
3. Những tutorial nào xuất hiện gần thời điểm bắt đầu level?
4. Dữ liệu độ khó level có bị nhiễu bởi tutorial không?
5. Có nên loại trừ một số lượt chơi khỏi phân tích độ khó thông thường không?
6. Tỷ lệ bắt đầu level gần tutorial thay đổi theo ngày như thế nào?
7. Dữ liệu tutorial có được gửi đúng thời điểm gần `Start_level` không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_tutorial_flag` đại diện cho một sự kiện `Start_level` có `level_name` hợp lệ.

Có thể hiểu đơn giản:

```text id="ih0l0d"
Một dòng = một lần người chơi bắt đầu level, kèm cờ cho biết có gần Tutorial_start hay không
```

View này chỉ xét `Start_level` từ `vw_level_events`.

View này chưa ghép `Start_level` với `End_level`. Việc ghép bắt đầu và kết thúc level được thực hiện ở:

```text id="zewtlz"
vw_level_attempts
```

---

## 5. Loại đối tượng

```text id="xwi1yz"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính từ các view nguồn mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào hai view nguồn:

| Nguồn                | Vai trò                                |
| -------------------- | -------------------------------------- |
| `vw_level_events`    | Cung cấp các sự kiện `Start_level`.    |
| `vw_tutorial_events` | Cung cấp các sự kiện `Tutorial_start`. |

View chỉ lấy từ `vw_level_events` các dòng:

```sql id="0fbbik"
event_name = 'Start_level'
AND level_name IS NOT NULL
```

View chỉ lấy từ `vw_tutorial_events` các dòng:

```sql id="8xiyrq"
event_name = 'Tutorial_start'
```

---

## 7. Các view đang sử dụng view này

`vw_level_tutorial_flag` được sử dụng bởi các view sau:

| View sử dụng               | Mục đích sử dụng                                                 |
| -------------------------- | ---------------------------------------------------------------- |
| `vw_level_attempts`        | Đưa thông tin tutorial vào dữ liệu lượt chơi level.              |
| `vw_pipeline_health_daily` | Theo dõi số lượng và tỷ lệ `Start_level` gần tutorial theo ngày. |

Do view này ảnh hưởng đến việc phân loại dữ liệu sạch và dữ liệu có tutorial, nếu logic sai thì các phân tích độ khó level có thể bị nhiễu.

---

## 8. Danh sách cột

| Cột                           | Kiểu dữ liệu | Ý nghĩa                                                                                                              |
| ----------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------- |
| `start_event_id`              | `STRING`     | Mã định danh của sự kiện bắt đầu level. Được tạo bằng hàm băm từ người chơi, thời gian bắt đầu, level và số lần thử. |
| `event_table_suffix`          | `STRING`     | Hậu tố ngày của bảng GA4 gốc, ví dụ `20260531`.                                                                      |
| `event_date`                  | `STRING`     | Ngày sự kiện theo định dạng của GA4.                                                                                 |
| `start_time_utc`              | `TIMESTAMP`  | Thời điểm bắt đầu level theo múi giờ UTC.                                                                            |
| `user_pseudo_id`              | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                                                                       |
| `start_app_version`           | `STRING`     | Phiên bản game tại thời điểm bắt đầu level.                                                                          |
| `level_name`                  | `STRING`     | Tên level.                                                                                                           |
| `attempt_no`                  | `INT64`      | Số lần thử level.                                                                                                    |
| `gold`                        | `INT64`      | Số vàng tại thời điểm bắt đầu level, nếu có tracking.                                                                |
| `lives_left`                  | `INT64`      | Số mạng còn lại tại thời điểm bắt đầu level, nếu có tracking.                                                        |
| `poka_name`                   | `STRING`     | Tham số `poka_name` tại thời điểm bắt đầu level, nếu có.                                                             |
| `booster_pre_used`            | `STRING`     | Booster được chọn trước khi vào level, nếu có.                                                                       |
| `is_near_tutorial_start`      | `BOOL`       | Cho biết có `Tutorial_start` gần thời điểm bắt đầu level hay không.                                                  |
| `nearby_tutorial_names`       | `STRING`     | Danh sách tên tutorial xuất hiện gần thời điểm bắt đầu level.                                                        |
| `nearby_tutorial_start_count` | `INT64`      | Số lượng sự kiện `Tutorial_start` gần thời điểm bắt đầu level.                                                       |

---

## 9. Trường định danh quan trọng

Trường định danh chính là:

```text id="8u7dq1"
start_event_id
```

Trường này được tạo bằng hàm băm từ các thành phần:

```text id="6kcx2y"
user_pseudo_id
start_time_utc
level_name
attempt_no
```

Trong tài liệu này, “hàm băm” nghĩa là cách biến một nhóm giá trị đầu vào thành một mã định danh dạng chuỗi. Mã này giúp nhận diện một sự kiện bắt đầu level.

Lưu ý: `start_event_id` trong view này được tạo từ `Start_level` trong `vw_level_events`. Nó không phải là mã gốc có sẵn từ GA4.

---

## 10. Logic xử lý dữ liệu

### 10.1 Tạo danh sách sự kiện bắt đầu level

View lấy các dòng từ `vw_level_events` với điều kiện:

```sql id="0s8hbq"
event_name = 'Start_level'
AND level_name IS NOT NULL
```

Sau đó đổi tên một số trường:

| Trường nguồn     | Trường trong view   |
| ---------------- | ------------------- |
| `event_time_utc` | `start_time_utc`    |
| `app_version`    | `start_app_version` |

View cũng giữ lại các thông tin tại thời điểm bắt đầu level:

```text id="t8l5fk"
gold
lives_left
poka_name
booster_pre_used
```

---

### 10.2 Tạo danh sách sự kiện bắt đầu hướng dẫn

View lấy các dòng từ `vw_tutorial_events` với điều kiện:

```sql id="j80yz7"
event_name = 'Tutorial_start'
```

Các trường được dùng gồm:

```text id="yfd3pb"
event_time_utc
user_pseudo_id
level_name
tutorial_name
```

Lưu ý: trong logic nối hiện tại, `level_name` của tutorial được lấy ra nhưng không được dùng làm điều kiện nối.

---

### 10.3 Điều kiện xác định “gần tutorial”

Một sự kiện `Start_level` được xem là gần tutorial nếu cùng người chơi có `Tutorial_start` nằm trong khoảng:

```text id="jdwcwt"
3 giây trước Start_level
đến
3 giây sau Start_level
```

Logic nối thời gian:

```sql id="2z6toc"
tutorial_start_events.event_time_utc BETWEEN
  TIMESTAMP_SUB(start_level_events.start_time_utc, INTERVAL 3 SECOND)
  AND TIMESTAMP_ADD(start_level_events.start_time_utc, INTERVAL 3 SECOND)
```

Điều kiện nối theo người chơi:

```sql id="4pfpa2"
start_level_events.user_pseudo_id = tutorial_start_events.user_pseudo_id
```

Như vậy, view dùng cửa sổ thời gian ±3 giây quanh thời điểm bắt đầu level để xác định mối liên hệ với tutorial.

---

### 10.4 Tạo cờ `is_near_tutorial_start`

Nếu có ít nhất một sự kiện `Tutorial_start` nằm trong cửa sổ ±3 giây, cột sau sẽ là `TRUE`:

```text id="74y5yw"
is_near_tutorial_start
```

Logic:

```sql id="gf6xy7"
COUNT(tutorial_start_events.event_time_utc) > 0
```

Nếu không có sự kiện `Tutorial_start` gần đó, cột này là `FALSE`.

---

### 10.5 Tổng hợp tên tutorial gần thời điểm bắt đầu level

View tạo cột:

```text id="qwtfyy"
nearby_tutorial_names
```

bằng cách gộp các `tutorial_name` khác nhau gần cùng một `Start_level`.

Các tên tutorial được nối bằng ký tự:

```text id="3037kx"
" | "
```

Ví dụ:

```text id="wta89p"
tutorial_swipe | Nectar
```

Lưu ý: thứ tự trong `nearby_tutorial_names` không nên được xem là có ý nghĩa phân tích, vì logic hiện tại dùng `STRING_AGG(DISTINCT ...)` mà không sắp xếp cố định.

---

### 10.6 Đếm số tutorial gần thời điểm bắt đầu level

View tạo cột:

```text id="m6nzi1"
nearby_tutorial_start_count
```

Cột này cho biết số lượng sự kiện `Tutorial_start` nằm trong cửa sổ ±3 giây quanh `Start_level`.

---

## 11. Lưu ý quan trọng về logic nối

### 11.1 View nối theo người chơi và thời gian, không nối theo level

Điều kiện nối hiện tại là:

```text id="qnjtey"
cùng user_pseudo_id
và Tutorial_start nằm trong ±3 giây quanh Start_level
```

View không nối theo `level_name`.

Điều này có nghĩa là nếu cùng một người chơi có tutorial gần thời điểm bắt đầu level, view sẽ đánh dấu là gần tutorial, ngay cả khi `level_name` giữa hai sự kiện không hoàn toàn khớp.

Cách làm này có thể hợp lý nếu tracking tutorial và level không luôn gửi cùng định dạng `level_name`. Tuy nhiên, khi kiểm tra dữ liệu, cần nhớ rằng cờ này là cờ theo khoảng thời gian gần nhau, không phải cờ theo level khớp tuyệt đối.

---

### 11.2 View chỉ xét `Start_level` thật

View này lấy `Start_level` từ `vw_level_events`.

Điều đó có nghĩa là các dòng bắt đầu level được suy luận từ `Tutorial_start` trong `vw_level_start_eff` không nằm trong view này.

Đây là điểm cần hiểu rõ khi so sánh giữa:

```text id="uvt1zq"
vw_level_start_eff
```

và:

```text id="oq4x0z"
vw_level_tutorial_flag
```

---

### 11.3 View không có `tutorial_event_id`

View này không đưa ra mã định danh của từng sự kiện `Tutorial_start` gần đó.

View chỉ cho biết:

* có tutorial gần đó hay không;
* tên tutorial gần đó là gì;
* có bao nhiêu sự kiện `Tutorial_start` gần đó.

Nếu cần phân tích chi tiết từng sự kiện tutorial, nên truy vấn trực tiếp `vw_tutorial_events`.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Kiểm tra tỷ lệ bắt đầu level gần tutorial theo level

```sql id="njwfoc"
SELECT
  event_date,
  level_name,

  COUNT(*) AS start_level_count,

  COUNTIF(is_near_tutorial_start IS TRUE)
    AS near_tutorial_start_count,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(is_near_tutorial_start IS TRUE),
      COUNT(*)
    ),
    4
  ) AS near_tutorial_start_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

GROUP BY
  event_date,
  level_name

ORDER BY
  event_date,
  level_name;
```

Truy vấn này giúp phát hiện level nào có tỷ lệ bắt đầu gần tutorial cao.

---

### 12.2 Kiểm tra danh sách tutorial gần từng level

```sql id="gazl1s"
SELECT
  level_name,
  nearby_tutorial_names,

  COUNT(*) AS start_level_count,

  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

WHERE
  is_near_tutorial_start IS TRUE

GROUP BY
  level_name,
  nearby_tutorial_names

ORDER BY
  start_level_count DESC;
```

Truy vấn này giúp biết level nào thường xuất hiện gần tutorial nào.

---

### 12.3 Kiểm tra trùng `start_event_id`

```sql id="wzmks7"
SELECT
  start_event_id,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

GROUP BY
  start_event_id

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="lk1p3g"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là mã định danh `start_event_id` không đủ duy nhất hoặc dữ liệu nguồn có dòng trùng.

---

### 12.4 Kiểm tra số lượng tutorial gần một lần bắt đầu level

```sql id="wq8x07"
SELECT
  nearby_tutorial_start_count,
  COUNT(*) AS start_level_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

GROUP BY
  nearby_tutorial_start_count

ORDER BY
  nearby_tutorial_start_count;
```

Nếu có nhiều dòng có `nearby_tutorial_start_count` lớn hơn 1, cần kiểm tra xem tutorial tracking có gửi nhiều sự kiện gần như cùng lúc hay không.

---

### 12.5 Kiểm tra các dòng thiếu `attempt_no`

```sql id="ndhbg1"
SELECT
  event_date,
  level_name,

  COUNT(*) AS start_level_count,

  COUNTIF(attempt_no IS NULL)
    AS missing_attempt_no_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

GROUP BY
  event_date,
  level_name

ORDER BY
  event_date,
  level_name;
```

Nếu thiếu `attempt_no` nhiều, việc ghép sang `End_level` trong `vw_level_attempts` có thể bị ảnh hưởng.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Lấy các lần bắt đầu level sạch, không gần tutorial

```sql id="rj1mzr"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

WHERE
  is_near_tutorial_start IS FALSE;
```

Các dòng này phù hợp hơn cho phân tích độ khó level thông thường.

---

### 13.2 Lấy các lần bắt đầu level gần tutorial

```sql id="vtm3jk"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

WHERE
  is_near_tutorial_start IS TRUE

ORDER BY
  start_time_utc,
  user_pseudo_id,
  level_name;
```

Các dòng này nên được phân tích riêng để hiểu tác động của tutorial.

---

### 13.3 So sánh số lượt bắt đầu sạch và gần tutorial theo level

```sql id="17dv4y"
SELECT
  level_name,

  COUNTIF(is_near_tutorial_start IS FALSE)
    AS clean_start_count,

  COUNTIF(is_near_tutorial_start IS TRUE)
    AS near_tutorial_start_count,

  COUNT(*) AS total_start_count,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(is_near_tutorial_start IS TRUE),
      COUNT(*)
    ),
    4
  ) AS near_tutorial_start_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`

GROUP BY
  level_name

ORDER BY
  near_tutorial_start_rate DESC,
  total_start_count DESC;
```

---

## 14. Lưu ý khi sử dụng

### 14.1 Không nên dùng dữ liệu gần tutorial để kết luận độ khó level thông thường

Nếu một lần chơi level xảy ra gần tutorial, hành vi người chơi có thể không phản ánh tự nhiên.

Ví dụ:

* người chơi đang được hướng dẫn;
* game có thể ép người chơi thao tác theo kịch bản;
* level có thể được thiết kế đặc biệt để dạy cơ chế;
* kết quả thắng/thua có thể không phản ánh độ khó thật.

Vì vậy, khi phân tích độ khó level, nên ưu tiên dùng dữ liệu có:

```text id="m9kqfy"
is_near_tutorial_start IS FALSE
```

---

### 14.2 Cờ tutorial là tín hiệu gần thời gian, không phải kết luận tuyệt đối

`is_near_tutorial_start = TRUE` nghĩa là có `Tutorial_start` gần thời điểm `Start_level`.

Điều này không nhất thiết khẳng định chắc chắn rằng toàn bộ lượt chơi bị tutorial điều khiển.

Nó là một tín hiệu cần dùng để phân loại và kiểm tra thêm.

---

### 14.3 Thứ tự trong `nearby_tutorial_names` không ổn định tuyệt đối

Do cách gộp tên tutorial hiện tại không sắp xếp cố định, cùng một tập tutorial có thể hiển thị theo thứ tự khác nhau.

Ví dụ hai chuỗi sau có thể mang cùng ý nghĩa:

```text id="xdpm71"
tutorial_swipe | Nectar
Nectar | tutorial_swipe
```

Khi so sánh dữ liệu bằng chuỗi này, nên chuẩn hóa bằng cách tách, sắp xếp rồi nối lại nếu cần so sánh chính xác.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                                          | Ảnh hưởng                                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------------- |
| Cửa sổ thời gian quá hẹp                           | Có thể bỏ sót các lần bắt đầu level bị ảnh hưởng bởi tutorial.                   |
| Cửa sổ thời gian quá rộng                          | Có thể đánh dấu nhầm các lần bắt đầu level không thật sự liên quan đến tutorial. |
| Không nối theo đúng người chơi                     | Có thể gắn tutorial của người này vào level của người khác.                      |
| Không xử lý trùng tutorial                         | Có thể làm tăng sai `nearby_tutorial_start_count`.                               |
| Sai `start_event_id`                               | Có thể ảnh hưởng đến việc ghép trong `vw_level_attempts`.                        |
| Không hiểu rõ rằng view chỉ lấy `Start_level` thật | Có thể nhầm với logic `vw_level_start_eff`.                                      |
| Dùng dữ liệu gần tutorial để kết luận độ khó       | Có thể đánh giá sai difficulty của level.                                        |

Trong đó, “difficulty” nghĩa là độ khó của level.

---

## 16. Mức độ quan trọng

```text id="j7xse5"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này quyết định lượt chơi nào được xem là gần tutorial;
* view này ảnh hưởng trực tiếp đến `vw_level_attempts`;
* view này ảnh hưởng trực tiếp đến phân tích độ khó level;
* view này giúp tách dữ liệu sạch khỏi dữ liệu có thể bị hướng dẫn làm nhiễu.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="bgpj71"
sql/ddl/core_foundation/vw_level_tutorial_flag.sql
```

Cấu trúc logic chính của view:

```sql id="g8el7a"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`
AS
WITH start_level_events AS (
  SELECT
    -- Tạo start_event_id từ user, thời gian, level và attempt_no
    event_table_suffix,
    event_date,
    event_time_utc AS start_time_utc,
    user_pseudo_id,
    app_version AS start_app_version,
    level_name,
    attempt_no,
    gold,
    lives_left,
    poka_name,
    booster_pre_used

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

  WHERE
    event_name = 'Start_level'
    AND level_name IS NOT NULL
),

tutorial_start_events AS (
  SELECT
    event_time_utc,
    user_pseudo_id,
    level_name,
    tutorial_name

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_tutorial_events`

  WHERE
    event_name = 'Tutorial_start'
)

SELECT
  start_level_events.*,

  COUNT(tutorial_start_events.event_time_utc) > 0
    AS is_near_tutorial_start,

  STRING_AGG(
    DISTINCT tutorial_start_events.tutorial_name,
    ' | '
  ) AS nearby_tutorial_names,

  COUNT(tutorial_start_events.event_time_utc)
    AS nearby_tutorial_start_count

FROM
  start_level_events

LEFT JOIN
  tutorial_start_events
ON
  start_level_events.user_pseudo_id = tutorial_start_events.user_pseudo_id
  AND tutorial_start_events.event_time_utc BETWEEN
    TIMESTAMP_SUB(start_level_events.start_time_utc, INTERVAL 3 SECOND)
    AND TIMESTAMP_ADD(start_level_events.start_time_utc, INTERVAL 3 SECOND)

GROUP BY
  -- các trường của start_level_events
;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[vw_level_events]]
	- [[vw_tutorial_events]]

- Downstream
	- [[vw_level_attempts]]
	- [[vw_pipeline_health_daily]]