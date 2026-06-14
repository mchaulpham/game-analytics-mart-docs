# vw_level_difficulty_daily

## 1. Mục đích

`vw_level_difficulty_daily` là view tổng hợp độ khó level theo ngày.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này trả lời câu hỏi:

```text id="5ls1gl"
Mỗi ngày, từng normal level có win rate, fail rate, duration, move usage và trạng thái độ khó như thế nào?
```

Trong tài liệu này, “normal level” nghĩa là level chính có tên dạng `Nxxx`, ví dụ `N001`, `N002`, `N010`.

View này chỉ dùng các lượt chơi không gần tutorial để đánh giá độ khó tự nhiên hơn.

---

## 2. Vai trò trong hệ thống

`vw_level_difficulty_daily` thuộc nhóm:

```text id="1vrdo0"
monitoring_level_analysis
```

Tức là nhóm theo dõi và phân tích level.

View này là bảng theo dõi độ khó theo ngày. Nó phù hợp để:

* xem trend độ khó của level theo thời gian;
* phát hiện level có win rate thấp bất thường;
* phát hiện tracking risk nếu `End_level` bị thiếu nhiều;
* theo dõi thời lượng level;
* theo dõi số move dùng và move còn lại.

Trong tài liệu này, “trend” nghĩa là xu hướng thay đổi theo thời gian.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Hôm nay level nào có win rate thấp?
2. Hôm nay level nào có fail rate cao?
3. Level nào đang bị phân loại là `very_hard`, `hard`, `medium`, `easy`, hoặc `very_easy`?
4. Level nào có `matched_end_rate` thấp, cần kiểm tra tracking?
5. Level nào có sample thấp nên chưa thể kết luận mạnh?
6. Duration trung bình và p90 của từng level theo ngày là bao nhiêu?
7. Người chơi dùng bao nhiêu move ở từng level?
8. Người chơi còn lại bao nhiêu move khi kết thúc?
9. Có level nào thay đổi độ khó theo ngày không?
10. Dữ liệu level hôm nay có đủ tin cậy để đọc không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_difficulty_daily` đại diện cho một level trong một ngày.

Có thể hiểu đơn giản:

```text id="p648bq"
Một dòng = một event_date + một level_name
```

View này không có chiều `app_version`.

Điều đó có nghĩa là nếu cùng một `level_name` xuất hiện ở nhiều phiên bản game trong cùng ngày, chỉ số sẽ được gộp chung.

---

## 5. Loại đối tượng

```text id="4bwwqj"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc trực tiếp vào:

```text id="ycdr61"
vw_level_attempts
```

View chỉ lấy dữ liệu thỏa mãn hai điều kiện:

```sql id="8v89m6"
REGEXP_CONTAINS(level_name, r'^N\d+$')
AND is_near_tutorial_start = FALSE
```

Ý nghĩa:

* chỉ lấy normal level dạng `Nxxx`;
* loại các lượt bắt đầu level gần tutorial.

---

## 7. Các view đang sử dụng view này

`vw_level_difficulty_daily` được sử dụng bởi:

| View sử dụng               | Mục đích sử dụng                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------- |
| `vw_pipeline_health_daily` | Theo dõi số lượng level có sample đủ, tracking risk, và các tín hiệu độ khó theo ngày. |

---

## 8. Danh sách cột

| Cột                            | Kiểu dữ liệu | Ý nghĩa                                          |
| ------------------------------ | ------------ | ------------------------------------------------ |
| `event_date`                   | `STRING`     | Ngày sự kiện, lấy từ ngày bắt đầu level.         |
| `level_name`                   | `STRING`     | Tên normal level, ví dụ `N001`.                  |
| `normal_start_count`           | `INT64`      | Tổng số lượt bắt đầu level sạch trong ngày.      |
| `matched_end_count`            | `INT64`      | Số lượt bắt đầu ghép được với `End_level`.       |
| `start_without_end_count`      | `INT64`      | Số lượt bắt đầu không ghép được với `End_level`. |
| `matched_end_rate`             | `FLOAT64`    | Tỷ lệ lượt bắt đầu ghép được với `End_level`.    |
| `win_count`                    | `INT64`      | Số lượt kết thúc bằng thắng.                     |
| `fail_count`                   | `INT64`      | Số lượt kết thúc bằng thua.                      |
| `win_rate_among_matched_ends`  | `FLOAT64`    | Tỷ lệ thắng trong các lượt có `End_level`.       |
| `fail_rate_among_matched_ends` | `FLOAT64`    | Tỷ lệ thua trong các lượt có `End_level`.        |
| `avg_duration_sec`             | `FLOAT64`    | Thời lượng trung bình của level theo giây.       |
| `median_duration_sec`          | `FLOAT64`    | Trung vị thời lượng level theo giây.             |
| `p75_duration_sec`             | `FLOAT64`    | Mốc 75% thời lượng level.                        |
| `p90_duration_sec`             | `FLOAT64`    | Mốc 90% thời lượng level.                        |
| `max_duration_sec`             | `FLOAT64`    | Thời lượng lớn nhất trong ngày.                  |
| `avg_move_used`                | `FLOAT64`    | Số move đã dùng trung bình.                      |
| `median_move_used`             | `FLOAT64`    | Trung vị số move đã dùng.                        |
| `avg_move_left`                | `FLOAT64`    | Số move còn lại trung bình.                      |
| `median_move_left`             | `FLOAT64`    | Trung vị số move còn lại.                        |
| `sample_size_status`           | `STRING`     | Trạng thái kích thước mẫu trong ngày.            |
| `difficulty_band`              | `STRING`     | Nhóm độ khó tự động.                             |
| `analyst_note`                 | `STRING`     | Ghi chú diễn giải tự động bằng tiếng Việt.       |

---

## 9. Công thức chỉ số chính

### 9.1 Tỷ lệ ghép được `End_level`

```text id="lok0p8"
matched_end_rate
=
matched_end_count
/
normal_start_count
```

Ý nghĩa: trong số lượt bắt đầu level, bao nhiêu lượt có sự kiện kết thúc tương ứng.

---

### 9.2 Tỷ lệ thắng

```text id="269gcr"
win_rate_among_matched_ends
=
win_count
/
matched_end_count
```

Ý nghĩa: trong các lượt có `End_level`, bao nhiêu lượt thắng.

---

### 9.3 Tỷ lệ thua

```text id="qcg51v"
fail_rate_among_matched_ends
=
fail_count
/
matched_end_count
```

Ý nghĩa: trong các lượt có `End_level`, bao nhiêu lượt thua.

---

## 10. Logic phân loại sample

Cột:

```text id="xq01bt"
sample_size_status
```

được tính như sau:

| Điều kiện                  | Nhãn                |
| -------------------------- | ------------------- |
| `normal_start_count >= 30` | `sufficient_sample` |
| còn lại                    | `low_sample`        |

Ý nghĩa:

* `sufficient_sample`: có thể đọc như một tín hiệu tương đối ổn;
* `low_sample`: chỉ nên dùng để tham khảo trend, không kết luận mạnh.

---

## 11. Logic phân loại độ khó

Cột:

```text id="px3927"
difficulty_band
```

được tính theo thứ tự ưu tiên sau:

| Điều kiện                             | Nhãn                                     | Ý nghĩa                                                |
| ------------------------------------- | ---------------------------------------- | ------------------------------------------------------ |
| `matched_end_rate < 0.7`              | `tracking_risk_do_not_conclude_strongly` | Tỷ lệ ghép `End_level` thấp, chưa nên kết luận độ khó. |
| `win_rate_among_matched_ends <= 0.25` | `very_hard`                              | Rất khó.                                               |
| `win_rate_among_matched_ends <= 0.45` | `hard`                                   | Khó.                                                   |
| `win_rate_among_matched_ends <= 0.70` | `medium`                                 | Trung bình.                                            |
| `win_rate_among_matched_ends <= 0.85` | `easy`                                   | Dễ.                                                    |
| còn lại                               | `very_easy`                              | Rất dễ.                                                |

Điểm quan trọng: view kiểm tra `matched_end_rate` trước khi phân loại theo win rate.

Nếu `matched_end_rate` thấp, view ưu tiên cảnh báo rủi ro tracking thay vì kết luận level khó hay dễ.

---

## 12. Ghi chú tự động

Cột:

```text id="7l84lc"
analyst_note
```

là ghi chú tiếng Việt tự động dựa trên sample, tracking và win rate.

Các nhóm ghi chú chính:

| Điều kiện           | Ý nghĩa ghi chú                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| Sample thấp         | Chỉ tham khảo trend, không kết luận mạnh.                              |
| Matched End thấp    | Cần kiểm tra tracking hoặc hành vi quit trước khi kết luận độ khó.     |
| Win rate rất thấp   | Rất khó, cần kiểm tra move, target, obstacle hoặc context trước level. |
| Win rate thấp       | Có thể là difficulty spike.                                            |
| Win rate trung bình | Cần so với mục tiêu difficulty curve.                                  |
| Win rate cao        | Có thể phù hợp cho early game hoặc level hồi nhịp.                     |
| Win rate rất cao    | Cần kiểm tra có quá dễ hoặc là tutorial/relief level không.            |

Trong tài liệu này:

* “difficulty spike” nghĩa là điểm tăng độ khó đột ngột;
* “difficulty curve” nghĩa là đường cong độ khó theo tiến trình game;
* “relief level” nghĩa là level nhẹ hơn, dùng để giảm căng thẳng sau các level khó.

---

## 13. Kiểm tra chất lượng dữ liệu khuyến nghị

### 13.1 Kiểm tra level có tracking risk theo ngày

```sql id="6cmtc1"
SELECT
  event_date,
  level_name,
  normal_start_count,
  matched_end_count,
  start_without_end_count,
  matched_end_rate,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  difficulty_band = 'tracking_risk_do_not_conclude_strongly'

ORDER BY
  event_date,
  matched_end_rate ASC;
```

Truy vấn này giúp tách vấn đề dữ liệu khỏi vấn đề thiết kế level.

---

### 13.2 Kiểm tra level rất khó hoặc khó

```sql id="5qzuad"
SELECT
  event_date,
  level_name,
  normal_start_count,
  matched_end_count,
  win_count,
  fail_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  avg_move_used,
  avg_move_left,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  difficulty_band IN ('very_hard', 'hard')

ORDER BY
  event_date,
  win_rate_among_matched_ends ASC;
```

---

### 13.3 Kiểm tra sample thấp

```sql id="3g3ak6"
SELECT
  event_date,
  level_name,
  normal_start_count,
  sample_size_status,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  sample_size_status = 'low_sample'

ORDER BY
  event_date,
  normal_start_count ASC;
```

Không nên ra quyết định mạnh từ các dòng này.

---

### 13.4 Xem trend độ khó của một level

```sql id="ei2e60"
SELECT
  event_date,
  level_name,
  normal_start_count,
  matched_end_rate,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  difficulty_band

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  level_name = 'N001'

ORDER BY
  event_date;
```

Thay `N001` bằng level cần kiểm tra.

---

### 13.5 Kiểm tra tỷ lệ thắng và thua có hợp lý không

```sql id="4mrvlm"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  win_rate_among_matched_ends < 0
  OR win_rate_among_matched_ends > 1
  OR fail_rate_among_matched_ends < 0
  OR fail_rate_among_matched_ends > 1;
```

Kỳ vọng:

```text id="7u9jli"
Không trả ra dòng nào
```

---

## 14. Cách sử dụng khuyến nghị

### 14.1 Theo dõi độ khó mỗi ngày

```sql id="yvuasi"
SELECT
  event_date,
  level_name,
  normal_start_count,
  matched_end_rate,
  win_rate_among_matched_ends,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

ORDER BY
  event_date,
  level_name;
```

---

### 14.2 Lấy danh sách level cần kiểm tra trong ngày gần nhất

```sql id="ysv740"
WITH latest_day AS (
  SELECT
    MAX(event_date) AS event_date

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`
)

SELECT
  d.*

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily` AS d

INNER JOIN
  latest_day
ON
  d.event_date = latest_day.event_date

WHERE
  d.sample_size_status = 'sufficient_sample'
  AND d.difficulty_band IN (
    'tracking_risk_do_not_conclude_strongly',
    'very_hard',
    'hard',
    'very_easy'
  )

ORDER BY
  CASE d.difficulty_band
    WHEN 'tracking_risk_do_not_conclude_strongly' THEN 1
    WHEN 'very_hard' THEN 2
    WHEN 'hard' THEN 3
    WHEN 'very_easy' THEN 4
    ELSE 99
  END,
  d.level_name;
```

---

### 14.3 So sánh duration và win rate

```sql id="oufybq"
SELECT
  event_date,
  level_name,
  normal_start_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  avg_move_used,
  avg_move_left,
  difficulty_band

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`

WHERE
  sample_size_status = 'sufficient_sample'

ORDER BY
  p90_duration_sec DESC,
  win_rate_among_matched_ends ASC;
```

Truy vấn này giúp phát hiện level vừa lâu vừa khó.

---

## 15. Lưu ý khi sử dụng

### 15.1 View không có `app_version`

View này gộp theo:

```text id="fnz2yg"
event_date
level_name
```

Không có `app_version`.

Nếu cùng một level có thay đổi giữa các phiên bản game, view này có thể gộp nhiều phiên bản lại với nhau.

Khi cần phân tích theo phiên bản game, không nên dùng view này làm nguồn duy nhất.

---

### 15.2 View chỉ dùng dữ liệu không gần tutorial

Điều kiện:

```sql id="0ed34m"
is_near_tutorial_start = FALSE
```

giúp loại các lượt chơi gần tutorial.

Điều này tốt cho phân tích độ khó tự nhiên, nhưng có nghĩa là dữ liệu tutorial sẽ không xuất hiện trong view này.

---

### 15.3 View chỉ phân tích normal level dạng `Nxxx`

Các level đặc biệt như `c001`, `E1`, hoặc các level không theo mẫu `Nxxx` không nằm trong view này.

---

### 15.4 Không nên kết luận mạnh khi sample thấp

Nếu:

```text id="0hn4ll"
sample_size_status = low_sample
```

thì chỉ nên dùng để theo dõi, không nên ra quyết định tuning level.

Trong tài liệu này, “tuning level” nghĩa là chỉnh các thông số thiết kế level như số move, mục tiêu, bố cục, blocker, spawn rate, hoặc độ khó tổng thể.

---

### 15.5 Tracking risk được ưu tiên hơn kết luận difficulty

Nếu:

```text id="6vmtui"
matched_end_rate < 0.7
```

view gán:

```text id="ka0rxk"
tracking_risk_do_not_conclude_strongly
```

Trong trường hợp này, cần kiểm tra dữ liệu trước khi kết luận level khó hay dễ.

---

## 16. Rủi ro nếu view sai logic

| Lỗi logic                           | Ảnh hưởng                                                      |
| ----------------------------------- | -------------------------------------------------------------- |
| Không loại tutorial                 | Độ khó bị nhiễu bởi hướng dẫn.                                 |
| Lọc sai normal level                | Thiếu hoặc thừa level trong phân tích.                         |
| Gộp không có app_version            | Có thể che mất khác biệt giữa các phiên bản game.              |
| Sai matched_end_rate                | Nhầm tracking risk với difficulty.                             |
| Sai win_rate                        | Phân loại độ khó sai.                                          |
| Sample thấp nhưng vẫn kết luận mạnh | Dễ chỉnh sai level.                                            |
| Ngưỡng difficulty không phù hợp     | Gắn nhãn very_hard/hard/easy không đúng với mục tiêu thiết kế. |

---

## 17. Mức độ quan trọng

```text id="m7u2wu"
Mức độ quan trọng: Trung bình đến cao
```

Lý do:

* view này hữu ích để theo dõi độ khó hằng ngày;
* view này giúp phát hiện tracking risk theo ngày;
* view này giúp theo dõi trend level;
* tuy nhiên, do không có `app_version`, cần dùng thận trọng khi game có nhiều phiên bản đang chạy cùng lúc.

---

## 18. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="f7xy3j"
sql/ddl/monitoring_level_analysis/vw_level_difficulty_daily.sql
```

Cấu trúc logic chính của view:

```sql id="22xzyp"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_daily`
AS
WITH daily_level_summary AS (
  SELECT
    event_date,
    level_name,

    COUNT(*) AS normal_start_count,
    COUNT(end_time_utc) AS matched_end_count,
    COUNTIF(end_time_utc IS NULL) AS start_without_end_count,

    COUNTIF(success = TRUE) AS win_count,
    COUNTIF(success = FALSE) AS fail_count,

    AVG(duration_sec) AS avg_duration_sec,
    APPROX_QUANTILES(duration_sec, 100)[OFFSET(50)] AS median_duration_sec,
    APPROX_QUANTILES(duration_sec, 100)[OFFSET(75)] AS p75_duration_sec,
    APPROX_QUANTILES(duration_sec, 100)[OFFSET(90)] AS p90_duration_sec,

    AVG(move_used) AS avg_move_used,
    APPROX_QUANTILES(move_used, 100)[OFFSET(50)] AS median_move_used,

    AVG(move_left) AS avg_move_left,
    APPROX_QUANTILES(move_left, 100)[OFFSET(50)] AS median_move_left

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

  WHERE
    REGEXP_CONTAINS(level_name, r'^N\d+$')
    AND is_near_tutorial_start = FALSE

  GROUP BY
    event_date,
    level_name
)

SELECT
  *,
  CASE
    WHEN normal_start_count >= 30 THEN 'sufficient_sample'
    ELSE 'low_sample'
  END AS sample_size_status,

  CASE
    WHEN matched_end_rate < 0.7 THEN 'tracking_risk_do_not_conclude_strongly'
    WHEN win_rate_among_matched_ends <= 0.25 THEN 'very_hard'
    WHEN win_rate_among_matched_ends <= 0.45 THEN 'hard'
    WHEN win_rate_among_matched_ends <= 0.70 THEN 'medium'
    WHEN win_rate_among_matched_ends <= 0.85 THEN 'easy'
    ELSE 'very_easy'
  END AS difficulty_band,

  CASE
    -- analyst_note tiếng Việt
  END AS analyst_note

FROM
  daily_level_summary;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: [[vw_level_attempts]]

- Downstream: [[vw_pipeline_health_daily]]

