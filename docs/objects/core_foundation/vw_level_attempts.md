# vw_level_attempts

## 1. Mục đích

`vw_level_attempts` là view ghép sự kiện bắt đầu level với sự kiện kết thúc level.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

Mục đích chính của view này là tạo ra dữ liệu ở cấp độ một lượt chơi level.

Một lượt chơi level được hiểu là:

```text id="p80nof"
người chơi bắt đầu một level
và nếu có dữ liệu hợp lệ thì ghép với sự kiện kết thúc level tương ứng
```

View này là nền tảng cho các phân tích:

* tỷ lệ thắng;
* tỷ lệ thua;
* số lần bắt đầu nhưng không có kết thúc;
* thời gian chơi level;
* số bước đi đã dùng;
* số bước còn lại;
* ảnh hưởng của tutorial;
* độ khó level;
* tiến trình người chơi qua các level.

---

## 2. Vai trò trong hệ thống

`vw_level_attempts` thuộc nhóm:

```text id="oho63k"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này nằm sau hai bước xử lý quan trọng:

```text id="ck8cy8"
vw_level_events
  └── làm sạch Start_level và End_level

vw_level_tutorial_flag
  └── đánh dấu Start_level có gần tutorial hay không
```

Sau đó `vw_level_attempts` ghép dữ liệu thành một dòng ở cấp độ lượt chơi.

Luồng phụ thuộc chính:

```text id="g6t375"
vw_level_events
vw_level_tutorial_flag
  └── vw_level_attempts
        ├── vw_f10_design_diag_24h
        ├── vw_user_level_reach
        ├── vw_level_difficulty_daily
        ├── vw_level_difficulty_summary
        └── vw_pipeline_health_daily
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Người chơi bắt đầu level nào?
2. Lượt bắt đầu level có được ghép với `End_level` không?
3. Người chơi thắng hay thua level?
4. Nếu thua, lý do thua là gì?
5. Người chơi dùng bao nhiêu bước đi?
6. Người chơi còn bao nhiêu bước khi kết thúc?
7. Thời lượng chơi level là bao lâu?
8. Có bao nhiêu lượt bắt đầu level không có kết thúc?
9. Lượt chơi có xảy ra gần tutorial không?
10. Lượt chơi có dùng booster không?
11. App version ở thời điểm bắt đầu và kết thúc có khớp nhau không?
12. Một level có tỷ lệ thắng, tỷ lệ thua và tỷ lệ không có kết thúc như thế nào?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_attempts` đại diện cho một sự kiện bắt đầu level từ `vw_level_tutorial_flag`.

Nếu tìm được `End_level` phù hợp, dòng đó sẽ có thêm thông tin kết thúc level.

Có thể hiểu đơn giản:

```text id="8h989x"
Một dòng = một lần người chơi bắt đầu một level
```

Trong đó:

* nếu có `End_level` phù hợp, dòng sẽ có thông tin kết thúc;
* nếu không có `End_level` phù hợp, các cột kết thúc sẽ là `NULL`.

View này giữ lại cả các lượt bắt đầu level không có kết thúc, vì đây là tín hiệu quan trọng để phân tích:

* người chơi thoát giữa level;
* lỗi tracking `End_level`;
* mất kết nối;
* lỗi ghép dữ liệu;
* hành vi rời game trong level.

---

## 5. Loại đối tượng

```text id="52ajpf"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính từ các view nguồn mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào hai nguồn chính:

| Nguồn                    | Vai trò                                                   |
| ------------------------ | --------------------------------------------------------- |
| `vw_level_tutorial_flag` | Cung cấp các sự kiện `Start_level` đã có cờ gần tutorial. |
| `vw_level_events`        | Cung cấp các sự kiện `End_level` đã được làm sạch.        |

Cụ thể:

* phần bắt đầu level lấy từ `vw_level_tutorial_flag`;
* phần kết thúc level lấy từ `vw_level_events` với điều kiện `event_name = 'End_level'`.

---

## 7. Các view đang sử dụng view này

`vw_level_attempts` được sử dụng bởi nhiều view quan trọng:

| View sử dụng                  | Mục đích sử dụng                                                                                 |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| `vw_f10_design_diag_24h`      | Chẩn đoán thiết kế 10 level đầu dựa trên lượt chơi, thắng/thua, thời lượng, số bước và tutorial. |
| `vw_user_level_reach`         | Xác định người chơi đã tới level nào, đã thắng chưa và có đi tiếp không.                         |
| `vw_level_difficulty_daily`   | Tính độ khó level theo ngày.                                                                     |
| `vw_level_difficulty_summary` | Tổng hợp độ khó level trên toàn bộ dữ liệu hiện có.                                              |
| `vw_pipeline_health_daily`    | Kiểm tra sức khỏe dữ liệu level hằng ngày.                                                       |

Do view này là nguồn chính cho phân tích lượt chơi level, mọi thay đổi logic trong view có thể ảnh hưởng lớn đến các báo cáo phía sau.

---

## 8. Danh sách cột

| Cột                           | Kiểu dữ liệu | Ý nghĩa                                                        |
| ----------------------------- | ------------ | -------------------------------------------------------------- |
| `start_event_id`              | `STRING`     | Mã định danh của sự kiện bắt đầu level.                        |
| `event_table_suffix`          | `STRING`     | Hậu tố ngày của bảng chứa sự kiện bắt đầu level.               |
| `event_date`                  | `STRING`     | Ngày của sự kiện bắt đầu level.                                |
| `start_time_utc`              | `TIMESTAMP`  | Thời điểm bắt đầu level theo múi giờ UTC.                      |
| `user_pseudo_id`              | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                 |
| `start_app_version`           | `STRING`     | Phiên bản game tại thời điểm bắt đầu level.                    |
| `level_name`                  | `STRING`     | Tên level.                                                     |
| `attempt_no`                  | `INT64`      | Số lần thử level.                                              |
| `start_gold`                  | `INT64`      | Số vàng tại thời điểm bắt đầu level, nếu có.                   |
| `start_lives_left`            | `INT64`      | Số mạng còn lại tại thời điểm bắt đầu level, nếu có.           |
| `start_poka_name`             | `STRING`     | Tham số `poka_name` tại thời điểm bắt đầu level, nếu có.       |
| `start_booster_pre_used`      | `STRING`     | Booster được chọn trước khi vào level, nếu có.                 |
| `is_near_tutorial_start`      | `BOOL`       | Cho biết lần bắt đầu level có gần `Tutorial_start` hay không.  |
| `nearby_tutorial_names`       | `STRING`     | Danh sách tên tutorial gần thời điểm bắt đầu level.            |
| `nearby_tutorial_start_count` | `INT64`      | Số lượng sự kiện `Tutorial_start` gần thời điểm bắt đầu level. |
| `end_event_table_suffix`      | `STRING`     | Hậu tố ngày của bảng chứa sự kiện kết thúc level.              |
| `end_event_date`              | `STRING`     | Ngày của sự kiện kết thúc level.                               |
| `end_time_utc`                | `TIMESTAMP`  | Thời điểm kết thúc level theo múi giờ UTC.                     |
| `end_app_version`             | `STRING`     | Phiên bản game tại thời điểm kết thúc level.                   |
| `success`                     | `BOOL`       | Người chơi thắng hay thua level.                               |
| `fail_reason`                 | `STRING`     | Lý do thua nếu có.                                             |
| `move_used`                   | `INT64`      | Số bước đi đã dùng.                                            |
| `move_left`                   | `INT64`      | Số bước đi còn lại.                                            |
| `duration_sec`                | `FLOAT64`    | Thời lượng chơi level theo giây.                               |
| `end_gold`                    | `INT64`      | Số vàng tại thời điểm kết thúc level, nếu có.                  |
| `end_lives_left`              | `INT64`      | Số mạng còn lại tại thời điểm kết thúc level, nếu có.          |
| `end_poka_name`               | `STRING`     | Tham số `poka_name` tại thời điểm kết thúc level, nếu có.      |
| `end_booster_used`            | `STRING`     | Booster được dùng trong level, nếu có.                         |
| `end_booster_pre_used`        | `STRING`     | Booster được chọn trước khi vào level, nếu có.                 |
| `seconds_from_start_to_end`   | `INT64`      | Số giây từ thời điểm bắt đầu đến thời điểm kết thúc level.     |

---

## 9. Trường định danh quan trọng

Trường định danh chính của view là:

```text id="7j661k"
start_event_id
```

Mỗi `start_event_id` đại diện cho một lần bắt đầu level.

View dùng `start_event_id` để đảm bảo mỗi lần bắt đầu level chỉ giữ lại tối đa một `End_level` phù hợp.

Các trường quan trọng để ghép bắt đầu và kết thúc level gồm:

```text id="t2yrju"
user_pseudo_id
level_name
attempt_no
start_time_utc
end_time_utc
```

---

## 10. Logic xử lý dữ liệu

### 10.1 Lấy dữ liệu bắt đầu level

View lấy dữ liệu bắt đầu level từ:

```text id="fubzxg"
vw_level_tutorial_flag
```

Các trường bắt đầu level gồm:

```text id="nn62x7"
start_event_id
event_table_suffix
event_date
start_time_utc
user_pseudo_id
start_app_version
level_name
attempt_no
start_gold
start_lives_left
start_poka_name
start_booster_pre_used
is_near_tutorial_start
nearby_tutorial_names
nearby_tutorial_start_count
```

Nguồn này đã có sẵn thông tin cờ tutorial.

---

### 10.2 Lấy dữ liệu kết thúc level

View lấy dữ liệu kết thúc level từ:

```text id="h9xjca"
vw_level_events
```

với điều kiện:

```sql id="om2m34"
event_name = 'End_level'
```

Các trường kết thúc level gồm:

```text id="dr2b58"
end_event_table_suffix
end_event_date
end_time_utc
end_app_version
success
fail_reason
move_used
move_left
duration_sec
end_gold
end_lives_left
end_poka_name
end_booster_used
end_booster_pre_used
```

---

### 10.3 Điều kiện ghép `Start_level` với `End_level`

Một sự kiện kết thúc level được ghép với một sự kiện bắt đầu level nếu thỏa mãn tất cả điều kiện sau:

```text id="ahk8cz"
cùng user_pseudo_id
cùng level_name
cùng attempt_no
end_time_utc >= start_time_utc
end_time_utc <= start_time_utc + 6 giờ
```

Tương ứng với logic:

```sql id="9xaq2g"
start_level_events.user_pseudo_id = end_level_events.user_pseudo_id
AND start_level_events.level_name = end_level_events.level_name
AND start_level_events.attempt_no = end_level_events.attempt_no
AND end_level_events.end_time_utc >= start_level_events.start_time_utc
AND end_level_events.end_time_utc <= TIMESTAMP_ADD(
  start_level_events.start_time_utc,
  INTERVAL 6 HOUR
)
```

Cửa sổ 6 giờ giúp tránh ghép một `End_level` quá xa thời điểm bắt đầu level.

---

### 10.4 Giữ lại dòng bắt đầu dù không có kết thúc

View dùng `LEFT JOIN`, nghĩa là giữ lại tất cả dòng bắt đầu level.

Nếu không tìm được `End_level` phù hợp, các cột kết thúc sẽ là `NULL`.

Trường hợp này rất quan trọng vì nó giúp đo:

```text id="f9hkw2"
unmatched start
```

Trong tài liệu này, “unmatched start” nghĩa là một lần bắt đầu level nhưng không tìm thấy sự kiện kết thúc level phù hợp.

---

### 10.5 Chọn `End_level` gần nhất sau `Start_level`

Nếu có nhiều `End_level` phù hợp với cùng một `Start_level`, view chọn sự kiện kết thúc sớm nhất sau thời điểm bắt đầu.

Logic:

```sql id="ilw4sn"
ROW_NUMBER() OVER (
  PARTITION BY start_level_events.start_event_id
  ORDER BY end_level_events.end_time_utc ASC
) = 1
```

Điều này đảm bảo mỗi `start_event_id` chỉ xuất hiện một dòng trong kết quả cuối.

---

### 10.6 Tính thời gian từ bắt đầu đến kết thúc

View tạo cột:

```text id="97q8vh"
seconds_from_start_to_end
```

bằng cách lấy số giây giữa:

```text id="x5g1y4"
end_time_utc
-
start_time_utc
```

Nếu không có `End_level` phù hợp, cột này sẽ là `NULL`.

---

## 11. Cách hiểu trạng thái một lượt chơi

Một dòng trong `vw_level_attempts` có thể thuộc một trong các trạng thái sau:

| Trạng thái                      | Cách nhận diện                                   | Ý nghĩa                                             |
| ------------------------------- | ------------------------------------------------ | --------------------------------------------------- |
| Có kết thúc và thắng            | `end_time_utc IS NOT NULL` và `success IS TRUE`  | Người chơi đã hoàn thành level thành công.          |
| Có kết thúc và thua             | `end_time_utc IS NOT NULL` và `success IS FALSE` | Người chơi đã kết thúc level nhưng thất bại.        |
| Có kết thúc nhưng thiếu success | `end_time_utc IS NOT NULL` và `success IS NULL`  | Có `End_level` nhưng thiếu trạng thái thắng/thua.   |
| Không có kết thúc               | `end_time_utc IS NULL`                           | Có `Start_level` nhưng không ghép được `End_level`. |

---

## 12. Lưu ý quan trọng về logic

### 12.1 `attempt_no` bị thiếu sẽ làm giảm khả năng ghép

Điều kiện ghép có dùng:

```text id="ake41s"
attempt_no
```

Nếu `attempt_no` bị thiếu ở `Start_level` hoặc `End_level`, hai sự kiện sẽ không ghép được theo điều kiện hiện tại.

Vì vậy, chất lượng tracking của `attempt_no` rất quan trọng.

---

### 12.2 View này chỉ dựa trên `Start_level` thật

Nguồn bắt đầu level của view này là:

```text id="j4vdmd"
vw_level_tutorial_flag
```

Mà `vw_level_tutorial_flag` chỉ lấy `Start_level` từ `vw_level_events`.

Vì vậy, các dòng bắt đầu level được suy luận từ `Tutorial_start` trong `vw_level_start_eff` không nằm trong `vw_level_attempts`.

Cần hiểu rõ điểm này khi so sánh số lượt bắt đầu level giữa các view.

---

### 12.3 Dữ liệu gần tutorial chưa bị loại bỏ

View này giữ cả hai loại:

```text id="azsawo"
is_near_tutorial_start = TRUE
is_near_tutorial_start = FALSE
```

Các view phân tích độ khó có thể lọc riêng dữ liệu sạch bằng điều kiện:

```sql id="49l5gv"
is_near_tutorial_start IS FALSE
```

---

### 12.4 Nếu có nhiều `End_level` cùng thời điểm, kết quả có thể không ổn định tuyệt đối

View chọn `End_level` bằng cách sắp xếp theo:

```text id="zb6e4l"
end_time_utc ASC
```

Nếu có nhiều `End_level` trùng cùng `end_time_utc` cho cùng một `start_event_id`, BigQuery có thể chọn một dòng bất kỳ trong các dòng trùng thời điểm.

Trường hợp này hiếm, nhưng nên được kiểm tra nếu phát hiện dữ liệu bất thường.

---

## 13. Kiểm tra chất lượng dữ liệu khuyến nghị

### 13.1 Kiểm tra tỷ lệ ghép được `End_level`

```sql id="8cry25"
SELECT
  event_date,
  start_app_version,
  level_name,

  COUNT(*) AS start_count,

  COUNTIF(end_time_utc IS NOT NULL)
    AS matched_end_count,

  COUNTIF(end_time_utc IS NULL)
    AS unmatched_start_count,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(end_time_utc IS NOT NULL),
      COUNT(*)
    ),
    4
  ) AS matched_end_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

GROUP BY
  event_date,
  start_app_version,
  level_name

ORDER BY
  event_date,
  start_app_version,
  level_name;
```

Tỷ lệ ghép được `End_level` thấp có thể do người chơi thoát, thiếu tracking, hoặc logic ghép chưa phù hợp.

---

### 13.2 Kiểm tra trùng `start_event_id`

```sql id="sn6j32"
SELECT
  start_event_id,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

GROUP BY
  start_event_id

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="aifjif"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là view không còn bảo toàn một dòng cho mỗi lần bắt đầu level.

---

### 13.3 Kiểm tra thiếu `attempt_no`

```sql id="ygyx4x"
SELECT
  event_date,
  start_app_version,
  level_name,

  COUNT(*) AS start_count,

  COUNTIF(attempt_no IS NULL)
    AS missing_attempt_no_count,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(attempt_no IS NULL),
      COUNT(*)
    ),
    4
  ) AS missing_attempt_no_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

GROUP BY
  event_date,
  start_app_version,
  level_name

ORDER BY
  event_date,
  start_app_version,
  level_name;
```

Nếu `missing_attempt_no_rate` cao, khả năng ghép với `End_level` sẽ bị ảnh hưởng.

---

### 13.4 Kiểm tra khác phiên bản giữa bắt đầu và kết thúc

```sql id="72gop1"
SELECT
  event_date,
  level_name,

  COUNT(*) AS matched_attempt_count,

  COUNTIF(start_app_version != end_app_version)
    AS app_version_mismatch_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

WHERE
  end_time_utc IS NOT NULL

GROUP BY
  event_date,
  level_name

ORDER BY
  app_version_mismatch_count DESC,
  event_date,
  level_name;
```

Nếu có nhiều dòng khác phiên bản giữa bắt đầu và kết thúc, cần kiểm tra khả năng người chơi cập nhật app giữa phiên hoặc tracking bị lệch.

---

### 13.5 Kiểm tra dữ liệu gần tutorial

```sql id="5t58o8"
SELECT
  event_date,
  level_name,

  COUNT(*) AS start_count,

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
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

GROUP BY
  event_date,
  level_name

ORDER BY
  event_date,
  level_name;
```

Dữ liệu gần tutorial nên được đọc riêng khi phân tích độ khó.

---

### 13.6 Kiểm tra thời gian từ bắt đầu đến kết thúc bất thường

```sql id="yl51z5"
SELECT
  event_date,
  level_name,
  start_event_id,
  user_pseudo_id,
  start_time_utc,
  end_time_utc,
  seconds_from_start_to_end

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

WHERE
  end_time_utc IS NOT NULL
  AND (
    seconds_from_start_to_end < 0
    OR seconds_from_start_to_end > 21600
  )

ORDER BY
  seconds_from_start_to_end DESC;
```

Kỳ vọng:

```text id="xg0omq"
Không trả ra dòng nào
```

Vì logic ghép chỉ cho phép `End_level` nằm trong vòng 6 giờ sau `Start_level`.

---

## 14. Cách sử dụng khuyến nghị

### 14.1 Tính tỷ lệ thắng theo level từ lượt chơi đã ghép

```sql id="mnr4lr"
SELECT
  start_app_version,
  level_name,

  COUNT(*) AS start_count,

  COUNTIF(end_time_utc IS NOT NULL)
    AS matched_end_count,

  COUNTIF(success IS TRUE)
    AS win_count,

  COUNTIF(success IS FALSE)
    AS fail_count,

  ROUND(
    SAFE_DIVIDE(COUNTIF(success IS TRUE), COUNTIF(end_time_utc IS NOT NULL)),
    4
  ) AS win_rate_among_matched_ends

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

WHERE
  is_near_tutorial_start IS FALSE

GROUP BY
  start_app_version,
  level_name

ORDER BY
  start_app_version,
  level_name;
```

Truy vấn này loại bỏ các lượt bắt đầu gần tutorial để phân tích độ khó tự nhiên hơn.

---

### 14.2 Xem các lượt bắt đầu không có kết thúc

```sql id="3qg42x"
SELECT
  event_date,
  start_app_version,
  level_name,
  start_event_id,
  user_pseudo_id,
  start_time_utc,
  attempt_no,
  is_near_tutorial_start,
  nearby_tutorial_names

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

WHERE
  end_time_utc IS NULL

ORDER BY
  start_time_utc,
  level_name;
```

Truy vấn này dùng để điều tra các trường hợp `Start_level` không ghép được `End_level`.

---

### 14.3 Tính thời lượng chơi trung bình theo level

```sql id="qmb52n"
SELECT
  start_app_version,
  level_name,

  COUNTIF(end_time_utc IS NOT NULL)
    AS matched_end_count,

  AVG(duration_sec) AS avg_duration_sec,
  APPROX_QUANTILES(duration_sec, 100)[OFFSET(50)]
    AS median_duration_sec,

  AVG(seconds_from_start_to_end)
    AS avg_seconds_from_start_to_end

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

WHERE
  end_time_utc IS NOT NULL
  AND is_near_tutorial_start IS FALSE

GROUP BY
  start_app_version,
  level_name

ORDER BY
  start_app_version,
  level_name;
```

Trong truy vấn này:

* `duration_sec` là thời lượng do game gửi;
* `seconds_from_start_to_end` là thời gian tính từ timestamp của `Start_level` và `End_level`.

Hai giá trị này có thể khác nhau và nên được kiểm tra khi có bất thường.

---

## 15. Lưu ý khi sử dụng

### 15.1 Không nên tính độ khó nếu bỏ qua `end_time_utc IS NULL`

Các lượt không có `End_level` cũng là thông tin quan trọng.

Nếu chỉ tính trên các dòng có `End_level`, có thể đánh giá thấp vấn đề rời bỏ giữa level hoặc thiếu tracking.

Nên luôn theo dõi thêm:

```text id="hxe7i8"
unmatched_start_count
matched_end_rate
```

---

### 15.2 Nên lọc tutorial khi phân tích độ khó thường

Với phân tích độ khó level thông thường, nên dùng:

```sql id="g4v1x9"
is_near_tutorial_start IS FALSE
```

Vì các lượt gần tutorial có thể không phản ánh hành vi chơi tự nhiên.

---

### 15.3 Cần phân biệt tỷ lệ thắng theo `End_level` và tỷ lệ thắng theo `Start_level`

Có hai cách tính thường gặp:

```text id="eray22"
win_count / matched_end_count
```

và:

```text id="yth0oy"
win_count / start_count
```

Hai cách này trả lời hai câu hỏi khác nhau:

| Cách tính                       | Ý nghĩa                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `win_count / matched_end_count` | Trong các lượt có kết thúc, tỷ lệ thắng là bao nhiêu?          |
| `win_count / start_count`       | Trong tất cả lượt bắt đầu, bao nhiêu lượt kết thúc bằng thắng? |

Nếu `matched_end_rate` thấp, hai tỷ lệ này có thể khác nhau nhiều.

---

## 16. Rủi ro nếu view sai logic

| Lỗi logic                                 | Ảnh hưởng                                                    |
| ----------------------------------------- | ------------------------------------------------------------ |
| Ghép sai `Start_level` với `End_level`    | Sai toàn bộ chỉ số lượt chơi level.                          |
| Không giữ unmatched start                 | Không phát hiện được người chơi thoát hoặc thiếu tracking.   |
| Sai điều kiện `attempt_no`                | Không ghép được hoặc ghép nhầm lượt chơi.                    |
| Cửa sổ 6 giờ quá rộng                     | Có thể ghép nhầm `End_level` xa thời điểm bắt đầu.           |
| Cửa sổ 6 giờ quá hẹp                      | Có thể bỏ sót các lượt chơi rất dài hoặc session bất thường. |
| Không tách dữ liệu gần tutorial           | Đánh giá sai độ khó level.                                   |
| Chọn sai `End_level` nếu có nhiều kết quả | Sai trạng thái thắng/thua và thời lượng.                     |

---

## 17. Mức độ quan trọng

```text id="k2uv4z"
Mức độ quan trọng: Rất cao
```

Lý do:

* view này là nguồn chính cho phân tích lượt chơi level;
* view này ảnh hưởng trực tiếp đến độ khó level;
* view này ảnh hưởng đến phân tích tiến trình người chơi;
* view này ảnh hưởng đến chẩn đoán thiết kế 10 level đầu;
* view này ảnh hưởng đến kiểm tra sức khỏe dữ liệu.

Nếu `vw_level_attempts` sai, nhiều view phân tích phía sau sẽ sai theo.

---

## 18. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="c0s110"
sql/ddl/core_foundation/vw_level_attempts.sql
```

Cấu trúc logic chính của view:

```sql id="8259ol"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`
AS
WITH start_level_events AS (
  SELECT
    start_event_id,
    event_table_suffix,
    event_date,
    start_time_utc,
    user_pseudo_id,
    start_app_version,
    level_name,
    attempt_no,
    gold AS start_gold,
    lives_left AS start_lives_left,
    poka_name AS start_poka_name,
    booster_pre_used AS start_booster_pre_used,
    is_near_tutorial_start,
    nearby_tutorial_names,
    nearby_tutorial_start_count

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_tutorial_flag`
),

end_level_events AS (
  SELECT
    event_table_suffix AS end_event_table_suffix,
    event_date AS end_event_date,
    event_time_utc AS end_time_utc,
    user_pseudo_id,
    app_version AS end_app_version,
    level_name,
    attempt_no,
    success,
    fail_reason,
    move_used,
    move_left,
    duration_sec,
    gold AS end_gold,
    lives_left AS end_lives_left,
    poka_name AS end_poka_name,
    booster_used AS end_booster_used,
    booster_pre_used AS end_booster_pre_used

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_events`

  WHERE
    event_name = 'End_level'
)

SELECT
  -- start fields
  -- end fields
  -- seconds_from_start_to_end

FROM
  start_level_events

LEFT JOIN
  end_level_events
ON
  start_level_events.user_pseudo_id = end_level_events.user_pseudo_id
  AND start_level_events.level_name = end_level_events.level_name
  AND start_level_events.attempt_no = end_level_events.attempt_no
  AND end_level_events.end_time_utc >= start_level_events.start_time_utc
  AND end_level_events.end_time_utc <= TIMESTAMP_ADD(
    start_level_events.start_time_utc,
    INTERVAL 6 HOUR
  )

QUALIFY
  ROW_NUMBER() OVER (
    PARTITION BY start_level_events.start_event_id
    ORDER BY end_level_events.end_time_utc ASC
  ) = 1;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[vw_level_tutorial_flag]]
	- [[vw_level_events]]

- Downstream:
	- [[vw_f10_design_diag_24h]]
	- [[vw_user_level_reach]]
	- [[vw_level_difficulty_daily]]
	- [[vw_level_difficulty_summary]]
	- [[vw_pipeline_health_daily]]

