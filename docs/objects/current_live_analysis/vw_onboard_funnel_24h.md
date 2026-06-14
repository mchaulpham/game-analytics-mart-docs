# vw_onboard_funnel_24h

## 1. Mục đích

`vw_onboard_funnel_24h` là view tổng hợp phễu người chơi mới trong vòng 24 giờ đầu.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

“Phễu” nghĩa là chuỗi bước mà người chơi cần đi qua. Với view này, phễu chính là:

```text id="6yzf3q"
first_open
  ↓
Open_first
  ↓
bắt đầu Level 1
```

Mục tiêu của view là trả lời câu hỏi:

```text id="jpf84v"
Trong 24 giờ đầu sau khi mở game lần đầu, người chơi có đi được từ first_open đến Open_first và bắt đầu Level 1 không?
```

View này tổng hợp kết quả theo từng phiên bản game.

---

## 2. Vai trò trong hệ thống

`vw_onboard_funnel_24h` thuộc nhóm:

```text id="3he2c4"
current_live_analysis
```

Tức là nhóm phân tích hiện tại.

View này là một bảng kết quả phục vụ trực tiếp cho việc theo dõi giai đoạn người chơi mới.

Nó không phải là view làm sạch dữ liệu gốc. Nó là view phân tích đã tổng hợp từ các view nền tảng.

Luồng phụ thuộc chính:

```text id="ydzclo"
dim_f10_level_map
vw_onboard_events
vw_level_start_eff
  └── vw_onboard_funnel_24h
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu người chơi mở game lần đầu?
2. Có bao nhiêu người chơi đã đủ 24 giờ để quan sát hành vi?
3. Có bao nhiêu người chơi đi tới `Open_first` trong 24 giờ đầu?
4. Có bao nhiêu người chơi bắt đầu Level 1 trong 24 giờ đầu?
5. Tỷ lệ từ `first_open` sang `Open_first` là bao nhiêu?
6. Tỷ lệ từ `Open_first` sang bắt đầu Level 1 là bao nhiêu?
7. Tỷ lệ từ `first_open` sang bắt đầu Level 1 là bao nhiêu?
8. Có bao nhiêu người chơi rơi trước `Open_first`?
9. Có bao nhiêu người chơi có `Open_first` nhưng không bắt đầu Level 1?
10. Có bao nhiêu người chơi có `Vuot_home` trong 24 giờ đầu?
11. Có bao nhiêu người chơi có `app_remove` trong 24 giờ đầu?
12. Thời gian trung bình từ `first_open` đến `Open_first` là bao lâu?
13. Thời gian trung bình từ `Open_first` đến bắt đầu Level 1 là bao lâu?
14. Thời gian trung bình từ `first_open` đến bắt đầu Level 1 là bao lâu?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_onboard_funnel_24h` đại diện cho một phiên bản game.

Có thể hiểu đơn giản:

```text id="f0ijm0"
Một dòng = một app_version với toàn bộ chỉ số onboarding trong cửa sổ 24 giờ
```

View này không có dữ liệu chi tiết từng người chơi. Nó là bảng tổng hợp theo phiên bản game.

---

## 5. Loại đối tượng

```text id="cmlfbn"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính lại mỗi khi truy vấn.

Đặc biệt, view này dùng `CURRENT_TIMESTAMP()` để xác định thời điểm chạy truy vấn. Vì vậy, kết quả có thể thay đổi theo thời gian khi có thêm người chơi đủ 24 giờ quan sát.

---

## 6. Nguồn dữ liệu phụ thuộc

View này phụ thuộc vào ba nguồn chính:

| Nguồn                | Vai trò                                                                     |
| -------------------- | --------------------------------------------------------------------------- |
| `dim_f10_level_map`  | Xác định Level 1 theo từng phiên bản game.                                  |
| `vw_onboard_events`  | Cung cấp các sự kiện `first_open`, `Open_first`, `Vuot_home`, `app_remove`. |
| `vw_level_start_eff` | Cung cấp sự kiện bắt đầu Level 1 có hiệu lực.                               |

---

## 7. Cách xác định Level 1

View không hard-code Level 1 trực tiếp.

Trong tài liệu này, “hard-code” nghĩa là viết trực tiếp một giá trị cố định vào câu truy vấn, ví dụ luôn dùng `N001` làm Level 1.

Thay vào đó, view lấy Level 1 từ:

```text id="2zwtd6"
dim_f10_level_map
```

với điều kiện:

```sql id="kckkj0"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step = 1
```

Điều này giúp view thích nghi tốt hơn nếu Level 1 thay đổi theo phiên bản game.

---

## 8. Cách xác định nhóm người chơi đầu vào

Nhóm người chơi đầu vào là những người có sự kiện:

```text id="ywfmcf"
first_open
```

View lấy lần `first_open` đầu tiên của mỗi người chơi theo từng `app_version`.

Cụ thể:

```text id="d73k9s"
Một người chơi + một app_version
→ lấy thời điểm first_open sớm nhất
```

Chỉ các `app_version` có ánh xạ Level 1 trong `dim_f10_level_map` mới được đưa vào view.

---

## 9. Logic 24 giờ

Với mỗi người chơi trong nhóm `first_open`, view tạo cửa sổ quan sát 24 giờ:

```text id="gvpi17"
first_open_time_utc
đến
first_open_time_utc + 24 giờ
```

Các sự kiện sau được tìm trong cửa sổ này:

```text id="z7utoh"
Open_first
bắt đầu Level 1
Vuot_home
app_remove
```

Một người chơi được xem là đã đủ thời gian quan sát nếu:

```text id="4jvcnk"
first_open_time_utc <= thời điểm chạy view - 24 giờ
```

Cột tương ứng trong logic nội bộ là:

```text id="mhpr1p"
is_matured_24h
```

Trong tài liệu này, “đã đủ thời gian quan sát” nghĩa là người chơi đã qua đủ 24 giờ kể từ mốc `first_open`, nên có thể đánh giá đầy đủ hành vi trong 24 giờ đầu.

---

## 10. Các bước chính trong logic

### 10.1 Tạo mapping Level 1

View lấy `level1_name` từ `dim_f10_level_map`.

Điều kiện:

```sql id="s8jv0j"
is_active IS TRUE
AND is_required IS TRUE
AND progression_step = 1
```

---

### 10.2 Tạo nhóm `first_open`

View lấy lần `first_open` đầu tiên của mỗi người chơi theo từng phiên bản game.

Nguồn:

```text id="y1c6i5"
vw_onboard_events
```

---

### 10.3 Tính mốc 24 giờ

Với mỗi người chơi, view tạo:

```text id="1zmd1p"
first_open_24h_cutoff_time_utc
```

bằng:

```text id="dz8vu6"
first_open_time_utc + 24 giờ
```

---

### 10.4 Tìm `Open_first`

View tìm lần `Open_first` đầu tiên của mỗi người chơi trong khoảng:

```text id="5th18i"
first_open_time_utc
đến
first_open_time_utc + 24 giờ
```

---

### 10.5 Tìm bắt đầu Level 1

View tìm lần bắt đầu Level 1 đầu tiên của mỗi người chơi trong khoảng 24 giờ sau `first_open`.

Nguồn:

```text id="cczt5n"
vw_level_start_eff
```

Level 1 được xác định theo `dim_f10_level_map`.

---

### 10.6 Tìm tín hiệu chẩn đoán

View tìm thêm hai tín hiệu trong 24 giờ đầu:

```text id="72k8gh"
Vuot_home
app_remove
```

Các tín hiệu này giúp hiểu rõ hơn người chơi rơi ở đâu trong giai đoạn đầu.

---

### 10.7 Tổng hợp theo phiên bản game

Sau khi tạo dữ liệu ở cấp độ người chơi, view tổng hợp lên cấp độ:

```text id="yjiadm"
app_version
```

---

## 11. Danh sách cột

| Vị trí | Cột                                                        | Kiểu dữ liệu | Ý nghĩa                                                                                    |
| -----: | ---------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------ |
|      1 | `app_version`                                              | `STRING`     | Phiên bản game.                                                                            |
|      2 | `first_open_user_count`                                    | `INT64`      | Số người chơi có `first_open`.                                                             |
|      3 | `matured_first_open_user_count`                            | `INT64`      | Số người chơi đã đủ 24 giờ quan sát từ `first_open`.                                       |
|      4 | `not_matured_first_open_user_count`                        | `INT64`      | Số người chơi chưa đủ 24 giờ quan sát.                                                     |
|      5 | `open_first_user_count`                                    | `INT64`      | Số người chơi đã đủ 24 giờ và có `Open_first` trong 24 giờ đầu.                            |
|      6 | `level1_start_user_count`                                  | `INT64`      | Số người chơi đã đủ 24 giờ và bắt đầu Level 1 trong 24 giờ đầu.                            |
|      7 | `open_first_and_level1_start_user_count`                   | `INT64`      | Số người chơi đã đủ 24 giờ, có cả `Open_first` và bắt đầu Level 1 trong 24 giờ đầu.        |
|      8 | `first_open_without_open_first_user_count`                 | `INT64`      | Số người chơi đã đủ 24 giờ, có `first_open` nhưng không có `Open_first`.                   |
|      9 | `open_first_without_level1_user_count`                     | `INT64`      | Số người chơi đã đủ 24 giờ, có `Open_first` nhưng không bắt đầu Level 1.                   |
|     10 | `first_open_without_level1_user_count`                     | `INT64`      | Số người chơi đã đủ 24 giờ, có `first_open` nhưng không bắt đầu Level 1.                   |
|     11 | `open_first_rate_from_first_open`                          | `FLOAT64`    | Tỷ lệ người chơi đi từ `first_open` đến `Open_first`.                                      |
|     12 | `level1_start_rate_from_open_first`                        | `FLOAT64`    | Tỷ lệ người chơi có cả `Open_first` và bắt đầu Level 1, chia cho số người có `Open_first`. |
|     13 | `level1_start_rate_from_first_open`                        | `FLOAT64`    | Tỷ lệ người chơi bắt đầu Level 1, chia cho số người đã đủ 24 giờ từ `first_open`.          |
|     14 | `first_open_without_open_first_rate`                       | `FLOAT64`    | Tỷ lệ người chơi có `first_open` nhưng không có `Open_first`.                              |
|     15 | `open_first_without_level1_rate`                           | `FLOAT64`    | Tỷ lệ người chơi có `Open_first` nhưng không bắt đầu Level 1.                              |
|     16 | `first_open_without_level1_rate`                           | `FLOAT64`    | Tỷ lệ người chơi có `first_open` nhưng không bắt đầu Level 1.                              |
|     17 | `users_with_vuot_home_within_24h`                          | `INT64`      | Số người chơi có `Vuot_home` trong 24 giờ đầu.                                             |
|     18 | `first_open_without_open_first_with_vuot_home_user_count`  | `INT64`      | Số người chơi không có `Open_first` nhưng có `Vuot_home`.                                  |
|     19 | `open_first_without_level1_with_vuot_home_user_count`      | `INT64`      | Số người chơi có `Open_first`, không bắt đầu Level 1, nhưng có `Vuot_home`.                |
|     20 | `first_open_without_level1_with_vuot_home_user_count`      | `INT64`      | Số người chơi không bắt đầu Level 1 nhưng có `Vuot_home`.                                  |
|     21 | `users_with_app_remove_within_24h`                         | `INT64`      | Số người chơi có `app_remove` trong 24 giờ đầu.                                            |
|     22 | `first_open_without_open_first_with_app_remove_user_count` | `INT64`      | Số người chơi không có `Open_first` nhưng có `app_remove`.                                 |
|     23 | `open_first_without_level1_with_app_remove_user_count`     | `INT64`      | Số người chơi có `Open_first`, không bắt đầu Level 1, nhưng có `app_remove`.               |
|     24 | `first_open_without_level1_with_app_remove_user_count`     | `INT64`      | Số người chơi không bắt đầu Level 1 nhưng có `app_remove`.                                 |
|     25 | `first_open_to_open_first_timing_sample_user_count`        | `INT64`      | Số người chơi có đủ dữ liệu để tính thời gian từ `first_open` đến `Open_first`.            |
|     26 | `first_open_to_open_first_timing_coverage_rate`            | `FLOAT64`    | Tỷ lệ bao phủ của mẫu thời gian từ `first_open` đến `Open_first`.                          |
|     27 | `avg_seconds_first_open_to_open_first`                     | `FLOAT64`    | Thời gian trung bình từ `first_open` đến `Open_first`, tính bằng giây.                     |
|     28 | `median_seconds_first_open_to_open_first`                  | `INT64`      | Trung vị thời gian từ `first_open` đến `Open_first`.                                       |
|     29 | `p90_seconds_first_open_to_open_first`                     | `INT64`      | Mốc 90% thời gian từ `first_open` đến `Open_first`.                                        |
|     30 | `open_first_to_level1_start_timing_sample_user_count`      | `INT64`      | Số người chơi có đủ dữ liệu để tính thời gian từ `Open_first` đến bắt đầu Level 1.         |
|     31 | `open_first_to_level1_start_timing_coverage_rate`          | `FLOAT64`    | Tỷ lệ bao phủ của mẫu thời gian từ `Open_first` đến bắt đầu Level 1.                       |
|     32 | `avg_seconds_open_first_to_level1_start`                   | `FLOAT64`    | Thời gian trung bình từ `Open_first` đến bắt đầu Level 1.                                  |
|     33 | `median_seconds_open_first_to_level1_start`                | `INT64`      | Trung vị thời gian từ `Open_first` đến bắt đầu Level 1.                                    |
|     34 | `p90_seconds_open_first_to_level1_start`                   | `INT64`      | Mốc 90% thời gian từ `Open_first` đến bắt đầu Level 1.                                     |
|     35 | `first_open_to_level1_start_timing_sample_user_count`      | `INT64`      | Số người chơi có đủ dữ liệu để tính thời gian từ `first_open` đến bắt đầu Level 1.         |
|     36 | `first_open_to_level1_start_timing_coverage_rate`          | `FLOAT64`    | Tỷ lệ bao phủ của mẫu thời gian từ `first_open` đến bắt đầu Level 1.                       |
|     37 | `avg_seconds_first_open_to_level1_start`                   | `FLOAT64`    | Thời gian trung bình từ `first_open` đến bắt đầu Level 1.                                  |
|     38 | `median_seconds_first_open_to_level1_start`                | `INT64`      | Trung vị thời gian từ `first_open` đến bắt đầu Level 1.                                    |
|     39 | `p90_seconds_first_open_to_level1_start`                   | `INT64`      | Mốc 90% thời gian từ `first_open` đến bắt đầu Level 1.                                     |
|     40 | `first_first_open_time_utc`                                | `TIMESTAMP`  | Thời điểm `first_open` sớm nhất trong phiên bản game đó.                                   |
|     41 | `last_first_open_time_utc`                                 | `TIMESTAMP`  | Thời điểm `first_open` muộn nhất trong phiên bản game đó.                                  |
|     42 | `onboarding_view_query_time_utc`                           | `TIMESTAMP`  | Thời điểm view được truy vấn.                                                              |

---

## 12. Công thức chỉ số chính

### 12.1 Tỷ lệ từ `first_open` đến `Open_first`

```text id="f5oxio"
open_first_rate_from_first_open
=
open_first_user_count
/
matured_first_open_user_count
```

Ý nghĩa: trong số người chơi đã đủ 24 giờ quan sát, bao nhiêu phần trăm có `Open_first`.

---

### 12.2 Tỷ lệ từ `Open_first` đến Level 1

```text id="5z9abd"
level1_start_rate_from_open_first
=
open_first_and_level1_start_user_count
/
open_first_user_count
```

Ý nghĩa: trong số người chơi có `Open_first`, bao nhiêu phần trăm cũng có bắt đầu Level 1 trong 24 giờ đầu.

Lưu ý: chỉ số này đếm người chơi có cả hai sự kiện trong cửa sổ 24 giờ. Phần tính thời gian từ `Open_first` đến Level 1 mới yêu cầu Level 1 xảy ra sau `Open_first`.

---

### 12.3 Tỷ lệ từ `first_open` đến Level 1

```text id="u0ox3l"
level1_start_rate_from_first_open
=
level1_start_user_count
/
matured_first_open_user_count
```

Ý nghĩa: trong số người chơi đã đủ 24 giờ quan sát, bao nhiêu phần trăm bắt đầu Level 1.

---

### 12.4 Tỷ lệ rơi trước `Open_first`

```text id="g8225e"
first_open_without_open_first_rate
=
first_open_without_open_first_user_count
/
matured_first_open_user_count
```

Ý nghĩa: tỷ lệ người chơi mở game lần đầu nhưng không đi tới `Open_first`.

---

### 12.5 Tỷ lệ có `Open_first` nhưng không bắt đầu Level 1

```text id="30jmyr"
open_first_without_level1_rate
=
open_first_without_level1_user_count
/
open_first_user_count
```

Ý nghĩa: tỷ lệ người chơi đã có `Open_first` nhưng không bắt đầu Level 1.

---

### 12.6 Tỷ lệ từ `first_open` nhưng không bắt đầu Level 1

```text id="1hys2y"
first_open_without_level1_rate
=
first_open_without_level1_user_count
/
matured_first_open_user_count
```

Ý nghĩa: tỷ lệ người chơi mở game lần đầu nhưng không bắt đầu Level 1 trong 24 giờ đầu.

---

## 13. Lưu ý về cột trùng tên

Trong schema hiện tại, cần chú ý các nhóm cột dễ gây nhầm lẫn:

```text id="xgbbkn"
open_first_without_level1_user_count
first_open_without_level1_user_count
open_first_without_level1_rate
first_open_without_level1_rate
```

Hai cặp chỉ số này có ý nghĩa khác nhau:

| Chỉ số                                 | Mẫu số logic                                         |
| -------------------------------------- | ---------------------------------------------------- |
| `open_first_without_level1_user_count` | Người chơi có `Open_first`.                          |
| `first_open_without_level1_user_count` | Người chơi có `first_open` và đã đủ 24 giờ quan sát. |
| `open_first_without_level1_rate`       | Chia cho `open_first_user_count`.                    |
| `first_open_without_level1_rate`       | Chia cho `matured_first_open_user_count`.            |

Khi đưa vào dashboard, cần đặt nhãn tiếng Việt rõ ràng để tránh hiểu nhầm.

Trong tài liệu này, “dashboard” nghĩa là bảng điều khiển hoặc màn hình báo cáo trực quan dùng để theo dõi chỉ số.

---

## 14. Kiểm tra chất lượng dữ liệu khuyến nghị

### 14.1 Kiểm tra app version có đủ người chơi đã trưởng thành 24 giờ

```sql id="4ekv4d"
SELECT
  app_version,
  first_open_user_count,
  matured_first_open_user_count,
  not_matured_first_open_user_count,

  ROUND(
    SAFE_DIVIDE(
      matured_first_open_user_count,
      first_open_user_count
    ),
    4
  ) AS matured_user_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

Nếu `matured_user_rate` thấp, các chỉ số chuyển đổi 24 giờ chưa ổn định.

---

### 14.2 Kiểm tra tỷ lệ chuyển đổi chính

```sql id="ajzgd6"
SELECT
  app_version,

  first_open_user_count,
  matured_first_open_user_count,

  open_first_user_count,
  open_first_rate_from_first_open,

  level1_start_user_count,
  level1_start_rate_from_first_open,

  level1_start_rate_from_open_first,

  first_open_without_open_first_rate,
  open_first_without_level1_rate,
  first_open_without_level1_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

Truy vấn này dùng để xem nhanh sức khỏe của phễu người chơi mới.

---

### 14.3 Kiểm tra tín hiệu `Vuot_home` và `app_remove`

```sql id="4m8zbv"
SELECT
  app_version,

  users_with_vuot_home_within_24h,
  first_open_without_open_first_with_vuot_home_user_count,
  open_first_without_level1_with_vuot_home_user_count,
  first_open_without_level1_with_vuot_home_user_count,

  users_with_app_remove_within_24h,
  first_open_without_open_first_with_app_remove_user_count,
  open_first_without_level1_with_app_remove_user_count,
  first_open_without_level1_with_app_remove_user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

Truy vấn này giúp phân biệt rơi do chưa đi qua bước đầu, rơi sau bước đầu, hoặc có tín hiệu gỡ game.

---

### 14.4 Kiểm tra độ phủ mẫu thời gian

```sql id="fhbk0x"
SELECT
  app_version,

  first_open_to_open_first_timing_sample_user_count,
  first_open_to_open_first_timing_coverage_rate,

  open_first_to_level1_start_timing_sample_user_count,
  open_first_to_level1_start_timing_coverage_rate,

  first_open_to_level1_start_timing_sample_user_count,
  first_open_to_level1_start_timing_coverage_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

Nếu tỷ lệ bao phủ mẫu thời gian thấp, không nên kết luận mạnh về thời gian trung bình hoặc trung vị.

---

## 15. Cách sử dụng khuyến nghị

### 15.1 Theo dõi phễu người chơi mới theo phiên bản

```sql id="cixk1z"
SELECT
  app_version,

  matured_first_open_user_count,

  open_first_rate_from_first_open,
  level1_start_rate_from_open_first,
  level1_start_rate_from_first_open,

  first_open_without_open_first_rate,
  open_first_without_level1_rate,
  first_open_without_level1_rate

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

---

### 15.2 Theo dõi thời gian đi qua các bước đầu

```sql id="jifxgt"
SELECT
  app_version,

  avg_seconds_first_open_to_open_first,
  median_seconds_first_open_to_open_first,
  p90_seconds_first_open_to_open_first,

  avg_seconds_open_first_to_level1_start,
  median_seconds_open_first_to_level1_start,
  p90_seconds_open_first_to_level1_start,

  avg_seconds_first_open_to_level1_start,
  median_seconds_first_open_to_level1_start,
  p90_seconds_first_open_to_level1_start

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

---

### 15.3 Tạo cảnh báo nếu rơi trước Level 1 quá cao

```sql id="mvx2i3"
SELECT
  app_version,
  matured_first_open_user_count,
  first_open_without_level1_rate,

  CASE
    WHEN matured_first_open_user_count < 100 THEN 'Chưa đủ mẫu'
    WHEN first_open_without_level1_rate >= 0.5 THEN 'Rủi ro cao'
    WHEN first_open_without_level1_rate >= 0.3 THEN 'Cần theo dõi'
    ELSE 'Tạm ổn'
  END AS onboarding_health_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`

ORDER BY
  app_version;
```

Ngưỡng trong truy vấn này chỉ là ví dụ. Team nên điều chỉnh theo chuẩn nội bộ.

---

## 16. Lưu ý khi sử dụng

### 16.1 View chỉ tính người chơi đã đủ 24 giờ cho các chỉ số chuyển đổi

Các chỉ số chuyển đổi chính dùng mẫu số:

```text id="d2wmp6"
matured_first_open_user_count
```

Không nên dùng `first_open_user_count` làm mẫu số cho các chỉ số 24 giờ nếu chưa hiểu rõ logic.

---

### 16.2 Kết quả có thể thay đổi theo thời điểm truy vấn

View dùng:

```text id="yie0ms"
CURRENT_TIMESTAMP()
```

để xác định người chơi đã đủ 24 giờ hay chưa.

Vì vậy, cùng một câu truy vấn có thể trả kết quả khác nhau ở các thời điểm khác nhau.

Cột sau cho biết thời điểm view được tính:

```text id="0bg7f3"
onboarding_view_query_time_utc
```

---

### 16.3 App version không có mapping Level 1 sẽ bị loại khỏi cohort

View chỉ tạo nhóm `first_open` cho các `app_version` có ánh xạ Level 1 trong `dim_f10_level_map`.

Nếu một phiên bản game không xuất hiện trong view này, cần kiểm tra:

```text id="onxukd"
dim_f10_level_map
```

trước khi kết luận là phiên bản đó không có người chơi.

---

### 16.4 View này là bảng tổng hợp, không dùng để debug từng người chơi

Nếu cần kiểm tra từng người chơi, cần truy vấn trực tiếp các view nguồn:

```text id="lqpbhn"
vw_onboard_events
vw_level_start_eff
```

---

## 17. Rủi ro nếu view sai logic

| Lỗi logic                              | Ảnh hưởng                                               |
| -------------------------------------- | ------------------------------------------------------- |
| Sai mapping Level 1                    | Tỷ lệ bắt đầu Level 1 sẽ sai.                           |
| Sai logic 24 giờ                       | Có thể tính nhầm người chơi chưa đủ thời gian quan sát. |
| Lọc thiếu `first_open`                 | Mẫu đầu vào bị thiếu.                                   |
| Lọc thiếu `Open_first`                 | Tỷ lệ bước đầu bị thấp giả.                             |
| Lọc thiếu Level 1 start                | Tỷ lệ vào gameplay bị thấp giả.                         |
| Không phân biệt matured và not matured | Chỉ số 24 giờ bị sai.                                   |
| Không theo dõi độ phủ mẫu thời gian    | Có thể kết luận sai từ mẫu quá nhỏ.                     |
| Không kiểm tra `app_remove`            | Bỏ lỡ tín hiệu rời bỏ sớm.                              |

---

## 18. Mức độ quan trọng

```text id="9439fo"
Mức độ quan trọng: Cao
```

Lý do:

* view này là báo cáo chính cho giai đoạn người chơi mới;
* view này giúp đánh giá chất lượng onboarding;
* view này phát hiện rơi trước khi người chơi bắt đầu gameplay;
* view này so sánh được onboarding giữa các phiên bản game;
* view này phụ thuộc vào mapping Level 1 nên cần được kiểm tra sau mỗi thay đổi progression đầu game.

---

## 19. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="e0tujh"
sql/ddl/current_live_analysis/vw_onboard_funnel_24h.sql
```

Cấu trúc logic chính của view:

```sql id="n7j6vx"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_funnel_24h`
AS
WITH analysis_params AS (
  SELECT
    CURRENT_TIMESTAMP() AS analysis_time_utc
),

level1_mapping AS (
  SELECT
    app_version,
    level_name AS level1_name

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

  WHERE
    is_active IS TRUE
    AND is_required IS TRUE
    AND progression_step = 1
),

first_open_by_user AS (...),

first_open_cohort AS (...),

open_first_by_user AS (...),

level1_start_by_user AS (...),

diagnostic_signals_by_user AS (...),

user_onboarding_funnel AS (...),

onboarding_metrics_by_app_version AS (...)

SELECT
  -- chỉ số phễu
  -- chỉ số rơi
  -- chỉ số Vuot_home
  -- chỉ số app_remove
  -- chỉ số thời gian
  -- thời điểm truy vấn

FROM
  onboarding_metrics_by_app_version;
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream:
	- [[dim_f10_level_map]]
	- [[vw_onboard_events]]
	- [[vw_level_start_eff]]

- Downstream: Không có