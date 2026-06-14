# vw_onboard_events

## 1. Mục đích

`vw_onboard_events` là view chuẩn hóa các sự kiện phục vụ phân tích giai đoạn người chơi mới.

Trong tài liệu này, “view” nghĩa là bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi khi gọi view, BigQuery sẽ chạy câu truy vấn đó để trả dữ liệu.

View này lấy ra một tập sự kiện quan trọng trong giai đoạn đầu của người chơi, gồm:

```text id="ilcix4"
first_open
Open_first
Vuot_home
app_remove
```

Mục đích là tạo một nguồn dữ liệu gọn và nhất quán để phân tích:

* người chơi mở game lần đầu;
* người chơi đi qua bước mở đầu;
* người chơi đi tới màn hình chính hoặc vượt qua đoạn đầu;
* người chơi gỡ ứng dụng trong giai đoạn đầu.

---

## 2. Vai trò trong hệ thống

`vw_onboard_events` thuộc nhóm:

```text id="ry1xqh"
core_foundation
```

Tức là nhóm nền tảng lõi.

View này là nguồn dữ liệu chính cho các phân tích người chơi mới, đặc biệt là:

* phễu từ `first_open` đến `Open_first`;
* phễu từ `Open_first` đến bắt đầu Level 1;
* chẩn đoán người chơi rơi khỏi giai đoạn đầu;
* kiểm tra người chơi có gỡ game trong 24 giờ đầu hay không.

Trong tài liệu này, “phễu” nghĩa là chuỗi bước mà người chơi cần đi qua. Ví dụ:

```text id="pi0pm2"
first_open
  ↓
Open_first
  ↓
bắt đầu Level 1
```

---

## 3. Câu hỏi phân tích có thể trả lời

View này giúp trả lời các câu hỏi sau:

1. Có bao nhiêu người chơi mở game lần đầu?
2. Có bao nhiêu người chơi đi từ `first_open` đến `Open_first`?
3. Có bao nhiêu người chơi có sự kiện `Vuot_home`?
4. Có bao nhiêu người chơi có sự kiện `app_remove`?
5. Thời gian từ `first_open` đến `Open_first` là bao lâu?
6. Thời gian từ `first_open` đến `Vuot_home` là bao lâu?
7. Người chơi có gỡ ứng dụng trong vòng 24 giờ đầu không?
8. Có phiên bản game nào bị rơi mạnh ở giai đoạn người chơi mới không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `vw_onboard_events` đại diện cho một sự kiện liên quan đến giai đoạn người chơi mới.

Có thể hiểu đơn giản:

```text id="a5mfce"
Một dòng = một sự kiện onboarding của một người chơi tại một thời điểm
```

Trong đó, “onboarding” nghĩa là giai đoạn đầu khi người chơi mới bắt đầu vào game và đi qua các bước đầu tiên.

Một người chơi có thể có nhiều dòng trong view này nếu người đó có nhiều sự kiện khác nhau, ví dụ:

```text id="n5kip9"
first_open
Open_first
Vuot_home
app_remove
```

---

## 5. Loại đối tượng

```text id="4lmql0"
Loại đối tượng: VIEW
```

View này không lưu dữ liệu vật lý. Kết quả được tính trực tiếp từ dữ liệu GA4 thô mỗi khi truy vấn.

---

## 6. Nguồn dữ liệu phụ thuộc

View này đọc trực tiếp từ dữ liệu GA4 thô:

```text id="6t1iw2"
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Trong đó:

* `events_*` là các bảng sự kiện GA4 được chia theo ngày;
* `_TABLE_SUFFIX` là hậu tố ngày của bảng, ví dụ `20260531`;
* view chỉ lấy các bảng có hậu tố đúng dạng 8 chữ số;
* dữ liệu được lọc theo các sự kiện phục vụ phân tích người chơi mới.

---

## 7. Các view đang sử dụng view này

`vw_onboard_events` được sử dụng bởi các view sau:

| View sử dụng               | Mục đích sử dụng                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------------------- |
| `vw_onboard_funnel_24h`    | Đo phễu người chơi mới trong vòng 24 giờ từ `first_open` đến `Open_first` và bắt đầu Level 1.             |
| `vw_onboard_drop_diag_24h` | Phân loại nhóm người chơi rơi trong giai đoạn đầu và kiểm tra các tín hiệu như `Vuot_home`, `app_remove`. |

Do view này là nguồn chính cho phân tích người chơi mới, nếu logic lọc sự kiện sai thì toàn bộ phân tích onboarding phía sau sẽ sai.

---

## 8. Danh sách cột

| Cột                   | Kiểu dữ liệu | Ý nghĩa                                                                                        |
| --------------------- | ------------ | ---------------------------------------------------------------------------------------------- |
| `onboarding_event_id` | `STRING`     | Mã định danh của sự kiện onboarding. Mã này được tạo để nhận diện một dòng sự kiện trong view. |
| `event_table_suffix`  | `STRING`     | Hậu tố ngày của bảng GA4 gốc, ví dụ `20260531`.                                                |
| `event_date`          | `STRING`     | Ngày sự kiện theo định dạng của GA4.                                                           |
| `event_time_utc`      | `TIMESTAMP`  | Thời điểm xảy ra sự kiện theo múi giờ UTC.                                                     |
| `user_pseudo_id`      | `STRING`     | Mã định danh ẩn danh của người chơi trong GA4.                                                 |
| `app_version`         | `STRING`     | Phiên bản game tại thời điểm sự kiện xảy ra.                                                   |
| `event_name`          | `STRING`     | Tên sự kiện, ví dụ `first_open`, `Open_first`, `Vuot_home`, `app_remove`.                      |

---

## 9. Trường định danh quan trọng

Trường định danh chính của view là:

```text id="d1pwiu"
onboarding_event_id
```

Trường này được dùng để nhận diện một dòng sự kiện onboarding.

Các trường quan trọng khác:

```text id="w5lxat"
user_pseudo_id
event_time_utc
app_version
event_name
```

Trong đó:

* `user_pseudo_id` cho biết sự kiện thuộc về người chơi nào;
* `event_time_utc` cho biết sự kiện xảy ra khi nào;
* `app_version` cho biết sự kiện thuộc phiên bản game nào;
* `event_name` cho biết loại sự kiện là gì.

---

## 10. Logic xử lý dữ liệu

### 10.1 Lọc bảng GA4 theo hậu tố ngày hợp lệ

View chỉ đọc các bảng có hậu tố ngày đúng định dạng:

```sql id="m6pwgk"
REGEXP_CONTAINS(_TABLE_SUFFIX, r'^\d{8}$')
```

Điều này giúp loại bỏ các bảng không phải bảng sự kiện theo ngày.

---

### 10.2 Lọc người chơi hợp lệ

View yêu cầu:

```sql id="kf4hnk"
user_pseudo_id IS NOT NULL
```

Điều này đảm bảo mỗi sự kiện có thể được gắn với một người chơi ẩn danh.

Nếu thiếu `user_pseudo_id`, sự kiện đó không thể dùng tốt cho phân tích theo người chơi.

---

### 10.3 Lọc các sự kiện onboarding

View chỉ giữ lại các sự kiện:

```sql id="f54o1j"
event_name IN (
  'first_open',
  'Open_first',
  'Vuot_home',
  'app_remove'
)
```

Lưu ý: tên sự kiện có phân biệt chữ hoa và chữ thường.

Ví dụ:

```text id="kiw3gd"
first_open
```

khác với:

```text id="piis1v"
Open_first
```

Vì vậy khi tracking hoặc truy vấn dữ liệu, cần dùng đúng tên sự kiện.

---

### 10.4 Chuẩn hóa thời gian sự kiện

GA4 lưu thời điểm sự kiện bằng `event_timestamp`.

View chuyển thời điểm này sang dạng `TIMESTAMP` ở múi giờ UTC và đặt tên là:

```text id="q5m31z"
event_time_utc
```

UTC là múi giờ chuẩn quốc tế, thường được dùng trong hệ thống dữ liệu để tránh sai lệch khi so sánh thời gian giữa nhiều khu vực.

---

## 11. Ý nghĩa từng sự kiện

### 11.1 `first_open`

`first_open` là sự kiện cho biết người chơi mở ứng dụng lần đầu theo ghi nhận của GA4.

Sự kiện này thường được dùng làm mốc bắt đầu cho nhóm người chơi mới.

Trong tài liệu này, “nhóm người chơi mới” là nhóm người chơi có cùng mốc bắt đầu là lần mở game đầu tiên.

---

### 11.2 `Open_first`

`Open_first` là sự kiện do game tracking để ghi nhận một bước mở đầu trong luồng vào game.

Sự kiện này được dùng để kiểm tra người chơi có đi tiếp sau `first_open` hay không.

Nếu nhiều người có `first_open` nhưng không có `Open_first`, có thể có vấn đề ở giai đoạn rất sớm như:

* loading;
* màn hình khởi động;
* lỗi tracking;
* crash;
* người chơi thoát ngay;
* logic gửi event chưa đúng.

---

### 11.3 `Vuot_home`

`Vuot_home` là sự kiện liên quan đến việc người chơi đi tới hoặc vượt qua một mốc đầu game, có thể liên quan đến màn hình chính hoặc điểm chuyển tiếp sau giai đoạn đầu.

Sự kiện này được dùng như tín hiệu chẩn đoán trong `vw_onboard_drop_diag_24h`.

---

### 11.4 `app_remove`

`app_remove` là sự kiện cho biết người chơi gỡ ứng dụng.

Sự kiện này được dùng để kiểm tra khả năng người chơi rời bỏ game rất sớm.

Cần lưu ý rằng `app_remove` phụ thuộc vào cách GA4 và nền tảng ghi nhận, nên không nên chỉ dựa vào riêng sự kiện này để kết luận toàn bộ nguyên nhân rời bỏ.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

### 12.1 Kiểm tra số lượng sự kiện theo ngày

```sql id="wqpcdy"
SELECT
  event_date,
  app_version,
  event_name,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

GROUP BY
  event_date,
  app_version,
  event_name

ORDER BY
  event_date,
  app_version,
  event_name;
```

Truy vấn này giúp theo dõi khối lượng sự kiện onboarding theo ngày và phiên bản game.

---

### 12.2 Kiểm tra trùng `onboarding_event_id`

```sql id="eu5wgw"
SELECT
  onboarding_event_id,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

GROUP BY
  onboarding_event_id

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC;
```

Kỳ vọng:

```text id="jq5s7j"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là mã định danh sự kiện không đủ duy nhất hoặc dữ liệu nguồn có dòng trùng.

---

### 12.3 Kiểm tra thiếu phiên bản game

```sql id="t3oy8f"
SELECT
  event_date,
  event_name,
  COUNT(*) AS null_app_version_event_count,
  COUNT(DISTINCT user_pseudo_id) AS affected_user_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

WHERE
  app_version IS NULL

GROUP BY
  event_date,
  event_name

ORDER BY
  event_date,
  event_name;
```

Nếu có nhiều sự kiện thiếu `app_version`, các phân tích theo phiên bản game sẽ bị giảm độ tin cậy.

---

### 12.4 Kiểm tra người chơi có `first_open` nhưng không có `Open_first`

```sql id="6n9rfs"
WITH first_open_users AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS first_open_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'first_open'

  GROUP BY
    app_version,
    user_pseudo_id
),

open_first_users AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS open_first_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'Open_first'

  GROUP BY
    app_version,
    user_pseudo_id
)

SELECT
  first_open_users.app_version,

  COUNT(DISTINCT first_open_users.user_pseudo_id)
    AS first_open_user_count,

  COUNT(DISTINCT open_first_users.user_pseudo_id)
    AS open_first_user_count,

  COUNT(DISTINCT IF(
    open_first_users.user_pseudo_id IS NULL,
    first_open_users.user_pseudo_id,
    NULL
  )) AS first_open_without_open_first_user_count

FROM
  first_open_users

LEFT JOIN
  open_first_users
ON
  first_open_users.app_version = open_first_users.app_version
  AND first_open_users.user_pseudo_id = open_first_users.user_pseudo_id
  AND open_first_users.open_first_time_utc >= first_open_users.first_open_time_utc

GROUP BY
  first_open_users.app_version

ORDER BY
  first_open_users.app_version;
```

Truy vấn này giúp kiểm tra điểm rơi đầu tiên trong luồng người chơi mới.

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Tạo nhóm người chơi mở game lần đầu

```sql id="9n9agk"
SELECT
  app_version,
  user_pseudo_id,
  MIN(event_time_utc) AS first_open_time_utc

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

WHERE
  event_name = 'first_open'

GROUP BY
  app_version,
  user_pseudo_id;
```

### 13.2 Đo thời gian từ `first_open` đến `Open_first`

```sql id="qticbz"
WITH first_open_by_user AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS first_open_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'first_open'

  GROUP BY
    app_version,
    user_pseudo_id
),

open_first_by_user AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS open_first_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'Open_first'

  GROUP BY
    app_version,
    user_pseudo_id
)

SELECT
  first_open_by_user.app_version,

  COUNT(DISTINCT first_open_by_user.user_pseudo_id)
    AS first_open_user_count,

  COUNT(DISTINCT open_first_by_user.user_pseudo_id)
    AS open_first_user_count,

  AVG(TIMESTAMP_DIFF(
    open_first_by_user.open_first_time_utc,
    first_open_by_user.first_open_time_utc,
    SECOND
  )) AS avg_seconds_first_open_to_open_first

FROM
  first_open_by_user

LEFT JOIN
  open_first_by_user
ON
  first_open_by_user.app_version = open_first_by_user.app_version
  AND first_open_by_user.user_pseudo_id = open_first_by_user.user_pseudo_id
  AND open_first_by_user.open_first_time_utc >= first_open_by_user.first_open_time_utc

GROUP BY
  first_open_by_user.app_version

ORDER BY
  first_open_by_user.app_version;
```

### 13.3 Đếm người chơi gỡ game trong 24 giờ đầu

```sql id="0r8mcr"
WITH first_open_by_user AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS first_open_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'first_open'

  GROUP BY
    app_version,
    user_pseudo_id
),

app_remove_by_user AS (
  SELECT
    app_version,
    user_pseudo_id,
    MIN(event_time_utc) AS app_remove_time_utc

  FROM
    `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`

  WHERE
    event_name = 'app_remove'

  GROUP BY
    app_version,
    user_pseudo_id
)

SELECT
  first_open_by_user.app_version,

  COUNT(DISTINCT first_open_by_user.user_pseudo_id)
    AS first_open_user_count,

  COUNT(DISTINCT IF(
    app_remove_by_user.app_remove_time_utc
      BETWEEN first_open_by_user.first_open_time_utc
      AND TIMESTAMP_ADD(first_open_by_user.first_open_time_utc, INTERVAL 24 HOUR),
    first_open_by_user.user_pseudo_id,
    NULL
  )) AS app_remove_within_24h_user_count

FROM
  first_open_by_user

LEFT JOIN
  app_remove_by_user
ON
  first_open_by_user.app_version = app_remove_by_user.app_version
  AND first_open_by_user.user_pseudo_id = app_remove_by_user.user_pseudo_id

GROUP BY
  first_open_by_user.app_version

ORDER BY
  first_open_by_user.app_version;
```

---

## 14. Lưu ý khi sử dụng

### 14.1 Đây là view sự kiện, chưa phải view phễu

`vw_onboard_events` chỉ gom và chuẩn hóa sự kiện.

Nếu muốn xem phễu chuyển đổi hoàn chỉnh, nên dùng:

```text id="bxdt6j"
vw_onboard_funnel_24h
```

Nếu muốn xem chẩn đoán rơi trong giai đoạn người chơi mới, nên dùng:

```text id="k6fvdo"
vw_onboard_drop_diag_24h
```

---

### 14.2 Sự kiện có phân biệt chữ hoa và chữ thường

Cần dùng đúng tên:

```text id="y4n150"
first_open
Open_first
Vuot_home
app_remove
```

Không nên tự đổi thành dạng chữ thường toàn bộ nếu chưa kiểm tra logic tracking.

---

### 14.3 Không nên kết luận nguyên nhân rời bỏ chỉ từ một sự kiện

Nếu người chơi có `app_remove`, có thể xem đó là tín hiệu rời bỏ mạnh.

Tuy nhiên, nếu người chơi không có `app_remove`, không có nghĩa chắc chắn là người đó còn chơi.

Cần kết hợp thêm:

* `session_start`;
* `user_engagement`;
* level progression;
* retention;
* dữ liệu quay lại ngày sau.

Trong tài liệu này, “retention” có thể hiểu là tỷ lệ người chơi quay lại game sau một khoảng thời gian, ví dụ ngày hôm sau hoặc sau 7 ngày.

---

## 15. Rủi ro nếu view sai logic

| Lỗi logic                   | Ảnh hưởng                                               |
| --------------------------- | ------------------------------------------------------- |
| Lọc thiếu `first_open`      | Sai số lượng người chơi mới.                            |
| Lọc thiếu `Open_first`      | Sai tỷ lệ đi qua bước mở đầu.                           |
| Lọc thiếu `Vuot_home`       | Thiếu tín hiệu chẩn đoán trong giai đoạn đầu.           |
| Lọc thiếu `app_remove`      | Đánh giá thấp tỷ lệ gỡ game sớm.                        |
| Sai `event_time_utc`        | Sai toàn bộ phân tích thời gian giữa các bước.          |
| Thiếu `app_version`         | Không phân tích được khác biệt giữa các phiên bản game. |
| Trùng `onboarding_event_id` | Có thể làm sai số lượng sự kiện.                        |

---

## 16. Mức độ quan trọng

```text id="zf58rk"
Mức độ quan trọng: Cao
```

Lý do:

* view này là nguồn chính cho phân tích người chơi mới;
* view này ảnh hưởng trực tiếp đến phễu onboarding;
* view này ảnh hưởng trực tiếp đến chẩn đoán rơi trong 24 giờ đầu;
* nếu view này sai, các view phân tích phía sau sẽ sai theo.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc view trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="xrioat"
sql/ddl/core_foundation/vw_onboard_events.sql
```

Cấu trúc logic chính của view:

```sql id="tze1id"
CREATE VIEW `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.vw_onboard_events`
AS
SELECT
  onboarding_event_id,
  event_table_suffix,
  event_date,
  event_time_utc,
  user_pseudo_id,
  app_version,
  event_name
FROM
  `project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*`
WHERE
  event_name IN (
    'first_open',
    'Open_first',
    'Vuot_home',
    'app_remove'
  );
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Nguồn ngoài mart: `analytics_524104373.events_*`

- Upstream: Không có

- Downstream:
	- [[vw_onboard_funnel_24h]]
	- [[vw_onboard_drop_diag_24h]]

