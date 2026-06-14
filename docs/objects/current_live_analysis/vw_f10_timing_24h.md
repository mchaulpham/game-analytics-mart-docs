# vw_f10_timing_24h

## 1. Mục đích

`vw_f10_timing_24h` là view đo thời gian người chơi đi qua 10 level đầu trong vòng 24 giờ kể từ lần đầu bắt đầu Level 1.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này tập trung vào yếu tố thời gian, gồm:

* thời gian từ Level 1 đến từng level;
* thời gian từ level trước đến level hiện tại;
* thời gian người chơi ở trong level, tính theo chênh lệch giữa thời điểm bắt đầu và kết thúc;
* thời lượng chơi level do game gửi lên qua `duration_sec`.

Mục tiêu chính là giúp team hiểu người chơi mất bao lâu để đi qua từng bước trong 10 level đầu.

---

## 2. Vai trò trong hệ thống

`vw_f10_timing_24h` thuộc nhóm:

```text id="oljck8"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là phần bổ sung về thời gian cho:

```text id="q475me"
vw_f10_funnel_24h
```

Nếu `vw_f10_funnel_24h` trả lời câu hỏi “bao nhiêu người tới từng level”, thì `vw_f10_timing_24h` trả lời câu hỏi “người chơi mất bao lâu để tới đó”.

Luồng phụ thuộc chính:

```text id="rjftw2"
dim_f10_level_map
vw_level_start_eff
vw_level_events
  └── vw_f10_timing_24h
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Người chơi mất bao lâu để đi từ Level 1 đến Level 2?
2. Người chơi mất bao lâu để đi từ Level 1 đến Level 10?
3. Level nào làm tăng thời gian đi qua phễu nhiều nhất?
4. Thời gian từ level trước đến level hiện tại là bao lâu?
5. Người chơi mất bao lâu ở bên trong mỗi level?
6. Thời lượng do game gửi qua `duration_sec` có khớp với thời gian thực tế giữa `Start_level` và `End_level` không?
7. Level nào có thời gian chơi trung bình cao?
8. Level nào có mốc 90% thời gian cao bất thường?
9. Có bao nhiêu người có dữ liệu `End_level` để tính thời gian trong level?
10. Có bao nhiêu người có `duration_sec` hợp lệ?
11. Có level nào có tỷ lệ dữ liệu thời gian quá thấp không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_f10_timing_24h` đại diện cho một bước trong 10 level đầu của một phiên bản game.

Có thể hiểu đơn giản:

```text id="iv9r8l"
Một dòng = một app_version + một progression_step trong 10 level đầu
```

View này là bảng tổng hợp theo bước level, không phải dữ liệu chi tiết từng người chơi.

---

## 5. Loại đối tượng

```text id="c0wx9r"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

View dùng `CURRENT_TIMESTAMP()` để xác định người chơi đã đủ 24 giờ kể từ Level 1 hay chưa. Vì vậy, kết quả có thể thay đổi theo thời điểm truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào ba nguồn chính:

| Nguồn                | Vai trò                                                     |
| -------------------- | ----------------------------------------------------------- |
| `dim_f10_level_map`  | Xác định 10 level đầu theo từng phiên bản game.             |
| `vw_level_start_eff` | Cung cấp các lần bắt đầu level có hiệu lực.                 |
| `vw_level_events`    | Cung cấp sự kiện `End_level` để tính thời gian trong level. |

Lưu ý: view này không dùng `vw_level_attempts`. Nó tự tìm `End_level` đầu tiên sau thời điểm bắt đầu level trong vòng 6 giờ.

---

## 7. Cách xác định 10 level đầu

View lấy mapping từ:

```text id="gsbx87"
dim_f10_level_map
```

với điều kiện:

```sql id="9dscrs"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step BETWEEN 1 AND 10
```

Các trường mapping được đưa vào kết quả gồm:

```text id="d8uuwi"
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

---

## 8. Cách xác định nhóm người chơi

View xác định người chơi đã bắt đầu Level 1 đầu tiên từ:

```text id="acpd5h"
vw_level_start_eff
```

Sau đó chỉ giữ những người chơi đã đủ 24 giờ kể từ lần đầu bắt đầu Level 1.

Điều kiện:

```text id="59rfzx"
level1_start_time_utc <= thời điểm chạy view - 24 giờ
```

Các chỉ số thời gian trong view chỉ tính trên nhóm người chơi đã đủ thời gian quan sát.

---

## 9. Logic thứ tự bước

View kiểm tra thứ tự các bước tương tự như `vw_f10_funnel_24h`.

Một người chơi được tính ở bước hiện tại nếu:

```text id="yxzo2x"
1. Có bắt đầu level của bước hiện tại trong 24 giờ sau Level 1
2. Không thiếu bất kỳ bước nào trước đó
3. Không có bước nào xảy ra ngược thứ tự thời gian
```

Điều này giúp view đo thời gian trên một phễu tuần tự, thay vì chỉ đếm bất kỳ lần chạm tới level.

Trong tài liệu này, “phễu tuần tự” nghĩa là người chơi phải đi qua các bước theo đúng thứ tự đã định nghĩa.

---

## 10. Các nhóm thời gian được đo

### 10.1 Thời gian từ Level 1 đến bước hiện tại

Cột nhóm này đo:

```text id="yoqu3e"
first_reach_time_utc của bước hiện tại
-
level1_start_time_utc
```

Các cột liên quan:

```text id="55f4uy"
level1_to_step_timing_sample_user_count
avg_seconds_from_level1_to_step
median_seconds_from_level1_to_step
p90_seconds_from_level1_to_step
```

---

### 10.2 Thời gian từ bước trước đến bước hiện tại

Cột nhóm này đo:

```text id="q984x1"
first_reach_time_utc của bước hiện tại
-
first_reach_time_utc của bước trước
```

Các cột liên quan:

```text id="u1rh6c"
previous_step_to_step_timing_sample_user_count
avg_seconds_from_previous_step_to_step
median_seconds_from_previous_step_to_step
p90_seconds_from_previous_step_to_step
```

Với bước 1, nhóm thời gian này sẽ không có giá trị vì không có bước trước.

---

### 10.3 Thời gian trong level theo đồng hồ hệ thống

View tìm `End_level` đầu tiên của cùng người chơi, cùng phiên bản game, cùng level, xảy ra trong vòng 6 giờ sau thời điểm bắt đầu level.

Sau đó tính:

```text id="eht11l"
End_level event_time_utc
-
Start_level start_time_utc
```

Các cột liên quan:

```text id="d3tmv4"
level_end_event_sample_user_count
level_end_event_coverage_rate
level_end_success_user_count
avg_wallclock_seconds_spent_inside_level
median_wallclock_seconds_spent_inside_level
p90_wallclock_seconds_spent_inside_level
```

Trong tài liệu này, “wall-clock” nghĩa là thời gian tính theo chênh lệch đồng hồ giữa hai timestamp. Ở đây là thời điểm bắt đầu level và thời điểm kết thúc level.

---

### 10.4 Thời lượng trong level do game gửi lên

View cũng lấy `duration_sec` từ `End_level`.

Chỉ những giá trị hợp lệ mới được giữ:

```text id="w6i9ln"
duration_sec >= 0
và
duration_sec <= 21600
```

21600 giây tương đương 6 giờ.

Các cột liên quan:

```text id="ha71j4"
reported_duration_sample_user_count
reported_duration_coverage_rate
avg_reported_duration_sec_inside_level
median_reported_duration_sec_inside_level
p90_reported_duration_sec_inside_level
```

Trong tài liệu này, “reported duration” nghĩa là thời lượng do game gửi lên qua tham số `duration_sec`.

---

## 11. Danh sách cột

| Cột                                              | Kiểu dữ liệu | Ý nghĩa                                                                            |
| ------------------------------------------------ | ------------ | ---------------------------------------------------------------------------------- |
| `app_version`                                    | `STRING`     | Phiên bản game.                                                                    |
| `progression_step`                               | `INT64`      | Bước trong 10 level đầu.                                                           |
| `progression_slot_id`                            | `STRING`     | Mã vị trí của bước trong tiến trình.                                               |
| `level_name`                                     | `STRING`     | Tên level của bước đó.                                                             |
| `level_design_id`                                | `STRING`     | Mã thiết kế level.                                                                 |
| `level_type`                                     | `STRING`     | Loại level.                                                                        |
| `previous_progression_step`                      | `INT64`      | Bước trước đó trong tiến trình.                                                    |
| `next_progression_step`                          | `INT64`      | Bước tiếp theo trong tiến trình.                                                   |
| `mapping_confidence_status`                      | `STRING`     | Mức độ tin cậy của mapping.                                                        |
| `mapping_source`                                 | `STRING`     | Nguồn tạo mapping.                                                                 |
| `user_count`                                     | `INT64`      | Số người chơi hợp lệ đã tới bước hiện tại theo đúng thứ tự.                        |
| `level1_to_step_timing_sample_user_count`        | `INT64`      | Số người có đủ dữ liệu để tính thời gian từ Level 1 đến bước hiện tại.             |
| `avg_seconds_from_level1_to_step`                | `FLOAT64`    | Thời gian trung bình từ Level 1 đến bước hiện tại.                                 |
| `median_seconds_from_level1_to_step`             | `INT64`      | Trung vị thời gian từ Level 1 đến bước hiện tại.                                   |
| `p90_seconds_from_level1_to_step`                | `INT64`      | Mốc 90% thời gian từ Level 1 đến bước hiện tại.                                    |
| `previous_step_to_step_timing_sample_user_count` | `INT64`      | Số người có đủ dữ liệu để tính thời gian từ bước trước đến bước hiện tại.          |
| `avg_seconds_from_previous_step_to_step`         | `FLOAT64`    | Thời gian trung bình từ bước trước đến bước hiện tại.                              |
| `median_seconds_from_previous_step_to_step`      | `INT64`      | Trung vị thời gian từ bước trước đến bước hiện tại.                                |
| `p90_seconds_from_previous_step_to_step`         | `INT64`      | Mốc 90% thời gian từ bước trước đến bước hiện tại.                                 |
| `level_end_event_sample_user_count`              | `INT64`      | Số người có `End_level` phù hợp trong vòng 6 giờ sau khi bắt đầu level.            |
| `level_end_event_coverage_rate`                  | `FLOAT64`    | Tỷ lệ người có `End_level` phù hợp trên tổng số người ở bước đó.                   |
| `level_end_success_user_count`                   | `INT64`      | Số người có `End_level` đầu tiên với `success = TRUE`.                             |
| `avg_wallclock_seconds_spent_inside_level`       | `FLOAT64`    | Thời gian trung bình trong level tính bằng chênh lệch timestamp giữa start và end. |
| `median_wallclock_seconds_spent_inside_level`    | `INT64`      | Trung vị thời gian trong level theo timestamp.                                     |
| `p90_wallclock_seconds_spent_inside_level`       | `INT64`      | Mốc 90% thời gian trong level theo timestamp.                                      |
| `reported_duration_sample_user_count`            | `INT64`      | Số người có `duration_sec` hợp lệ.                                                 |
| `reported_duration_coverage_rate`                | `FLOAT64`    | Tỷ lệ người có `duration_sec` hợp lệ trên tổng số người ở bước đó.                 |
| `avg_reported_duration_sec_inside_level`         | `FLOAT64`    | Thời lượng trung bình trong level theo `duration_sec`.                             |
| `median_reported_duration_sec_inside_level`      | `FLOAT64`    | Trung vị thời lượng trong level theo `duration_sec`.                               |
| `p90_reported_duration_sec_inside_level`         | `FLOAT64`    | Mốc 90% thời lượng trong level theo `duration_sec`.                                |
| `timing_view_query_time_utc`                     | `TIMESTAMP`  | Thời điểm view được truy vấn.                                                      |

---

## 12. Công thức chỉ số chính

### 12.1 Tỷ lệ có `End_level`

```text id="lbrd92"
level_end_event_coverage_rate
=
level_end_event_sample_user_count
/
user_count
```

Ý nghĩa: trong số người đã tới bước hiện tại, bao nhiêu người có `End_level` phù hợp trong vòng 6 giờ.

---

### 12.2 Tỷ lệ có `duration_sec` hợp lệ

```text id="nst3an"
reported_duration_coverage_rate
=
reported_duration_sample_user_count
/
user_count
```

Ý nghĩa: trong số người đã tới bước hiện tại, bao nhiêu người có `duration_sec` hợp lệ.

---

### 12.3 Thời gian từ Level 1 đến bước hiện tại

```text id="cm496s"
seconds_from_level1_to_step
=
first_reach_time_utc của bước hiện tại
-
level1_start_time_utc
```

---

### 12.4 Thời gian từ bước trước đến bước hiện tại

```text id="nwx95o"
seconds_from_previous_step_to_step
=
first_reach_time_utc của bước hiện tại
-
first_reach_time_utc của bước trước
```

---

### 12.5 Thời gian trong level theo timestamp

```text id="bvc1xg"
wallclock_seconds_spent_inside_level
=
End_level event_time_utc
-
Start_level first_reach_time_utc
```

---

## 13. Lưu ý quan trọng về logic ghép `End_level`

View tìm `End_level` đầu tiên sau khi người chơi bắt đầu level, trong vòng 6 giờ.

Điều kiện chính:

```text id="1z71hl"
cùng user_pseudo_id
cùng app_version
cùng level_name
End_level xảy ra sau Start_level
End_level không quá 6 giờ sau Start_level
```

Lưu ý quan trọng: view này không ghép theo `attempt_no`.

Điều đó khác với:

```text id="7f9ueg"
vw_level_attempts
```

`vw_level_attempts` ghép theo `attempt_no`, còn `vw_f10_timing_24h` chỉ tìm `End_level` đầu tiên cùng level trong vòng 6 giờ.

Điểm này cần được hiểu rõ khi so sánh hai view.

---

## 14. Kiểm tra chất lượng dữ liệu khuyến nghị

### 14.1 Kiểm tra độ phủ `End_level`

```sql id="kiiu08"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  level_end_event_sample_user_count,
  level_end_event_coverage_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

Nếu `level_end_event_coverage_rate` thấp, các chỉ số thời gian trong level cần được đọc thận trọng.

---

### 14.2 Kiểm tra độ phủ `duration_sec`

```sql id="pidxox"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  reported_duration_sample_user_count,
  reported_duration_coverage_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

Nếu `reported_duration_coverage_rate` thấp, không nên dùng `duration_sec` làm chỉ số chính.

---

### 14.3 So sánh thời gian theo timestamp và `duration_sec`

```sql id="wrn7e8"
SELECT
  app_version,
  progression_step,
  level_name,

  avg_wallclock_seconds_spent_inside_level,
  avg_reported_duration_sec_inside_level,

  avg_wallclock_seconds_spent_inside_level
    - avg_reported_duration_sec_inside_level
    AS avg_duration_gap_sec

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

WHERE
  avg_wallclock_seconds_spent_inside_level IS NOT NULL
  AND avg_reported_duration_sec_inside_level IS NOT NULL

ORDER BY
  ABS(avg_duration_gap_sec) DESC;
```

Nếu chênh lệch lớn, cần kiểm tra cách game gửi `duration_sec`.

---

### 14.4 Tìm level có thời gian p90 cao

```sql id="o0chgz"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  p90_seconds_from_level1_to_step,
  p90_seconds_from_previous_step_to_step,
  p90_wallclock_seconds_spent_inside_level,
  p90_reported_duration_sec_inside_level

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

ORDER BY
  p90_seconds_from_previous_step_to_step DESC;
```

Mốc 90% giúp phát hiện các trường hợp người chơi mất rất nhiều thời gian ở một bước.

---

### 14.5 Kiểm tra bước 1

```sql id="ob5qbu"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  level1_to_step_timing_sample_user_count,
  avg_seconds_from_level1_to_step,
  previous_step_to_step_timing_sample_user_count,
  avg_seconds_from_previous_step_to_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

WHERE
  progression_step = 1

ORDER BY
  app_version;
```

Kỳ vọng thông thường:

```text id="k82cn7"
avg_seconds_from_level1_to_step = 0
previous_step_to_step_timing_sample_user_count = 0
avg_seconds_from_previous_step_to_step = NULL
```

Vì bước 1 chính là điểm bắt đầu của phễu.

---

## 15. Cách sử dụng khuyến nghị

### 15.1 Xem thời gian đi qua từng bước

```sql id="rlytv6"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,

  avg_seconds_from_level1_to_step,
  median_seconds_from_level1_to_step,
  p90_seconds_from_level1_to_step,

  avg_seconds_from_previous_step_to_step,
  median_seconds_from_previous_step_to_step,
  p90_seconds_from_previous_step_to_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

---

### 15.2 Xem thời gian ở trong từng level

```sql id="zq2p5o"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,

  level_end_event_coverage_rate,
  avg_wallclock_seconds_spent_inside_level,
  median_wallclock_seconds_spent_inside_level,
  p90_wallclock_seconds_spent_inside_level,

  reported_duration_coverage_rate,
  avg_reported_duration_sec_inside_level,
  median_reported_duration_sec_inside_level,
  p90_reported_duration_sec_inside_level

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

ORDER BY
  app_version,
  progression_step;
```

---

### 15.3 Tìm bước có dấu hiệu chậm bất thường

```sql id="8p8xz8"
SELECT
  app_version,
  progression_step,
  level_name,
  user_count,
  p90_seconds_from_previous_step_to_step,

  CASE
    WHEN user_count < 30 THEN 'Chưa đủ mẫu'
    WHEN p90_seconds_from_previous_step_to_step >= 900 THEN 'Rất chậm'
    WHEN p90_seconds_from_previous_step_to_step >= 300 THEN 'Cần theo dõi'
    ELSE 'Tạm ổn'
  END AS timing_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`

WHERE
  progression_step > 1

ORDER BY
  p90_seconds_from_previous_step_to_step DESC;
```

Ngưỡng trong truy vấn này chỉ là ví dụ. Team nên điều chỉnh theo chuẩn nội bộ.

---

## 16. Lưu ý khi sử dụng

### 16.1 View này đo thời gian, không đo tỷ lệ rơi

Nếu muốn xem tỷ lệ rơi từ bước này sang bước khác, dùng:

```text id="k2ldx4"
vw_f10_funnel_24h
```

Nếu muốn kết hợp cả tỷ lệ rơi và thời gian, dùng:

```text id="jcxpsx"
vw_f10_funnel_timing_24h
```

---

### 16.2 View chỉ tính người chơi đi đúng thứ tự

Người chơi phải đi qua các bước trước đó theo đúng thứ tự mới được tính ở bước hiện tại.

Điều này giúp chỉ số thời gian phản ánh phễu tuần tự, nhưng có thể thấp hơn cách đếm đơn giản theo từng level riêng lẻ.

---

### 16.3 Không nên dùng chỉ số thời gian nếu độ phủ thấp

Nếu:

```text id="gg5ipd"
level_end_event_coverage_rate thấp
```

hoặc:

```text id="lb7vq1"
reported_duration_coverage_rate thấp
```

thì chỉ số thời gian trong level có thể không đại diện tốt.

---

### 16.4 Cần phân biệt `wallclock` và `duration_sec`

Có hai cách đo thời gian trong level:

| Loại thời gian                         | Ý nghĩa                                                      |
| -------------------------------------- | ------------------------------------------------------------ |
| `wallclock_seconds_spent_inside_level` | Chênh lệch timestamp giữa sự kiện bắt đầu và kết thúc level. |
| `reported_duration_sec_inside_level`   | Thời lượng do game gửi qua tham số `duration_sec`.           |

Hai giá trị này có thể khác nhau nếu:

* game tạm dừng;
* app đi nền;
* tracking gửi chậm;
* người chơi mất kết nối;
* `duration_sec` được tính bằng logic nội bộ khác.

---

### 16.5 Không ghép theo `attempt_no`

View này tìm `End_level` đầu tiên sau khi bắt đầu level, nhưng không ghép theo `attempt_no`.

Nếu cần phân tích chính xác theo từng lượt chơi, nên xem thêm:

```text id="yuuc3e"
vw_level_attempts
```

---

## 17. Rủi ro nếu view sai logic

| Lỗi logic                                             | Ảnh hưởng                                        |
| ----------------------------------------------------- | ------------------------------------------------ |
| Sai mapping 10 level đầu                              | Thời gian được tính cho sai level hoặc sai bước. |
| Sai Level 1 cohort                                    | Mốc bắt đầu thời gian bị sai.                    |
| Không kiểm tra thứ tự bước                            | Có thể tính thời gian cho người chơi nhảy cóc.   |
| Sai logic 24 giờ                                      | Tính cả người chơi chưa đủ thời gian quan sát.   |
| Ghép `End_level` sai                                  | Thời gian trong level bị sai.                    |
| Không hiểu khác biệt giữa timestamp và `duration_sec` | Diễn giải sai thời lượng chơi.                   |
| Độ phủ thời gian thấp nhưng vẫn kết luận mạnh         | Dễ ra quyết định sai về thiết kế level.          |

---

## 18. Mức độ quan trọng

```text id="w7xnud"
Mức độ quan trọng: Cao
```

Lý do:

* view này giúp hiểu tốc độ đi qua 10 level đầu;
* view này phát hiện level hoặc bước có thời gian bất thường;
* view này là đầu vào cho `vw_f10_funnel_timing_24h`;
* view này hỗ trợ chẩn đoán thiết kế level khi kết hợp với tỷ lệ rơi và tỷ lệ thắng;
* view này giúp phát hiện vấn đề trong `duration_sec` hoặc `End_level`.

---

## 19. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="hch40w"
sql/ddl/current_live_analysis/vw_f10_timing_24h.sql
```

Cấu trúc logic chính của view:

```sql id="gtckzx"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_f10_timing_24h`
AS
WITH analysis_params AS (
  SELECT
    CURRENT_TIMESTAMP() AS analysis_time_utc
),

first10_mapping AS (...),

level1_mapping AS (...),

level1_first_start_by_user AS (...),

level1_matured_cohort AS (...),

user_step_reaches AS (...),

user_step_with_previous_time AS (...),

user_step_sequence_check AS (...),

valid_user_step_reaches AS (...),

user_step_timing_base AS (...),

first_same_level_end_after_step_start AS (...),

user_step_timing_with_end AS (...),

timing_metrics_by_step AS (...)

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

  user_count,

  -- thời gian từ Level 1 đến bước hiện tại
  level1_to_step_timing_sample_user_count,
  avg_seconds_from_level1_to_step,
  median_seconds_from_level1_to_step,
  p90_seconds_from_level1_to_step,

  -- thời gian từ bước trước đến bước hiện tại
  previous_step_to_step_timing_sample_user_count,
  avg_seconds_from_previous_step_to_step,
  median_seconds_from_previous_step_to_step,
  p90_seconds_from_previous_step_to_step,

  -- thời gian trong level theo timestamp
  level_end_event_sample_user_count,
  level_end_event_coverage_rate,
  level_end_success_user_count,
  avg_wallclock_seconds_spent_inside_level,
  median_wallclock_seconds_spent_inside_level,
  p90_wallclock_seconds_spent_inside_level,

  -- thời lượng trong level do game gửi
  reported_duration_sample_user_count,
  reported_duration_coverage_rate,
  avg_reported_duration_sec_inside_level,
  median_reported_duration_sec_inside_level,
  p90_reported_duration_sec_inside_level,

  timing_view_query_time_utc

FROM
  timing_metrics_by_step;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[dim_f10_level_map]]
	- [[vw_level_start_eff]]
	- [[vw_level_events]]

- Downstream: [[vw_f10_funnel_timing_24h]]

