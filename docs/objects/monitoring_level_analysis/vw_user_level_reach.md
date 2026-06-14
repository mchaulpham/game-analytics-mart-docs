# vw_user_level_reach

## 1. Mục đích

`vw_user_level_reach` là view tổng hợp trạng thái người chơi ở từng level thường.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này trả lời câu hỏi:

```text id="he2wgw"
Mỗi người chơi đã từng tới level nào, thử bao nhiêu lần, đã thắng level đó chưa, và lần thắng đầu tiên xảy ra khi nào?
```

View này hoạt động ở cấp độ:

```text id="ee5xlr"
user_pseudo_id + level_name
```

Nghĩa là mỗi dòng mô tả một người chơi tại một level cụ thể.

---

## 2. Vai trò trong hệ thống

`vw_user_level_reach` thuộc nhóm:

```text id="ijijvq"
monitoring_level_analysis
```

Tức là nhóm theo dõi và phân tích level.

View này nằm sau:

```text id="9bqvk4"
vw_level_attempts
```

và được dùng bởi:

```text id="iokd5j"
vw_level_progression_version
vw_pipeline_health_daily
```

Vai trò chính:

* xác định người chơi đã từng tới level nào;
* xác định người chơi đã thắng level nào;
* tính số lần thử theo người chơi và level;
* tạo nền tảng cho phân tích tiến trình theo phiên bản game;
* hỗ trợ kiểm tra dữ liệu người chơi đã đủ 24 giờ để xem có đi tiếp hay không.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Một người chơi đã từng tới những level nào?
2. Một người chơi thử một level bao nhiêu lần?
3. Người chơi đã thắng level đó chưa?
4. Người chơi thắng level ở lần thử nào?
5. Thời điểm đầu tiên người chơi bắt đầu level là khi nào?
6. Thời điểm đầu tiên người chơi thắng level là khi nào?
7. Có bao nhiêu người chơi tới level nhưng chưa thắng?
8. Có bao nhiêu người chơi có `Start_level` nhưng không có `End_level`?
9. Level nào có nhiều người thử nhiều lần?
10. Dữ liệu của người chơi đã đủ 24 giờ để kiểm tra next level chưa?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_user_level_reach` đại diện cho một người chơi tại một level thường.

Có thể hiểu đơn giản:

```text id="l5mq32"
Một dòng = một user_pseudo_id + một level_name
```

Ví dụ:

```text id="2uzd38"
user_pseudo_id = A
level_name = N003
```

nghĩa là dòng đó mô tả trạng thái của người chơi A tại level `N003`.

---

## 5. Loại đối tượng

```text id="bpyw9u"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc trực tiếp vào:

```text id="sq0cee"
vw_level_attempts
```

View chỉ lấy các level có dạng:

```text id="ivj001"
N001
N002
N003
...
```

Điều kiện trong logic:

```sql id="l8hpxs"
REGEXP_CONTAINS(level_name, r'^N\d+$')
```

Điều này có nghĩa là view chỉ phân tích “normal level” theo quy ước tên `Nxxx`.

Trong tài liệu này, “normal level” nghĩa là level chính có tên bắt đầu bằng `N` và theo sau là số, ví dụ `N001`.

Các level khác như `c001`, `E1`, hoặc level đặc biệt không được đưa vào view này.

---

## 7. Các view đang sử dụng view này

`vw_user_level_reach` được sử dụng bởi:

| View sử dụng                   | Mục đích sử dụng                                          |
| ------------------------------ | --------------------------------------------------------- |
| `vw_level_progression_version` | Tổng hợp tiến trình người chơi theo level và app version. |
| `vw_pipeline_health_daily`     | Kiểm tra sức khỏe dữ liệu progression và coverage.        |

---

## 8. Danh sách cột

| Cột                                       | Kiểu dữ liệu | Ý nghĩa                                                                     |
| ----------------------------------------- | ------------ | --------------------------------------------------------------------------- |
| `user_pseudo_id`                          | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                              |
| `level_name`                              | `STRING`     | Tên level, chỉ gồm level thường dạng `Nxxx`.                                |
| `level_number`                            | `INT64`      | Số level được tách từ `level_name`, ví dụ `N003` thành `3`.                 |
| `first_start_event_id`                    | `STRING`     | Mã sự kiện bắt đầu level đầu tiên của người chơi tại level này.             |
| `first_start_event_table_suffix`          | `STRING`     | Hậu tố bảng GA4 chứa lần bắt đầu đầu tiên.                                  |
| `first_start_event_date`                  | `STRING`     | Ngày của lần bắt đầu level đầu tiên.                                        |
| `first_start_time_utc`                    | `TIMESTAMP`  | Thời điểm bắt đầu level đầu tiên theo múi giờ UTC.                          |
| `first_start_app_version`                 | `STRING`     | Phiên bản game tại lần bắt đầu level đầu tiên.                              |
| `first_start_is_near_tutorial_start`      | `BOOL`       | Lần bắt đầu đầu tiên có gần tutorial hay không.                             |
| `first_start_nearby_tutorial_names`       | `STRING`     | Tên tutorial gần lần bắt đầu đầu tiên, nếu có.                              |
| `first_start_nearby_tutorial_start_count` | `INT64`      | Số sự kiện `Tutorial_start` gần lần bắt đầu đầu tiên.                       |
| `attempt_count`                           | `INT64`      | Tổng số lần người chơi bắt đầu level này.                                   |
| `matched_end_count`                       | `INT64`      | Số lần bắt đầu level ghép được với `End_level`.                             |
| `start_without_end_count`                 | `INT64`      | Số lần bắt đầu level không ghép được với `End_level`.                       |
| `win_attempt_count`                       | `INT64`      | Số lượt chơi kết thúc bằng thắng.                                           |
| `fail_attempt_count`                      | `INT64`      | Số lượt chơi kết thúc bằng thua.                                            |
| `has_won_level`                           | `BOOL`       | Người chơi đã từng thắng level này hay chưa.                                |
| `first_win_start_event_id`                | `STRING`     | Mã sự kiện bắt đầu của lần thắng đầu tiên.                                  |
| `first_win_start_event_table_suffix`      | `STRING`     | Hậu tố bảng chứa sự kiện bắt đầu của lần thắng đầu tiên.                    |
| `first_win_start_event_date`              | `STRING`     | Ngày bắt đầu của lần thắng đầu tiên.                                        |
| `first_win_start_time_utc`                | `TIMESTAMP`  | Thời điểm bắt đầu lượt thắng đầu tiên.                                      |
| `first_win_start_app_version`             | `STRING`     | Phiên bản game tại lúc bắt đầu lượt thắng đầu tiên.                         |
| `first_win_end_event_table_suffix`        | `STRING`     | Hậu tố bảng chứa `End_level` của lần thắng đầu tiên.                        |
| `first_win_end_event_date`                | `STRING`     | Ngày kết thúc của lần thắng đầu tiên.                                       |
| `first_win_time_utc`                      | `TIMESTAMP`  | Thời điểm kết thúc thắng đầu tiên.                                          |
| `first_win_end_app_version`               | `STRING`     | Phiên bản game tại lúc kết thúc thắng đầu tiên.                             |
| `first_win_attempt_no`                    | `INT64`      | Số lần thử của lần thắng đầu tiên.                                          |
| `first_win_move_used`                     | `INT64`      | Số bước đã dùng trong lần thắng đầu tiên.                                   |
| `first_win_move_left`                     | `INT64`      | Số bước còn lại trong lần thắng đầu tiên.                                   |
| `first_win_duration_sec`                  | `FLOAT64`    | Thời lượng level trong lần thắng đầu tiên.                                  |
| `first_win_seconds_from_start_to_end`     | `INT64`      | Số giây từ bắt đầu đến kết thúc trong lần thắng đầu tiên.                   |
| `latest_start_time_utc`                   | `TIMESTAMP`  | Thời điểm `Start_level` mới nhất trong dữ liệu level thường.                |
| `maturity_cutoff_time_utc`                | `TIMESTAMP`  | Mốc 24 giờ sau lần đầu người chơi bắt đầu level này.                        |
| `is_matured_24h_for_next_level_check`     | `BOOL`       | Cho biết đã đủ 24 giờ để kiểm tra người chơi có đi tiếp level sau hay chưa. |

---

## 9. Logic xử lý chính

### 9.1 Lọc normal level

View chỉ lấy level có tên khớp mẫu:

```sql id="sdxk8m"
REGEXP_CONTAINS(level_name, r'^N\d+$')
```

Sau đó tách số level bằng:

```sql id="et4mul"
CAST(REGEXP_EXTRACT(level_name, r'^N(\d+)$') AS INT64)
```

Ví dụ:

| `level_name` | `level_number` |
| ------------ | -------------: |
| `N001`       |              1 |
| `N002`       |              2 |
| `N010`       |             10 |

---

### 9.2 Lấy lần bắt đầu đầu tiên

Với mỗi người chơi và level, view lấy lần bắt đầu đầu tiên bằng cách sắp xếp:

```text id="g7k6g9"
start_time_utc ASC
start_event_id ASC
```

và lấy dòng đầu tiên.

Các thông tin được giữ lại gồm:

```text id="qybdkf"
first_start_event_id
first_start_event_date
first_start_time_utc
first_start_app_version
first_start_is_near_tutorial_start
first_start_nearby_tutorial_names
```

---

### 9.3 Tổng hợp số lần thử

Với mỗi người chơi và level, view tính:

```text id="j3u7b0"
attempt_count
matched_end_count
start_without_end_count
win_attempt_count
fail_attempt_count
```

Trong đó:

* `attempt_count` là tổng số dòng từ `vw_level_attempts`;
* `matched_end_count` là số lượt có `end_time_utc`;
* `start_without_end_count` là số lượt không có `end_time_utc`;
* `win_attempt_count` là số lượt có `success = TRUE`;
* `fail_attempt_count` là số lượt có `success = FALSE`.

---

### 9.4 Xác định đã thắng level hay chưa

Cột:

```text id="s48tfj"
has_won_level
```

được tính bằng:

```sql id="oee9bc"
win_attempt_count > 0
```

Nếu người chơi có ít nhất một lượt thắng, `has_won_level = TRUE`.

---

### 9.5 Lấy lần thắng đầu tiên

View tìm lần thắng đầu tiên của mỗi người chơi tại mỗi level bằng điều kiện:

```sql id="v6a8v7"
success = TRUE
```

Sau đó sắp xếp:

```text id="75ih6k"
end_time_utc ASC
start_time_utc ASC
start_event_id ASC
```

và lấy dòng đầu tiên.

Các trường được giữ lại gồm:

```text id="lf9ii3"
first_win_start_time_utc
first_win_time_utc
first_win_attempt_no
first_win_move_used
first_win_move_left
first_win_duration_sec
first_win_seconds_from_start_to_end
```

---

### 9.6 Tính mốc trưởng thành 24 giờ

View tính:

```text id="2tn5l9"
maturity_cutoff_time_utc = first_start_time_utc + 24 giờ
```

Sau đó so sánh với:

```text id="74j9ya"
latest_start_time_utc
```

để tạo cột:

```text id="2rzmds"
is_matured_24h_for_next_level_check
```

Logic:

```sql id="unrmtf"
maturity_cutoff_time_utc <= latest_start_time_utc
```

Lưu ý quan trọng: view này dùng `latest_start_time_utc` trong dữ liệu làm mốc so sánh, không dùng `CURRENT_TIMESTAMP()`.

Điều này giúp đánh giá trưởng thành dựa trên phạm vi dữ liệu hiện có, phù hợp hơn cho kiểm tra progression khi dữ liệu raw có thể chưa cập nhật đến hiện tại.

---

## 10. Kiểm tra chất lượng dữ liệu khuyến nghị

### 10.1 Kiểm tra trùng dòng user + level

```sql id="d97lln"
SELECT
  user_pseudo_id,
  level_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

GROUP BY
  user_pseudo_id,
  level_name

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="sf4jmw"
Không trả ra dòng nào
```

Vì mỗi người chơi và mỗi level chỉ nên có một dòng.

---

### 10.2 Kiểm tra tính nhất quán của attempt count

```sql id="epwq9p"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

WHERE
  attempt_count != matched_end_count + start_without_end_count;
```

Kỳ vọng:

```text id="cd4yq8"
Không trả ra dòng nào
```

Vì tổng lượt có kết thúc và lượt không có kết thúc phải bằng tổng lượt bắt đầu.

---

### 10.3 Kiểm tra số thắng/thua không vượt quá matched end

```sql id="wcsopk"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

WHERE
  win_attempt_count + fail_attempt_count > matched_end_count;
```

Kỳ vọng:

```text id="sywtt0"
Không trả ra dòng nào
```

Nếu có kết quả, cần kiểm tra dữ liệu `success`.

---

### 10.4 Kiểm tra `has_won_level` và first win

```sql id="j6jbjl"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

WHERE
  (
    has_won_level IS TRUE
    AND first_win_time_utc IS NULL
  )
  OR (
    has_won_level IS FALSE
    AND first_win_time_utc IS NOT NULL
  );
```

Kỳ vọng:

```text id="mh1s9j"
Không trả ra dòng nào
```

---

### 10.5 Kiểm tra level_number bị thiếu

```sql id="4iizkm"
SELECT
  level_name,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

WHERE
  level_number IS NULL

GROUP BY
  level_name

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="8hzv22"
Không trả ra dòng nào
```

Vì view đã lọc level dạng `Nxxx`.

---

## 11. Cách sử dụng khuyến nghị

### 11.1 Xem level cao nhất mỗi người chơi từng tới

```sql id="oxla13"
SELECT
  user_pseudo_id,
  MAX(level_number) AS max_level_number_reached

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

GROUP BY
  user_pseudo_id

ORDER BY
  max_level_number_reached DESC;
```

---

### 11.2 Đếm số người từng tới và từng thắng mỗi level

```sql id="n9bif7"
SELECT
  level_name,
  level_number,

  COUNT(DISTINCT user_pseudo_id) AS reached_user_count,

  COUNT(DISTINCT IF(
    has_won_level IS TRUE,
    user_pseudo_id,
    NULL
  )) AS won_user_count,

  ROUND(
    SAFE_DIVIDE(
      COUNT(DISTINCT IF(has_won_level IS TRUE, user_pseudo_id, NULL)),
      COUNT(DISTINCT user_pseudo_id)
    ),
    4
  ) AS user_win_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

GROUP BY
  level_name,
  level_number

ORDER BY
  level_number;
```

---

### 11.3 Tìm người chơi tới level nhưng chưa thắng

```sql id="qqdlcy"
SELECT
  user_pseudo_id,
  level_name,
  level_number,
  first_start_time_utc,
  attempt_count,
  matched_end_count,
  start_without_end_count,
  win_attempt_count,
  fail_attempt_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

WHERE
  has_won_level IS FALSE

ORDER BY
  level_number,
  attempt_count DESC;
```

---

### 11.4 Tính số lần thử trung bình theo người chơi và level

```sql id="44u7ej"
SELECT
  level_name,
  level_number,

  COUNT(DISTINCT user_pseudo_id) AS reached_user_count,

  AVG(attempt_count) AS avg_attempt_count_per_user,

  APPROX_QUANTILES(attempt_count, 100)[OFFSET(50)]
    AS median_attempt_count_per_user,

  APPROX_QUANTILES(attempt_count, 100)[OFFSET(90)]
    AS p90_attempt_count_per_user

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`

GROUP BY
  level_name,
  level_number

ORDER BY
  level_number;
```

---

## 12. Lưu ý khi sử dụng

### 12.1 View này chỉ bao gồm normal level dạng `Nxxx`

Các level không có dạng `Nxxx` không được đưa vào view này.

Ví dụ các level hoặc event group sau có thể bị loại:

```text id="lt5rn2"
c001
c010
E1
E5
```

Nếu cần phân tích các level đặc biệt, không dùng view này làm nguồn duy nhất.

---

### 12.2 View này không phải bảng attempt-level

View này đã tổng hợp theo người chơi và level.

Nếu cần xem từng lượt chơi, dùng:

```text id="v584sp"
vw_level_attempts
```

---

### 12.3 Cờ tutorial chỉ lấy từ lần bắt đầu đầu tiên

Các cột:

```text id="02l4ng"
first_start_is_near_tutorial_start
first_start_nearby_tutorial_names
first_start_nearby_tutorial_start_count
```

chỉ mô tả lần bắt đầu level đầu tiên của người chơi.

Nếu các lần thử sau có tutorial hoặc không có tutorial, view này không phản ánh chi tiết từng attempt.

---

### 12.4 Dữ liệu bắt đầu level được lấy từ `vw_level_attempts`

View này dựa trên `vw_level_attempts`.

Do đó, nó phản ánh các `Start_level` thật đã đi qua pipeline attempt. Nó không trực tiếp dùng `vw_level_start_eff`.

Khi so sánh với các view phễu dùng `vw_level_start_eff`, cần hiểu khác biệt này.

---

### 12.5 Mốc trưởng thành 24 giờ dựa trên dữ liệu mới nhất

Cột:

```text id="hx1y5l"
is_matured_24h_for_next_level_check
```

dựa trên `latest_start_time_utc` trong dữ liệu, không dựa trên thời gian hiện tại.

Điều này giúp tránh việc đánh giá một user là đã đủ 24 giờ nếu dữ liệu thực tế chưa được cập nhật đến thời điểm đó.

---

## 13. Rủi ro nếu view sai logic

| Lỗi logic                    | Ảnh hưởng                                                             |
| ---------------------------- | --------------------------------------------------------------------- |
| Lọc sai normal level         | Thiếu hoặc thừa level trong progression.                              |
| Sai lần bắt đầu đầu tiên     | Sai mốc reach level.                                                  |
| Sai lần thắng đầu tiên       | Sai phân tích người chơi đã vượt qua level.                           |
| Sai attempt count            | Sai đánh giá retry pressure.                                          |
| Sai `has_won_level`          | Sai toàn bộ phân tích progression.                                    |
| Không hiểu view đã aggregate | Dễ dùng sai khi cần dữ liệu từng lượt chơi.                           |
| Dùng sai cờ tutorial         | Có thể tưởng mọi attempt đều có cùng trạng thái tutorial như lần đầu. |
| Mốc 24 giờ sai               | Sai phân tích đi tiếp level sau.                                      |

---

## 14. Mức độ quan trọng

```text id="w2rodo"
Mức độ quan trọng: Cao
```

Lý do:

* view này là nguồn chính để biết người chơi đã tới và thắng từng normal level;
* view này là đầu vào của phân tích progression theo app version;
* view này hỗ trợ kiểm tra người chơi đã đủ 24 giờ để đánh giá next level;
* view này gom thông tin attempt ở cấp người chơi, rất hữu ích cho phân tích tiến trình.

---

## 15. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="1s4s50"
sql/ddl/monitoring_level_analysis/vw_user_level_reach.sql
```

Cấu trúc logic chính của view:

```sql id="vfamzn"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`
AS
WITH normal_level_attempts AS (
  SELECT
    *,
    CAST(REGEXP_EXTRACT(level_name, r'^N(\d+)$') AS INT64)
      AS level_number

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_attempts`

  WHERE
    REGEXP_CONTAINS(level_name, r'^N\d+$')
),

latest_data AS (
  SELECT
    MAX(start_time_utc) AS latest_start_time_utc

  FROM
    normal_level_attempts
),

user_level_agg AS (
  SELECT
    user_pseudo_id,
    level_name,
    level_number,

    -- lấy lần bắt đầu đầu tiên
    -- đếm attempt, matched end, start without end
    -- đếm win/fail

  FROM
    normal_level_attempts

  GROUP BY
    user_pseudo_id,
    level_name,
    level_number
),

first_win_details AS (
  SELECT
    -- lấy lần thắng đầu tiên của mỗi user + level

  FROM
    normal_level_attempts

  WHERE
    success = TRUE

  QUALIFY
    ROW_NUMBER() OVER (
      PARTITION BY user_pseudo_id, level_name
      ORDER BY end_time_utc ASC, start_time_utc ASC, start_event_id ASC
    ) = 1
)

SELECT
  -- thông tin user + level
  -- thông tin lần bắt đầu đầu tiên
  -- tổng hợp attempt
  -- thông tin lần thắng đầu tiên
  -- mốc dữ liệu mới nhất
  -- mốc trưởng thành 24 giờ

FROM
  user_level_agg

CROSS JOIN
  latest_data

LEFT JOIN
  first_win_details
ON
  user_level_agg.user_pseudo_id = first_win_details.user_pseudo_id
  AND user_level_agg.level_name = first_win_details.level_name;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: [[vw_level_attempts]]

- Downstream:
	- [[vw_level_progression_version]]
	- [[vw_pipeline_health_daily]]

