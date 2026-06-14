# vw_onboard_drop_diag_24h

## 1. Mục đích

`vw_onboard_drop_diag_24h` là view chẩn đoán các nhóm người chơi rơi trong giai đoạn người chơi mới.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này tập trung vào người chơi đã mở game lần đầu và đã đủ 24 giờ để quan sát hành vi.

Mục tiêu chính là phân loại người chơi vào các nhóm:

```text id="q1h37f"
1. Có first_open nhưng không có Open_first và không bắt đầu Level 1
2. Có Open_first nhưng không bắt đầu Level 1
3. Bắt đầu Level 1 nhưng không có Open_first
4. Có cả Open_first và bắt đầu Level 1
```

Ngoài việc đếm người chơi trong từng nhóm, view còn đo thêm:

* tỷ lệ của từng nhóm trong toàn bộ người chơi đã đủ 24 giờ;
* số người có `Vuot_home` trong từng nhóm;
* số người có `app_remove` trong từng nhóm;
* thời gian giữa các mốc quan trọng.

---

## 2. Vai trò trong hệ thống

`vw_onboard_drop_diag_24h` thuộc nhóm:

```text id="xzs3cq"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này bổ sung chiều sâu cho:

```text id="za0pgm"
vw_onboard_funnel_24h
```

Nếu `vw_onboard_funnel_24h` cho biết tỷ lệ rơi ở cấp tổng quan, thì `vw_onboard_drop_diag_24h` cho biết người chơi rơi thuộc nhóm nào và có tín hiệu chẩn đoán gì.

Luồng phụ thuộc chính:

```text id="2v2jgn"
dim_f10_level_map
vw_onboard_events
vw_level_start_eff
  └── vw_onboard_drop_diag_24h
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Trong số người chơi đã đủ 24 giờ quan sát, bao nhiêu người không có `Open_first`?
2. Bao nhiêu người có `Open_first` nhưng không bắt đầu Level 1?
3. Bao nhiêu người bắt đầu Level 1 nhưng thiếu `Open_first`?
4. Bao nhiêu người đi đủ từ `Open_first` đến Level 1?
5. Nhóm rơi nào chiếm tỷ trọng lớn nhất?
6. Nhóm nào có nhiều `Vuot_home` nhất?
7. Nhóm nào có nhiều `app_remove` nhất?
8. Thời gian từ `first_open` đến `Open_first` ở từng nhóm là bao lâu?
9. Thời gian từ `Open_first` đến Level 1 ở từng nhóm là bao lâu?
10. Người chơi gỡ game thường gỡ trước hay sau `Open_first`?
11. Người chơi gỡ game thường gỡ trước hay sau khi bắt đầu Level 1?
12. Có nhóm nào cho thấy lỗi tracking không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_onboard_drop_diag_24h` đại diện cho một nhóm onboarding trong một phiên bản game.

Có thể hiểu đơn giản:

```text id="0c8b75"
Một dòng = một app_version + một nhóm onboarding_bucket
```

View này không có dữ liệu chi tiết từng người chơi. Nó là bảng tổng hợp theo:

```text id="gzcw8g"
app_version
onboarding_bucket
```

Trong tài liệu này, “bucket” nghĩa là một nhóm phân loại người chơi theo điều kiện hành vi.

---

## 5. Loại đối tượng

```text id="wsi0m9"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

Vì view dùng `CURRENT_TIMESTAMP()` để xác định người chơi đã đủ 24 giờ hay chưa, kết quả có thể thay đổi theo thời điểm truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào ba nguồn chính:

| Nguồn                | Vai trò                                                         |
| -------------------- | --------------------------------------------------------------- |
| `dim_f10_level_map`  | Xác định Level 1 theo từng phiên bản game.                      |
| `vw_onboard_events`  | Cung cấp `first_open`, `Open_first`, `Vuot_home`, `app_remove`. |
| `vw_level_start_eff` | Cung cấp sự kiện bắt đầu Level 1 có hiệu lực.                   |

---

## 7. Cách xác định Level 1

View lấy Level 1 từ:

```text id="l6h1h1"
dim_f10_level_map
```

với điều kiện:

```sql id="7iyqpo"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step = 1
```

Điều này giúp view không phụ thuộc cứng vào một tên level cụ thể như `N001`.

Trong tài liệu này, “phụ thuộc cứng” nghĩa là viết trực tiếp một giá trị cố định vào logic, làm cho view dễ sai khi thiết kế level thay đổi.

---

## 8. Logic 24 giờ

View chỉ phân tích người chơi đã đủ 24 giờ kể từ `first_open`.

Điều kiện:

```text id="z8jnip"
first_open_time_utc <= thời điểm chạy view - 24 giờ
```

Nhóm này được gọi là nhóm đã đủ thời gian quan sát.

Các sự kiện sau được tìm trong vòng 24 giờ kể từ `first_open`:

```text id="iqr2pz"
Open_first
bắt đầu Level 1
Vuot_home
app_remove
```

---

## 9. Các nhóm onboarding

View phân người chơi vào 4 nhóm chính.

### 9.1 Nhóm 1: có `first_open` nhưng không có `Open_first` và không bắt đầu Level 1

```text id="vq4agz"
01_first_open_without_open_first_without_level1
```

Ý nghĩa:

* người chơi mở game lần đầu;
* không có `Open_first`;
* không bắt đầu Level 1 trong 24 giờ đầu.

Nhóm này có thể gợi ý vấn đề rất sớm trong luồng vào game, ví dụ:

* loading lâu;
* crash;
* người chơi thoát ngay;
* lỗi tracking `Open_first`;
* lỗi tracking bắt đầu level.

---

### 9.2 Nhóm 2: có `Open_first` nhưng không bắt đầu Level 1

```text id="kls4cf"
02_open_first_without_level1
```

Ý nghĩa:

* người chơi đi qua `Open_first`;
* nhưng không bắt đầu Level 1 trong 24 giờ đầu.

Nhóm này có thể gợi ý vấn đề sau bước mở đầu, ví dụ:

* kẹt ở màn hình chính;
* luồng vào level chưa rõ;
* lỗi loading level;
* hướng dẫn trước level quá dài;
* người chơi thoát trước khi vào gameplay.

---

### 9.3 Nhóm 3: bắt đầu Level 1 nhưng không có `Open_first`

```text id="43xdvg"
03_reached_level1_without_open_first
```

Ý nghĩa:

* người chơi có bắt đầu Level 1;
* nhưng không có `Open_first`.

Nhóm này thường là tín hiệu cần kiểm tra tracking.

Có thể xảy ra khi:

* `Open_first` không được gửi;
* `Open_first` gửi sai tên;
* người chơi đi thẳng vào Level 1 theo luồng đặc biệt;
* `Open_first` nằm ngoài cửa sổ 24 giờ;
* app version hoặc user id giữa các sự kiện không khớp.

---

### 9.4 Nhóm 4: có cả `Open_first` và bắt đầu Level 1

```text id="ef7eti"
04_reached_level1_after_open_first
```

Ý nghĩa:

* người chơi có `Open_first`;
* người chơi có bắt đầu Level 1 trong 24 giờ đầu.

Đây là nhóm hoàn thành luồng onboarding chính theo định nghĩa hiện tại.

---

## 10. Danh sách cột

| Cột                                                   | Kiểu dữ liệu | Ý nghĩa                                                               |
| ----------------------------------------------------- | ------------ | --------------------------------------------------------------------- |
| `app_version`                                         | `STRING`     | Phiên bản game.                                                       |
| `onboarding_bucket_order`                             | `INT64`      | Thứ tự nhóm onboarding để sắp xếp.                                    |
| `onboarding_bucket`                                   | `STRING`     | Tên nhóm onboarding.                                                  |
| `matured_first_open_user_count`                       | `INT64`      | Tổng số người chơi đã đủ 24 giờ quan sát trong phiên bản đó.          |
| `user_count`                                          | `INT64`      | Số người chơi trong nhóm onboarding hiện tại.                         |
| `bucket_rate_within_matured_first_open`               | `FLOAT64`    | Tỷ lệ của nhóm trong toàn bộ người chơi đã đủ 24 giờ.                 |
| `users_with_vuot_home_within_24h`                     | `INT64`      | Số người trong nhóm có `Vuot_home` trong 24 giờ đầu.                  |
| `vuot_home_rate_within_bucket`                        | `FLOAT64`    | Tỷ lệ người có `Vuot_home` trong nhóm.                                |
| `users_with_app_remove_within_24h`                    | `INT64`      | Số người trong nhóm có `app_remove` trong 24 giờ đầu.                 |
| `app_remove_rate_within_bucket`                       | `FLOAT64`    | Tỷ lệ người có `app_remove` trong nhóm.                               |
| `users_with_open_first`                               | `INT64`      | Số người trong nhóm có `Open_first`.                                  |
| `users_with_level1_start`                             | `INT64`      | Số người trong nhóm có bắt đầu Level 1.                               |
| `first_open_to_open_first_timing_sample_user_count`   | `INT64`      | Số người có đủ dữ liệu thời gian từ `first_open` đến `Open_first`.    |
| `open_first_to_level1_start_timing_sample_user_count` | `INT64`      | Số người có đủ dữ liệu thời gian từ `Open_first` đến bắt đầu Level 1. |
| `first_open_to_level1_start_timing_sample_user_count` | `INT64`      | Số người có đủ dữ liệu thời gian từ `first_open` đến bắt đầu Level 1. |
| `avg_seconds_first_open_to_open_first`                | `FLOAT64`    | Thời gian trung bình từ `first_open` đến `Open_first`.                |
| `median_seconds_first_open_to_open_first`             | `INT64`      | Trung vị thời gian từ `first_open` đến `Open_first`.                  |
| `p90_seconds_first_open_to_open_first`                | `INT64`      | Mốc 90% thời gian từ `first_open` đến `Open_first`.                   |
| `avg_seconds_open_first_to_level1_start`              | `FLOAT64`    | Thời gian trung bình từ `Open_first` đến bắt đầu Level 1.             |
| `median_seconds_open_first_to_level1_start`           | `INT64`      | Trung vị thời gian từ `Open_first` đến bắt đầu Level 1.               |
| `p90_seconds_open_first_to_level1_start`              | `INT64`      | Mốc 90% thời gian từ `Open_first` đến bắt đầu Level 1.                |
| `avg_seconds_first_open_to_level1_start`              | `FLOAT64`    | Thời gian trung bình từ `first_open` đến bắt đầu Level 1.             |
| `median_seconds_first_open_to_level1_start`           | `INT64`      | Trung vị thời gian từ `first_open` đến bắt đầu Level 1.               |
| `p90_seconds_first_open_to_level1_start`              | `INT64`      | Mốc 90% thời gian từ `first_open` đến bắt đầu Level 1.                |
| `avg_seconds_first_open_to_vuot_home`                 | `FLOAT64`    | Thời gian trung bình từ `first_open` đến `Vuot_home`.                 |
| `median_seconds_first_open_to_vuot_home`              | `INT64`      | Trung vị thời gian từ `first_open` đến `Vuot_home`.                   |
| `p90_seconds_first_open_to_vuot_home`                 | `INT64`      | Mốc 90% thời gian từ `first_open` đến `Vuot_home`.                    |
| `avg_seconds_open_first_to_vuot_home`                 | `FLOAT64`    | Thời gian trung bình từ `Open_first` đến `Vuot_home`.                 |
| `median_seconds_open_first_to_vuot_home`              | `INT64`      | Trung vị thời gian từ `Open_first` đến `Vuot_home`.                   |
| `p90_seconds_open_first_to_vuot_home`                 | `INT64`      | Mốc 90% thời gian từ `Open_first` đến `Vuot_home`.                    |
| `avg_seconds_level1_start_to_vuot_home`               | `FLOAT64`    | Thời gian trung bình từ bắt đầu Level 1 đến `Vuot_home`.              |
| `median_seconds_level1_start_to_vuot_home`            | `INT64`      | Trung vị thời gian từ bắt đầu Level 1 đến `Vuot_home`.                |
| `p90_seconds_level1_start_to_vuot_home`               | `INT64`      | Mốc 90% thời gian từ bắt đầu Level 1 đến `Vuot_home`.                 |
| `avg_seconds_first_open_to_app_remove`                | `FLOAT64`    | Thời gian trung bình từ `first_open` đến `app_remove`.                |
| `median_seconds_first_open_to_app_remove`             | `INT64`      | Trung vị thời gian từ `first_open` đến `app_remove`.                  |
| `p90_seconds_first_open_to_app_remove`                | `INT64`      | Mốc 90% thời gian từ `first_open` đến `app_remove`.                   |
| `avg_seconds_open_first_to_app_remove`                | `FLOAT64`    | Thời gian trung bình từ `Open_first` đến `app_remove`.                |
| `median_seconds_open_first_to_app_remove`             | `INT64`      | Trung vị thời gian từ `Open_first` đến `app_remove`.                  |
| `p90_seconds_open_first_to_app_remove`                | `INT64`      | Mốc 90% thời gian từ `Open_first` đến `app_remove`.                   |
| `avg_seconds_level1_start_to_app_remove`              | `FLOAT64`    | Thời gian trung bình từ bắt đầu Level 1 đến `app_remove`.             |
| `median_seconds_level1_start_to_app_remove`           | `INT64`      | Trung vị thời gian từ bắt đầu Level 1 đến `app_remove`.               |
| `p90_seconds_level1_start_to_app_remove`              | `INT64`      | Mốc 90% thời gian từ bắt đầu Level 1 đến `app_remove`.                |
| `first_first_open_time_utc`                           | `TIMESTAMP`  | Thời điểm `first_open` sớm nhất trong nhóm.                           |
| `last_first_open_time_utc`                            | `TIMESTAMP`  | Thời điểm `first_open` muộn nhất trong nhóm.                          |
| `diagnostics_view_query_time_utc`                     | `TIMESTAMP`  | Thời điểm view được truy vấn.                                         |

---

## 11. Công thức chỉ số chính

### 11.1 Tỷ lệ nhóm trong toàn bộ người chơi đã đủ 24 giờ

```text id="4z78wr"
bucket_rate_within_matured_first_open
=
user_count
/
matured_first_open_user_count
```

Ý nghĩa: nhóm này chiếm bao nhiêu phần trăm trong toàn bộ người chơi đã đủ 24 giờ quan sát.

---

### 11.2 Tỷ lệ có `Vuot_home` trong nhóm

```text id="w84iju"
vuot_home_rate_within_bucket
=
users_with_vuot_home_within_24h
/
user_count
```

Ý nghĩa: trong nhóm này, bao nhiêu phần trăm người chơi có `Vuot_home`.

---

### 11.3 Tỷ lệ có `app_remove` trong nhóm

```text id="7l6jp5"
app_remove_rate_within_bucket
=
users_with_app_remove_within_24h
/
user_count
```

Ý nghĩa: trong nhóm này, bao nhiêu phần trăm người chơi gỡ ứng dụng trong 24 giờ đầu.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Kiểm tra tổng các nhóm có bằng mẫu số không

```sql id="mal38o"
SELECT
  app_version,

  MAX(matured_first_open_user_count)
    AS matured_first_open_user_count,

  SUM(user_count)
    AS summed_bucket_user_count,

  MAX(matured_first_open_user_count) - SUM(user_count)
    AS user_count_gap

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

GROUP BY
  app_version

ORDER BY
  app_version;
```

Kỳ vọng:

```text id="a0qrxb"
user_count_gap = 0
```

Nếu khác 0, logic phân nhóm có thể bị sai.

---

### 12.2 Kiểm tra tổng tỷ lệ nhóm

```sql id="k64bo0"
SELECT
  app_version,

  SUM(bucket_rate_within_matured_first_open)
    AS total_bucket_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

GROUP BY
  app_version

ORDER BY
  app_version;
```

Kỳ vọng:

```text id="cpo2kq"
total_bucket_rate gần bằng 1
```

Có thể có sai số rất nhỏ do kiểu số thực.

---

### 12.3 Xem phân bố nhóm onboarding

```sql id="iwqlln"
SELECT
  app_version,
  onboarding_bucket_order,
  onboarding_bucket,
  matured_first_open_user_count,
  user_count,
  bucket_rate_within_matured_first_open

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

ORDER BY
  app_version,
  onboarding_bucket_order;
```

Truy vấn này là truy vấn chính để đọc phân bố người chơi theo nhóm.

---

### 12.4 Kiểm tra nhóm có `app_remove` cao

```sql id="zyl6xp"
SELECT
  app_version,
  onboarding_bucket_order,
  onboarding_bucket,
  user_count,
  users_with_app_remove_within_24h,
  app_remove_rate_within_bucket

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

ORDER BY
  app_remove_rate_within_bucket DESC,
  users_with_app_remove_within_24h DESC;
```

Truy vấn này giúp phát hiện nhóm nào có tỷ lệ gỡ game cao nhất.

---

### 12.5 Kiểm tra nhóm bất thường: Level 1 nhưng không có `Open_first`

```sql id="b2vmv2"
SELECT
  app_version,
  onboarding_bucket,
  user_count,
  bucket_rate_within_matured_first_open,
  users_with_level1_start,
  users_with_open_first

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

WHERE
  onboarding_bucket = '03_reached_level1_without_open_first'

ORDER BY
  user_count DESC;
```

Nếu nhóm này cao, cần kiểm tra tracking của `Open_first`.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Dùng để phân tích rơi ở giai đoạn đầu

View này phù hợp để trả lời:

```text id="nck6gv"
Người chơi rơi trước Open_first hay sau Open_first?
```

Truy vấn:

```sql id="xbalvf"
SELECT
  app_version,
  onboarding_bucket_order,
  onboarding_bucket,
  user_count,
  bucket_rate_within_matured_first_open

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

ORDER BY
  app_version,
  onboarding_bucket_order;
```

---

### 13.2 Dùng để phân tích rơi kèm tín hiệu `Vuot_home`

```sql id="17xe53"
SELECT
  app_version,
  onboarding_bucket,
  user_count,
  users_with_vuot_home_within_24h,
  vuot_home_rate_within_bucket

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

ORDER BY
  vuot_home_rate_within_bucket DESC;
```

Nếu một nhóm có nhiều `Vuot_home` nhưng không bắt đầu Level 1, có thể cần kiểm tra ý nghĩa tracking của `Vuot_home` hoặc luồng từ màn hình chính vào level.

---

### 13.3 Dùng để phân tích thời gian đến gỡ game

```sql id="46m7l7"
SELECT
  app_version,
  onboarding_bucket,
  user_count,
  users_with_app_remove_within_24h,
  app_remove_rate_within_bucket,

  avg_seconds_first_open_to_app_remove,
  median_seconds_first_open_to_app_remove,
  p90_seconds_first_open_to_app_remove

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`

WHERE
  users_with_app_remove_within_24h > 0

ORDER BY
  app_remove_rate_within_bucket DESC;
```

Truy vấn này giúp xem nhóm nào gỡ game nhanh sau `first_open`.

---

## 14. Lưu ý khi sử dụng

### 14.1 View chỉ tính người chơi đã đủ 24 giờ

Khác với `vw_onboard_funnel_24h`, view này chỉ phân tích nhóm đã đủ 24 giờ.

Vì vậy, mẫu số chính là:

```text id="z7pb08"
matured_first_open_user_count
```

Không có cột `not_matured_first_open_user_count` trong view này.

---

### 14.2 View là bảng chẩn đoán, không phải bảng phễu tổng quan

Nếu cần xem phễu tổng quan, dùng:

```text id="p4jnjx"
vw_onboard_funnel_24h
```

Nếu cần xem người chơi rơi thuộc nhóm nào, dùng:

```text id="rvz3ku"
vw_onboard_drop_diag_24h
```

---

### 14.3 Nhóm 3 thường là tín hiệu kiểm tra tracking

Nhóm:

```text id="g3th7v"
03_reached_level1_without_open_first
```

không nhất thiết là lỗi, nhưng thường cần được kiểm tra vì theo luồng thông thường người chơi nên có `Open_first` trước khi bắt đầu Level 1.

---

### 14.4 Kết quả thay đổi theo thời điểm truy vấn

View dùng thời điểm hiện tại để xác định ai đã đủ 24 giờ quan sát.

Cột sau cho biết thời điểm tính view:

```text id="e7yczx"
diagnostics_view_query_time_utc
```

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic               | Ảnh hưởng                                                |
| ----------------------- | -------------------------------------------------------- |
| Sai mapping Level 1     | Phân loại nhóm có hoặc không bắt đầu Level 1 bị sai.     |
| Sai logic 24 giờ        | Người chơi chưa đủ quan sát có thể bị đưa vào phân tích. |
| Lọc thiếu `Open_first`  | Tăng giả nhóm rơi trước `Open_first`.                    |
| Lọc thiếu Level 1 start | Tăng giả nhóm không bắt đầu Level 1.                     |
| Lọc thiếu `Vuot_home`   | Mất tín hiệu chẩn đoán sau bước đầu.                     |
| Lọc thiếu `app_remove`  | Đánh giá thấp tỷ lệ gỡ game sớm.                         |
| Không kiểm tra nhóm 3   | Có thể bỏ qua lỗi tracking `Open_first`.                 |

---

## 16. Mức độ quan trọng

```text id="qmvjxk"
Mức độ quan trọng: Cao
```

Lý do:

* view này giúp xác định người chơi rơi ở đoạn nào trong giai đoạn đầu;
* view này bổ sung chẩn đoán cho `vw_onboard_funnel_24h`;
* view này giúp phát hiện lỗi tracking `Open_first`;
* view này hỗ trợ phân tích rời bỏ sớm qua `app_remove`;
* view này hữu ích cho cả Game Design, Product và Data.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="w89xzw"
sql/ddl/current_live_analysis/vw_onboard_drop_diag_24h.sql
```

Cấu trúc logic chính của view:

```sql id="lq7zfc"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_drop_diag_24h`
AS
WITH analysis_params AS (
  SELECT
    CURRENT_TIMESTAMP() AS analysis_time_utc
),

level1_mapping AS (...),

first_open_by_user AS (...),

matured_first_open_cohort AS (...),

open_first_by_user AS (...),

level1_start_by_user AS (...),

diagnostic_signals_by_user AS (...),

user_classification AS (
  SELECT
    -- phân người chơi vào 4 nhóm onboarding
    -- tính thời gian giữa các mốc
),

app_version_denominator AS (...)

SELECT
  app_version,
  onboarding_bucket_order,
  onboarding_bucket,
  matured_first_open_user_count,
  user_count,
  bucket_rate_within_matured_first_open,
  -- chỉ số Vuot_home
  -- chỉ số app_remove
  -- chỉ số thời gian
  -- thời điểm truy vấn

FROM
  user_classification

GROUP BY
  app_version,
  onboarding_bucket_order,
  onboarding_bucket;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[dim_f10_level_map]]
	- [[vw_onboard_events]]
	- [[vw_level_start_eff]]

- Downstream: Không có

