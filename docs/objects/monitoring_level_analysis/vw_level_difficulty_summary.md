# vw_level_difficulty_summary

## 1. Mục đích

`vw_level_difficulty_summary` là view tổng hợp độ khó level trên toàn bộ dữ liệu hiện có.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này trả lời câu hỏi:

```text id="muv845"
Tính trên toàn bộ dữ liệu hiện có, từng normal level đang có độ khó như thế nào?
```

View này tương tự `vw_level_difficulty_daily`, nhưng không chia theo ngày. Nó gom toàn bộ dữ liệu lại theo `level_name`.

---

## 2. Vai trò trong hệ thống

`vw_level_difficulty_summary` thuộc nhóm:

```text id="tiduas"
monitoring_level_analysis
```

Tức là nhóm theo dõi và phân tích level.

View này phù hợp để xem bức tranh tổng quan về độ khó từng level, không phải biến động theo từng ngày.

Nó hữu ích khi cần:

* xếp hạng level theo win rate;
* tìm level rất khó hoặc rất dễ;
* kiểm tra tracking risk ở cấp tổng hợp;
* xem duration và move usage tổng thể;
* có một baseline độ khó cho từng level.

Trong tài liệu này, “baseline” nghĩa là mức nền hoặc mức tham chiếu ban đầu để so sánh về sau.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Tính trên toàn bộ dữ liệu, level nào khó nhất?
2. Level nào dễ nhất?
3. Level nào có `matched_end_rate` thấp, cần kiểm tra tracking?
4. Level nào có win rate thấp?
5. Level nào có duration cao?
6. Level nào có p90 duration cao?
7. Level nào có move usage cao?
8. Người chơi thường còn lại bao nhiêu move khi thắng hoặc kết thúc?
9. Có level nào có dữ liệu chưa đủ để đưa vào summary không?
10. Độ khó tổng quan của từng normal level là gì?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_difficulty_summary` đại diện cho một normal level.

Có thể hiểu đơn giản:

```text id="4h266o"
Một dòng = một level_name
```

View này không có chiều ngày và không có chiều `app_version`.

---

## 5. Loại đối tượng

```text id="qvz50s"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc trực tiếp vào:

```text id="7m4bur"
vw_level_attempts
```

View chỉ lấy dữ liệu thỏa mãn:

```sql id="2rm3xz"
REGEXP_CONTAINS(level_name, r'^N\d+$')
AND is_near_tutorial_start = FALSE
```

Ý nghĩa:

* chỉ lấy normal level dạng `Nxxx`;
* loại các lượt bắt đầu level gần tutorial.

---

## 7. Điều kiện sample tối thiểu

View chỉ giữ các level có:

```sql id="2grj0q"
COUNT(*) >= 30
```

Điều này có nghĩa là level phải có ít nhất 30 lượt bắt đầu sạch mới xuất hiện trong view.

Khác với `vw_level_difficulty_daily`, view này không có cột `sample_size_status`, vì các level sample thấp đã bị loại bằng điều kiện `HAVING`.

---

## 8. Danh sách cột

| Cột                            | Kiểu dữ liệu | Ý nghĩa                                          |
| ------------------------------ | ------------ | ------------------------------------------------ |
| `level_name`                   | `STRING`     | Tên normal level, ví dụ `N001`.                  |
| `normal_start_count`           | `INT64`      | Tổng số lượt bắt đầu level sạch.                 |
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
| `max_duration_sec`             | `FLOAT64`    | Thời lượng lớn nhất của level.                   |
| `avg_move_used`                | `FLOAT64`    | Số move đã dùng trung bình.                      |
| `median_move_used`             | `FLOAT64`    | Trung vị số move đã dùng.                        |
| `avg_move_left`                | `FLOAT64`    | Số move còn lại trung bình.                      |
| `median_move_left`             | `FLOAT64`    | Trung vị số move còn lại.                        |
| `difficulty_band`              | `STRING`     | Nhóm độ khó tự động.                             |
| `analyst_note`                 | `STRING`     | Ghi chú diễn giải tự động bằng tiếng Việt.       |

---

## 9. Công thức chỉ số chính

### 9.1 Tỷ lệ ghép được `End_level`

```text id="8gv4ms"
matched_end_rate
=
matched_end_count
/
normal_start_count
```

Ý nghĩa: trong số lượt bắt đầu level, bao nhiêu lượt có sự kiện kết thúc tương ứng.

---

### 9.2 Tỷ lệ thắng

```text id="kwcvq1"
win_rate_among_matched_ends
=
win_count
/
matched_end_count
```

Ý nghĩa: trong các lượt có `End_level`, bao nhiêu lượt thắng.

---

### 9.3 Tỷ lệ thua

```text id="l7r85c"
fail_rate_among_matched_ends
=
fail_count
/
matched_end_count
```

Ý nghĩa: trong các lượt có `End_level`, bao nhiêu lượt thua.

---

## 10. Logic phân loại độ khó

Cột:

```text id="umne73"
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

Nếu `matched_end_rate` thấp, view ưu tiên cảnh báo rủi ro tracking.

---

## 11. Ghi chú tự động

Cột:

```text id="4pw3gq"
analyst_note
```

là ghi chú tiếng Việt tự động dựa trên tracking và win rate.

Các nhóm ghi chú chính:

| Điều kiện                | Ý nghĩa ghi chú                                                                          |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `matched_end_rate < 0.7` | Matched End thấp, cần kiểm tra tracking hoặc hành vi quit trước khi kết luận difficulty. |
| `win_rate <= 0.25`       | Rất khó, cần kiểm tra moves, target, obstacle hoặc tutorial/context trước level.         |
| `win_rate <= 0.45`       | Khó, có thể là difficulty spike nếu nằm trong early progression.                         |
| `win_rate <= 0.70`       | Trung bình, cần so với mục tiêu difficulty curve.                                        |
| `win_rate <= 0.85`       | Dễ, có thể phù hợp cho early game hoặc level hồi nhịp.                                   |
| còn lại                  | Rất dễ, cần kiểm tra có quá ít thử thách hoặc là level tutorial/relief không.            |

Trong tài liệu này:

* “difficulty” nghĩa là độ khó;
* “difficulty spike” nghĩa là điểm tăng độ khó đột ngột;
* “difficulty curve” nghĩa là đường cong độ khó theo tiến trình game;
* “relief level” nghĩa là level nhẹ hơn, dùng để giảm căng thẳng sau các level khó.

---

## 12. So sánh với `vw_level_difficulty_daily`

| Tiêu chí                      | `vw_level_difficulty_daily`  | `vw_level_difficulty_summary` |
| ----------------------------- | ---------------------------- | ----------------------------- |
| Độ chi tiết                   | `event_date + level_name`    | `level_name`                  |
| Có chia theo ngày không       | Có                           | Không                         |
| Có `sample_size_status` không | Có                           | Không                         |
| Có lọc sample thấp không      | Không loại, chỉ gắn nhãn     | Có loại bằng `COUNT(*) >= 30` |
| Mục đích                      | Theo dõi biến động hằng ngày | Xem tổng quan toàn bộ dữ liệu |
| Có `app_version` không        | Không                        | Không                         |

---

## 13. Kiểm tra chất lượng dữ liệu khuyến nghị

### 13.1 Kiểm tra tracking risk tổng hợp

```sql id="b46o8e"
SELECT
  level_name,
  normal_start_count,
  matched_end_count,
  start_without_end_count,
  matched_end_rate,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  difficulty_band = 'tracking_risk_do_not_conclude_strongly'

ORDER BY
  matched_end_rate ASC;
```

---

### 13.2 Xem level khó nhất

```sql id="zfo42d"
SELECT
  level_name,
  normal_start_count,
  matched_end_count,
  win_count,
  fail_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  avg_move_used,
  avg_move_left,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  difficulty_band IN ('very_hard', 'hard')

ORDER BY
  win_rate_among_matched_ends ASC,
  normal_start_count DESC;
```

---

### 13.3 Xem level rất dễ

```sql id="ljjobn"
SELECT
  level_name,
  normal_start_count,
  matched_end_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  avg_move_left,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  difficulty_band = 'very_easy'

ORDER BY
  win_rate_among_matched_ends DESC,
  normal_start_count DESC;
```

---

### 13.4 Kiểm tra level không xuất hiện do sample thấp

```sql id="i0nktc"
WITH all_clean_levels AS (
  SELECT
    level_name,
    COUNT(*) AS normal_start_count

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

  WHERE
    REGEXP_CONTAINS(level_name, r'^N\d+$')
    AND is_near_tutorial_start = FALSE

  GROUP BY
    level_name
)

SELECT
  all_clean_levels.level_name,
  all_clean_levels.normal_start_count

FROM
  all_clean_levels

LEFT JOIN
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary` AS summary
ON
  all_clean_levels.level_name = summary.level_name

WHERE
  summary.level_name IS NULL

ORDER BY
  all_clean_levels.normal_start_count DESC;
```

Truy vấn này giúp biết level nào bị loại khỏi summary vì chưa đủ 30 lượt bắt đầu sạch.

---

### 13.5 Kiểm tra tỷ lệ thắng và thua

```sql id="yepdmy"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  win_rate_among_matched_ends < 0
  OR win_rate_among_matched_ends > 1
  OR fail_rate_among_matched_ends < 0
  OR fail_rate_among_matched_ends > 1;
```

Kỳ vọng:

```text id="xcpwru"
Không trả ra dòng nào
```

---

## 14. Cách sử dụng khuyến nghị

### 14.1 Bảng tổng quan độ khó level

```sql id="j3n31c"
SELECT
  level_name,
  normal_start_count,
  matched_end_rate,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  avg_move_used,
  avg_move_left,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

ORDER BY
  level_name;
```

---

### 14.2 Xếp hạng level theo độ khó

```sql id="lkpl70"
SELECT
  level_name,
  normal_start_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  difficulty_band

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  difficulty_band != 'tracking_risk_do_not_conclude_strongly'

ORDER BY
  win_rate_among_matched_ends ASC,
  p90_duration_sec DESC;
```

---

### 14.3 Tìm level vừa khó vừa lâu

```sql id="h15195"
SELECT
  level_name,
  normal_start_count,
  win_rate_among_matched_ends,
  avg_duration_sec,
  p90_duration_sec,
  avg_move_used,
  difficulty_band,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`

WHERE
  difficulty_band IN ('very_hard', 'hard')
  AND p90_duration_sec >= 180

ORDER BY
  win_rate_among_matched_ends ASC,
  p90_duration_sec DESC;
```

Ngưỡng `180` giây chỉ là ví dụ. Team nên điều chỉnh theo mục tiêu thiết kế.

---

## 15. Lưu ý khi sử dụng

### 15.1 View không có `event_date`

View này là tổng hợp toàn bộ dữ liệu hiện có, không thể dùng để xem biến động theo ngày.

Nếu cần xem trend ngày, dùng:

```text id="1jzfr1"
vw_level_difficulty_daily
```

---

### 15.2 View không có `app_version`

View này không chia theo phiên bản game.

Nếu một level thay đổi thiết kế giữa nhiều phiên bản, view có thể gộp dữ liệu cũ và mới lại với nhau.

Khi cần đánh giá chính xác theo phiên bản, cần viết truy vấn bổ sung hoặc dùng view khác có chiều `app_version`.

---

### 15.3 View chỉ dùng dữ liệu không gần tutorial

Điều kiện:

```sql id="3jlqii"
is_near_tutorial_start = FALSE
```

giúp loại các lượt chơi gần tutorial.

Điều này tốt cho phân tích độ khó tự nhiên, nhưng cũng có nghĩa là các lượt tutorial không được tính.

---

### 15.4 View loại level có sample thấp

Điều kiện:

```sql id="xyzzbp"
HAVING COUNT(*) >= 30
```

giúp loại level có quá ít dữ liệu.

Nếu một level không xuất hiện trong view này, không nên kết luận là level đó không tồn tại. Cần kiểm tra sample bằng `vw_level_attempts`.

---

### 15.5 Tracking risk được ưu tiên hơn kết luận difficulty

Nếu:

```text id="dkm81s"
matched_end_rate < 0.7
```

view gán:

```text id="5k7i0z"
tracking_risk_do_not_conclude_strongly
```

Trong trường hợp này, cần kiểm tra tracking trước khi chỉnh độ khó level.

---

## 16. Rủi ro nếu view sai logic

| Lỗi logic                                       | Ảnh hưởng                                               |
| ----------------------------------------------- | ------------------------------------------------------- |
| Không loại tutorial                             | Độ khó bị nhiễu bởi hướng dẫn.                          |
| Lọc sai normal level                            | Thiếu hoặc thừa level trong summary.                    |
| Không có app version                            | Có thể gộp sai dữ liệu trước và sau khi level thay đổi. |
| Không có ngày                                   | Không thấy được biến động theo thời gian.               |
| Sai sample threshold                            | Có thể loại quá nhiều hoặc giữ quá nhiều level.         |
| Sai matched_end_rate                            | Nhầm tracking risk với difficulty.                      |
| Sai win_rate                                    | Phân loại độ khó sai.                                   |
| Dùng view này để quyết định tuning theo version | Có thể chỉnh sai nếu version mới khác version cũ.       |

---

## 17. Mức độ quan trọng

```text id="qohjrw"
Mức độ quan trọng: Trung bình đến cao
```

Lý do:

* view này hữu ích để xem tổng quan độ khó level;
* view này giúp có baseline ban đầu;
* view này loại sample thấp nên dễ đọc hơn daily view;
* tuy nhiên, do không có `event_date` và `app_version`, cần dùng thận trọng khi phân tích thay đổi theo thời gian hoặc theo phiên bản.

---

## 18. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="4v57mn"
sql/ddl/monitoring_level_analysis/vw_level_difficulty_summary.sql
```

Cấu trúc logic chính của view:

```sql id="hjp1ho"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_difficulty_summary`
AS
WITH level_summary AS (
  SELECT
    level_name,

    COUNT(*) AS normal_start_count,
    COUNT(end_time_utc) AS matched_end_count,
    COUNTIF(end_time_utc IS NULL) AS start_without_end_count,

    ROUND(
      SAFE_DIVIDE(COUNT(end_time_utc), COUNT(*)),
      4
    ) AS matched_end_rate,

    COUNTIF(success = TRUE) AS win_count,
    COUNTIF(success = FALSE) AS fail_count,

    ROUND(
      SAFE_DIVIDE(COUNTIF(success = TRUE), COUNT(end_time_utc)),
      4
    ) AS win_rate_among_matched_ends,

    ROUND(
      SAFE_DIVIDE(COUNTIF(success = FALSE), COUNT(end_time_utc)),
      4
    ) AS fail_rate_among_matched_ends,

    ROUND(AVG(duration_sec), 2) AS avg_duration_sec,
    ROUND(APPROX_QUANTILES(duration_sec, 100)[OFFSET(50)], 2)
      AS median_duration_sec,
    ROUND(APPROX_QUANTILES(duration_sec, 100)[OFFSET(75)], 2)
      AS p75_duration_sec,
    ROUND(APPROX_QUANTILES(duration_sec, 100)[OFFSET(90)], 2)
      AS p90_duration_sec,

    ROUND(AVG(move_used), 2) AS avg_move_used,
    ROUND(AVG(move_left), 2) AS avg_move_left

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

  WHERE
    REGEXP_CONTAINS(level_name, r'^N\d+$')
    AND is_near_tutorial_start = FALSE

  GROUP BY
    level_name

  HAVING
    COUNT(*) >= 30
)

SELECT
  *,
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
  level_summary;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: [[vw_level_attempts]]

- Downstream: Không có

