# vw_level_progression_version

## 1. Mục đích

`vw_level_progression_version` là view tổng hợp tiến trình người chơi từ một level sang level kế tiếp, theo phiên bản game.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này trả lời câu hỏi:

```text id="6q2xjk"
Theo từng app_version, người chơi đã tới level hiện tại có tiếp tục sang level kế tiếp không?
```

View tập trung vào chuyển tiếp giữa hai normal level liên tiếp:

```text id="gi22u4"
N001 → N002
N002 → N003
N003 → N004
...
```

Trong tài liệu này, “normal level” nghĩa là level chính có tên dạng `Nxxx`, ví dụ `N001`, `N002`, `N010`.

---

## 2. Vai trò trong hệ thống

`vw_level_progression_version` thuộc nhóm:

```text id="i8ju66"
monitoring_level_analysis
```

Tức là nhóm theo dõi và phân tích level.

View này nằm sau:

```text id="tx7diz"
vw_user_level_reach
```

và được dùng bởi:

```text id="t0otf2"
vw_pipeline_health_daily
```

Vai trò chính:

* đo người chơi có đi từ level hiện tại sang level kế tiếp không;
* đo rơi sau khi bắt đầu level;
* đo rơi sau khi đã thắng level;
* theo dõi progression theo `app_version`;
* kiểm tra ảnh hưởng của tutorial ở lần bắt đầu đầu tiên;
* đo thời gian từ level hiện tại sang level kế tiếp.

Trong tài liệu này, “progression” nghĩa là tiến trình người chơi đi qua các level theo thứ tự.

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Theo từng app version, bao nhiêu người chơi tới mỗi level?
2. Bao nhiêu người đã đủ 24 giờ để kiểm tra có đi tiếp level sau không?
3. Bao nhiêu người đi tiếp từ level hiện tại sang level kế tiếp?
4. Tỷ lệ rơi từ start level hiện tại sang start level kế tiếp là bao nhiêu?
5. Bao nhiêu người đã thắng level hiện tại?
6. Trong số người đã thắng, bao nhiêu người đi tiếp level sau?
7. Tỷ lệ rơi sau khi thắng level là bao nhiêu?
8. Lần bắt đầu đầu tiên của level có bị ảnh hưởng bởi tutorial không?
9. Người chơi mất bao lâu từ lần đầu bắt đầu level hiện tại đến lần đầu bắt đầu level sau?
10. Người chơi mất bao lâu từ lần thắng đầu tiên đến lần đầu bắt đầu level sau?
11. App version nào có progression yếu ở một level cụ thể?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_level_progression_version` đại diện cho một level trong một phiên bản game.

Có thể hiểu đơn giản:

```text id="t0mu6h"
Một dòng = một app_version + một level_name
```

Dòng đó mô tả việc người chơi từ `level_name` có đi tiếp sang `next_level_name` hay không.

---

## 5. Loại đối tượng

```text id="v5pu0i"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc trực tiếp vào:

```text id="l0vg1o"
vw_user_level_reach
```

`vw_user_level_reach` đã tổng hợp trạng thái người chơi ở từng normal level, gồm:

* lần đầu bắt đầu level;
* số lần thử;
* số lần thắng/thua;
* lần thắng đầu tiên;
* trạng thái đã đủ 24 giờ để kiểm tra level sau.

`vw_level_progression_version` dùng dữ liệu này để tạo chuyển tiếp giữa level hiện tại và level kế tiếp.

---

## 7. Cách xác định level kế tiếp

View xác định `next_level_name` bằng cách lấy `level_number + 1`.

Công thức logic:

```sql id="oaa7vq"
CONCAT(
  'N',
  LPAD(CAST(level_number + 1 AS STRING), 3, '0')
)
```

Ví dụ:

| `level_name` | `level_number` | `next_level_name` |
| ------------ | -------------: | ----------------- |
| `N001`       |              1 | `N002`            |
| `N002`       |              2 | `N003`            |
| `N010`       |             10 | `N011`            |

Lưu ý: view này suy luận level kế tiếp theo số thứ tự `Nxxx`, không dùng `dim_level_seq` hoặc `dim_f10_level_map`.

---

## 8. Logic chuyển tiếp giữa level hiện tại và level sau

View tự nối `vw_user_level_reach` với chính nó.

Điều kiện nối:

```text id="1v49x4"
cùng user_pseudo_id
level_number hiện tại + 1 = level_number tiếp theo
next_level.first_start_time_utc > current_level.first_start_time_utc
```

Điều này giúp xác định người chơi có bắt đầu level kế tiếp sau khi đã bắt đầu level hiện tại hay không.

---

## 9. Logic đi tiếp sau khi thắng

Người chơi được xem là đi tiếp sau khi thắng nếu:

```text id="ee3wsw"
current_level.has_won_level = TRUE
next_level.first_start_time_utc IS NOT NULL
next_level.first_start_time_utc >= current_level.first_win_time_utc
```

Cột liên quan:

```text id="cb76sm"
has_reached_next_level_after_current_win
```

Ở kết quả tổng hợp, chỉ số này trở thành:

```text id="3bkpxc"
next_level_reached_after_win_user_count
win_to_next_start_rate
win_to_next_start_drop_rate
```

---

## 10. Logic trưởng thành 24 giờ

View chỉ tính người chơi vào mẫu số chuyển tiếp nếu:

```text id="o0q5bz"
is_matured_24h_for_next_level_check = TRUE
```

Trường này đến từ `vw_user_level_reach`.

Nó cho biết đã đủ 24 giờ từ lần đầu người chơi bắt đầu level hiện tại để kiểm tra việc người đó có đi tiếp level sau hay không.

Lưu ý: trạng thái trưởng thành 24 giờ này dựa trên `latest_start_time_utc` trong dữ liệu level, không dựa trực tiếp vào `CURRENT_TIMESTAMP()`.

---

## 11. Danh sách cột

| Cột                                             | Kiểu dữ liệu | Ý nghĩa                                                                                 |
| ----------------------------------------------- | ------------ | --------------------------------------------------------------------------------------- |
| `app_version`                                   | `STRING`     | Phiên bản game tại lần đầu người chơi bắt đầu level hiện tại.                           |
| `level_number`                                  | `INT64`      | Số thứ tự của level hiện tại.                                                           |
| `level_name`                                    | `STRING`     | Tên level hiện tại.                                                                     |
| `next_level_name`                               | `STRING`     | Tên level kế tiếp được suy luận từ `level_number + 1`.                                  |
| `reached_user_count`                            | `INT64`      | Số người chơi từng tới level hiện tại.                                                  |
| `matured_reached_user_count`                    | `INT64`      | Số người chơi đã đủ 24 giờ để kiểm tra có đi tiếp level sau hay không.                  |
| `not_matured_reached_user_count`                | `INT64`      | Số người chơi chưa đủ 24 giờ.                                                           |
| `won_user_count`                                | `INT64`      | Số người chơi từng thắng level hiện tại.                                                |
| `matured_won_user_count`                        | `INT64`      | Số người chơi đã thắng level hiện tại và đã đủ 24 giờ.                                  |
| `next_level_reached_user_count`                 | `INT64`      | Số người chơi đã đủ 24 giờ và có bắt đầu level kế tiếp sau khi bắt đầu level hiện tại.  |
| `start_to_next_start_drop_user_count`           | `INT64`      | Số người chơi đã đủ 24 giờ nhưng không bắt đầu level kế tiếp.                           |
| `start_to_next_start_rate`                      | `FLOAT64`    | Tỷ lệ người chơi đi từ start level hiện tại sang start level kế tiếp.                   |
| `start_to_next_start_drop_rate`                 | `FLOAT64`    | Tỷ lệ người chơi rơi từ start level hiện tại sang start level kế tiếp.                  |
| `next_level_reached_after_win_user_count`       | `INT64`      | Số người chơi đã thắng level hiện tại và có bắt đầu level kế tiếp sau thời điểm thắng.  |
| `win_to_next_start_drop_user_count`             | `INT64`      | Số người chơi đã thắng level hiện tại nhưng không bắt đầu level kế tiếp.                |
| `win_to_next_start_rate`                        | `FLOAT64`    | Tỷ lệ người chơi đi tiếp level sau khi thắng level hiện tại.                            |
| `win_to_next_start_drop_rate`                   | `FLOAT64`    | Tỷ lệ người chơi rơi sau khi thắng level hiện tại.                                      |
| `first_start_near_tutorial_user_count`          | `INT64`      | Số người chơi có lần bắt đầu đầu tiên của level gần tutorial.                           |
| `first_start_non_tutorial_user_count`           | `INT64`      | Số người chơi có lần bắt đầu đầu tiên của level không gần tutorial.                     |
| `first_start_near_tutorial_rate`                | `FLOAT64`    | Tỷ lệ người chơi có lần bắt đầu đầu tiên gần tutorial.                                  |
| `total_attempt_count`                           | `INT64`      | Tổng số lượt thử của người chơi ở level hiện tại.                                       |
| `total_matched_end_count`                       | `INT64`      | Tổng số lượt thử có `End_level`.                                                        |
| `total_start_without_end_count`                 | `INT64`      | Tổng số lượt thử không có `End_level`.                                                  |
| `total_win_attempt_count`                       | `INT64`      | Tổng số lượt thắng.                                                                     |
| `total_fail_attempt_count`                      | `INT64`      | Tổng số lượt thua.                                                                      |
| `avg_attempt_count_per_reached_user`            | `FLOAT64`    | Số lần thử trung bình trên mỗi người chơi đã tới level.                                 |
| `median_seconds_from_first_start_to_next_start` | `INT64`      | Trung vị thời gian từ lần đầu bắt đầu level hiện tại đến lần đầu bắt đầu level kế tiếp. |
| `p75_seconds_from_first_start_to_next_start`    | `INT64`      | Mốc 75% thời gian từ lần đầu bắt đầu level hiện tại đến lần đầu bắt đầu level kế tiếp.  |
| `median_seconds_from_first_win_to_next_start`   | `INT64`      | Trung vị thời gian từ lần thắng đầu tiên đến lần đầu bắt đầu level kế tiếp.             |
| `p75_seconds_from_first_win_to_next_start`      | `INT64`      | Mốc 75% thời gian từ lần thắng đầu tiên đến lần đầu bắt đầu level kế tiếp.              |
| `sample_size_status`                            | `STRING`     | Trạng thái kích thước mẫu.                                                              |
| `analyst_note`                                  | `STRING`     | Ghi chú phân tích tự động bằng tiếng Việt.                                              |

---

## 12. Công thức chỉ số chính

### 12.1 Tỷ lệ đi tiếp từ start level hiện tại sang start level kế tiếp

```text id="vc7xgf"
start_to_next_start_rate
=
next_level_reached_user_count
/
matured_reached_user_count
```

Ý nghĩa: trong số người đã tới level hiện tại và đã đủ 24 giờ, bao nhiêu người bắt đầu level kế tiếp.

---

### 12.2 Tỷ lệ rơi từ start level hiện tại sang start level kế tiếp

```text id="sj8uoy"
start_to_next_start_drop_rate
=
start_to_next_start_drop_user_count
/
matured_reached_user_count
```

Trong đó:

```text id="qpo2m0"
start_to_next_start_drop_user_count
=
matured_reached_user_count - next_level_reached_user_count
```

---

### 12.3 Tỷ lệ đi tiếp sau khi thắng

```text id="swogjk"
win_to_next_start_rate
=
next_level_reached_after_win_user_count
/
matured_won_user_count
```

Ý nghĩa: trong số người đã thắng level hiện tại và đã đủ 24 giờ, bao nhiêu người bắt đầu level kế tiếp sau khi thắng.

---

### 12.4 Tỷ lệ rơi sau khi thắng

```text id="2h1gmj"
win_to_next_start_drop_rate
=
win_to_next_start_drop_user_count
/
matured_won_user_count
```

Trong đó:

```text id="2hc2o7"
win_to_next_start_drop_user_count
=
matured_won_user_count - next_level_reached_after_win_user_count
```

---

### 12.5 Tỷ lệ lần đầu bắt đầu gần tutorial

```text id="6c2jpi"
first_start_near_tutorial_rate
=
first_start_near_tutorial_user_count
/
reached_user_count
```

Ý nghĩa: trong số người đã tới level hiện tại, bao nhiêu người có lần đầu bắt đầu level gần tutorial.

---

## 13. Logic phân loại sample

Cột:

```text id="68sam0"
sample_size_status
```

được tính theo `matured_reached_user_count`.

| Điều kiện                          | Nhãn                |
| ---------------------------------- | ------------------- |
| `matured_reached_user_count >= 30` | `sufficient_sample` |
| còn lại                            | `low_sample`        |

Nếu sample thấp, không nên kết luận mạnh về drop giữa level.

---

## 14. Logic ghi chú tự động

Cột:

```text id="vepbk5"
analyst_note
```

được tạo theo thứ tự ưu tiên sau:

| Điều kiện                              | Ghi chú                                                                         |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| `sample_size_status = 'low_sample'`    | Sample thấp, chỉ nên xem xu hướng.                                              |
| `start_to_next_start_drop_rate >= 0.5` | Drop từ start level hiện tại sang start level kế tiếp cao.                      |
| `win_to_next_start_drop_rate >= 0.3`   | Người chơi đã thắng nhưng không tiếp tục sang level kế tiếp ở tỷ lệ đáng chú ý. |
| còn lại                                | Progression nhìn ổn ở mức tổng quan.                                            |

Lưu ý: view kiểm tra `start_to_next_start_drop_rate >= 0.5` trước `win_to_next_start_drop_rate >= 0.3`.

Vì vậy, nếu cả hai điều kiện đều đúng, ghi chú sẽ ưu tiên thông báo về drop từ start sang next.

---

## 15. Kiểm tra chất lượng dữ liệu khuyến nghị

### 15.1 Kiểm tra tỷ lệ trong khoảng 0 đến 1

```sql id="gqgfht"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  start_to_next_start_rate < 0
  OR start_to_next_start_rate > 1
  OR start_to_next_start_drop_rate < 0
  OR start_to_next_start_drop_rate > 1
  OR win_to_next_start_rate < 0
  OR win_to_next_start_rate > 1
  OR win_to_next_start_drop_rate < 0
  OR win_to_next_start_drop_rate > 1
  OR first_start_near_tutorial_rate < 0
  OR first_start_near_tutorial_rate > 1;
```

Kỳ vọng:

```text id="xw5rln"
Không trả ra dòng nào
```

---

### 15.2 Kiểm tra tổng start-to-next

```sql id="wqt0qu"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  matured_reached_user_count
    != next_level_reached_user_count + start_to_next_start_drop_user_count;
```

Kỳ vọng:

```text id="xqc3yl"
Không trả ra dòng nào
```

---

### 15.3 Kiểm tra tổng win-to-next

```sql id="bkeiyt"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  matured_won_user_count
    != next_level_reached_after_win_user_count + win_to_next_start_drop_user_count;
```

Kỳ vọng:

```text id="8nku9g"
Không trả ra dòng nào
```

---

### 15.4 Kiểm tra level có drop cao từ start sang next

```sql id="y1200a"
SELECT
  app_version,
  level_number,
  level_name,
  next_level_name,

  matured_reached_user_count,
  next_level_reached_user_count,
  start_to_next_start_drop_user_count,
  start_to_next_start_drop_rate,

  sample_size_status,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  sample_size_status = 'sufficient_sample'
  AND start_to_next_start_drop_rate >= 0.5

ORDER BY
  start_to_next_start_drop_rate DESC;
```

---

### 15.5 Kiểm tra level có drop cao sau khi thắng

```sql id="pbu17o"
SELECT
  app_version,
  level_number,
  level_name,
  next_level_name,

  matured_won_user_count,
  next_level_reached_after_win_user_count,
  win_to_next_start_drop_user_count,
  win_to_next_start_drop_rate,

  sample_size_status,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  sample_size_status = 'sufficient_sample'
  AND win_to_next_start_drop_rate >= 0.3

ORDER BY
  win_to_next_start_drop_rate DESC;
```

---

### 15.6 Kiểm tra tỷ lệ tutorial ở lần start đầu tiên

```sql id="wz2mne"
SELECT
  app_version,
  level_number,
  level_name,

  reached_user_count,
  first_start_near_tutorial_user_count,
  first_start_near_tutorial_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

ORDER BY
  first_start_near_tutorial_rate DESC;
```

Nếu tỷ lệ này cao, cần cẩn thận khi đọc progression của level đó vì lần đầu vào level có thể bị tutorial ảnh hưởng.

---

## 16. Cách sử dụng khuyến nghị

### 16.1 Theo dõi progression theo app version

```sql id="ac252q"
SELECT
  app_version,
  level_number,
  level_name,
  next_level_name,

  matured_reached_user_count,
  start_to_next_start_rate,
  start_to_next_start_drop_rate,

  matured_won_user_count,
  win_to_next_start_rate,
  win_to_next_start_drop_rate,

  sample_size_status,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

ORDER BY
  app_version,
  level_number;
```

---

### 16.2 Tìm điểm rơi lớn nhất trong progression

```sql id="xigg0s"
SELECT
  app_version,
  level_number,
  level_name,
  next_level_name,

  matured_reached_user_count,
  start_to_next_start_drop_rate,
  win_to_next_start_drop_rate,

  avg_attempt_count_per_reached_user,
  first_start_near_tutorial_rate,

  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  sample_size_status = 'sufficient_sample'

ORDER BY
  start_to_next_start_drop_rate DESC,
  win_to_next_start_drop_rate DESC;
```

---

### 16.3 So sánh progression giữa app version

```sql id="rsqxa6"
SELECT
  level_number,
  level_name,
  next_level_name,

  app_version,

  matured_reached_user_count,
  start_to_next_start_rate,
  win_to_next_start_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`

WHERE
  sample_size_status = 'sufficient_sample'

ORDER BY
  level_number,
  app_version;
```

---

## 17. Lưu ý khi sử dụng

### 17.1 `app_version` lấy từ lần đầu bắt đầu level hiện tại

Cột:

```text id="4mem8x"
app_version
```

trong view này đến từ:

```text id="r8kli2"
first_start_app_version
```

của level hiện tại.

Nếu người chơi cập nhật app giữa level hiện tại và level sau, việc diễn giải theo app version cần cẩn thận.

---

### 17.2 Level kế tiếp được suy luận theo số thứ tự

View suy luận next level bằng:

```text id="k3n2w2"
level_number + 1
```

Nó không dùng mapping từ `dim_level_seq`.

Nếu progression thực tế không đi theo chuỗi `N001 → N002 → N003`, view có thể không phản ánh đúng progression thiết kế.

---

### 17.3 View chỉ bao gồm normal level

Vì nguồn là `vw_user_level_reach`, view chỉ bao gồm level dạng `Nxxx`.

Các level đặc biệt hoặc level dạng `c001`, `E1` không được tính.

---

### 17.4 Chỉ số đi tiếp dùng mốc 24 giờ

Các chỉ số drop sang next level chỉ nên đọc trên nhóm:

```text id="ogn36s"
matured_reached_user_count
```

Nếu sample thấp, view đã gắn:

```text id="0muv4s"
sample_size_status = low_sample
```

---

### 17.5 Timing dùng lần đầu start và lần đầu win

Các cột thời gian:

```text id="i9j6q3"
median_seconds_from_first_start_to_next_start
p75_seconds_from_first_start_to_next_start
median_seconds_from_first_win_to_next_start
p75_seconds_from_first_win_to_next_start
```

dựa trên mốc lần đầu bắt đầu level và lần thắng đầu tiên.

Chúng không mô tả mọi attempt, mà mô tả progression ở cấp người chơi.

---

## 18. Rủi ro nếu view sai logic

| Lỗi logic                                                     | Ảnh hưởng                                                  |
| ------------------------------------------------------------- | ---------------------------------------------------------- |
| Sai `vw_user_level_reach`                                     | Toàn bộ progression theo version sai.                      |
| Suy luận next level không khớp progression thật               | Tỷ lệ đi tiếp level sau sai.                               |
| Dùng app version từ first start nhưng không hiểu cập nhật app | Có thể gán progression sai version.                        |
| Không lọc matured cohort                                      | Đánh giá sai drop do người chơi chưa đủ thời gian.         |
| Sample thấp nhưng kết luận mạnh                               | Dễ quyết định sai về thiết kế level.                       |
| Không tách tutorial                                           | Có thể diễn giải sai progression ở các level gần tutorial. |
| Không hiểu win-to-next khác start-to-next                     | Nhầm vấn đề difficulty với vấn đề sau khi thắng.           |

---

## 19. Mức độ quan trọng

```text id="l4ah4t"
Mức độ quan trọng: Cao
```

Lý do:

* view này là nguồn chính để phân tích progression theo app version;
* view này giúp tách drop trước khi thắng và drop sau khi thắng;
* view này hỗ trợ phát hiện vấn đề UX sau level;
* view này được dùng trong pipeline health;
* view này hữu ích cho quyết định thiết kế level và luồng progression.

---

## 20. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="ce7r5x"
sql/ddl/monitoring_level_analysis/vw_level_progression_version.sql
```

Cấu trúc logic chính của view:

```sql id="kdpd7d"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_level_progression_version`
AS
WITH user_level AS (
  SELECT
    *
  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_user_level_reach`
),

level_transitions AS (
  SELECT
    current_level.user_pseudo_id,
    current_level.first_start_app_version AS app_version,
    current_level.level_number,
    current_level.level_name,

    CONCAT(
      'N',
      LPAD(CAST(current_level.level_number + 1 AS STRING), 3, '0')
    ) AS next_level_name,

    -- thông tin current level
    -- thông tin next level
    -- cờ đi tiếp sau start
    -- cờ đi tiếp sau win
    -- thời gian từ current level sang next level

  FROM
    user_level AS current_level

  LEFT JOIN
    user_level AS next_level
  ON
    current_level.user_pseudo_id = next_level.user_pseudo_id
    AND current_level.level_number + 1 = next_level.level_number
    AND next_level.first_start_time_utc > current_level.first_start_time_utc
),

aggregated AS (
  SELECT
    app_version,
    level_number,
    level_name,
    next_level_name,

    COUNT(*) AS reached_user_count,
    COUNTIF(is_matured_24h_for_next_level_check)
      AS matured_reached_user_count,
    COUNTIF(NOT is_matured_24h_for_next_level_check)
      AS not_matured_reached_user_count,

    -- win counts
    -- next level counts
    -- tutorial first start counts
    -- attempt counts
    -- timing quantiles

  FROM
    level_transitions

  GROUP BY
    app_version,
    level_number,
    level_name,
    next_level_name
),

final_metrics AS (
  SELECT
    *,
    -- rates
    -- drop counts
    -- sample_size_status
  FROM
    aggregated
)

SELECT
  *,
  CASE
    -- analyst_note tiếng Việt
  END AS analyst_note

FROM
  final_metrics;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: [[vw_user_level_reach]]

- Downstream: [[vw_pipeline_health_daily]]

