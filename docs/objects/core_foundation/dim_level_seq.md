# dim_level_seq

## 1. Mục đích

`dim_level_seq` là bảng mô tả thứ tự và đặc điểm của các level trong game.

Bảng này dùng để quản lý thông tin tổng quát về chuỗi level, bao gồm:

* level thuộc nhóm nào;
* level thuộc loại nào;
* tên level là gì;
* số thứ tự level là gì;
* level đứng ở vị trí nào trong chuỗi;
* level trước và level sau là gì;
* level có thuộc tiến trình chính hay không;
* level có bắt buộc trong chuỗi hay không;
* level có liên quan đến hướng dẫn hay không;
* level có phải level đặc biệt hay không;
* level được quan sát trong dữ liệu từ ngày nào đến ngày nào;
* số lượng sự kiện đã quan sát được cho level đó.

Nếu `dim_f10_level_map` tập trung vào 10 level đầu, thì `dim_level_seq` có phạm vi rộng hơn: mô tả chuỗi level tổng quát của game.

---

## 2. Vai trò trong hệ thống

`dim_level_seq` thuộc nhóm:

```text id="yu3qoe"
core_foundation
```

Tức là nhóm nền tảng lõi.

Bảng này hiện là bảng mô tả cấu trúc level, chưa phải là bảng kết quả phân tích hành vi người chơi.

Trong kiến trúc hiện tại, `dim_level_seq` chưa được các view phân tích khác sử dụng trực tiếp. Tuy nhiên, bảng này có vai trò quan trọng cho các hướng mở rộng sau:

* phân tích toàn bộ tiến trình level, không chỉ 10 level đầu;
* kiểm tra tính liền mạch của chuỗi level;
* phân loại level thường, level hướng dẫn, level đặc biệt;
* đối chiếu dữ liệu tracking với thiết kế level;
* xây dựng các phễu tiến trình dài hơn;
* hỗ trợ phân tích giữ chân người chơi theo đoạn level.

Trong tài liệu này, “chuỗi level” nghĩa là thứ tự các level mà người chơi dự kiến đi qua trong game.

---

## 3. Câu hỏi phân tích có thể trả lời

Bảng này giúp trả lời các câu hỏi nền tảng sau:

1. Một level thuộc nhóm nào?
2. Một level là level thường, level hướng dẫn hay level đặc biệt?
3. Thứ tự của level trong chuỗi là gì?
4. Level trước và level sau của một level là gì?
5. Level có thuộc tiến trình chính hay không?
6. Level có bắt buộc trong chuỗi hay không?
7. Level có liên quan đến hướng dẫn hay không?
8. Level được quan sát lần đầu trong dữ liệu vào ngày nào?
9. Level được quan sát lần cuối trong dữ liệu vào ngày nào?
10. Level có bao nhiêu sự kiện `Start_level`, `End_level`, hoặc sự kiện hướng dẫn?
11. Chuỗi level có bị thiếu hoặc đứt đoạn không?
12. Dữ liệu tracking có xuất hiện những level chưa nằm trong chuỗi thiết kế không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `dim_level_seq` đại diện cho một level trong một chuỗi level.

Có thể hiểu đơn giản:

```text id="xekiqg"
Một dòng = một level được mô tả trong chuỗi level tổng quát của game
```

Bảng này không phải dữ liệu theo người chơi, không phải dữ liệu theo lượt chơi, và không phải dữ liệu theo ngày. Đây là bảng mô tả cấu trúc level.

---

## 5. Loại đối tượng

```text id="72ru3e"
Loại đối tượng: BASE TABLE
```

Trong BigQuery, `BASE TABLE` nghĩa là bảng vật lý có lưu dữ liệu thật.

Khác với view, bảng này không tự chạy lại câu truy vấn mỗi khi được gọi. Dữ liệu trong bảng cần được tạo hoặc cập nhật bởi một quy trình riêng.

---

## 6. Nguồn dữ liệu phụ thuộc

`dim_level_seq` là bảng vật lý. Bản thân bảng này không phụ thuộc trực tiếp vào view nào trong kiến trúc hiện tại.

Tuy nhiên, dựa trên các cột hiện có, bảng này nhiều khả năng được xây dựng từ việc kết hợp:

* dữ liệu thiết kế level;
* dữ liệu quan sát từ các sự kiện level;
* dữ liệu quan sát từ các sự kiện hướng dẫn;
* đánh giá và ghi chú của người phân tích dữ liệu.

Các cột sau cho thấy bảng có yếu tố quản trị và kiểm chứng dữ liệu:

* `sequence_source`;
* `confidence_status`;
* `analyst_note`;
* `created_at`;
* `updated_at`.

---

## 7. Các view đang sử dụng bảng này

Trong kiến trúc hiện tại, chưa có view nào phụ thuộc trực tiếp vào `dim_level_seq`.

Điều này có nghĩa là:

* thay đổi bảng này hiện chưa ảnh hưởng trực tiếp đến các view phân tích đang chạy;
* bảng này vẫn nên được duy trì vì có thể dùng cho các phân tích mở rộng;
* nếu sau này xây dựng phân tích toàn bộ chuỗi level, bảng này có thể trở thành bảng tham chiếu quan trọng.

---

## 8. Danh sách cột

| Cột                             | Kiểu dữ liệu | Ý nghĩa                                                                         |
| ------------------------------- | ------------ | ------------------------------------------------------------------------------- |
| `sequence_id`                   | `STRING`     | Mã định danh của dòng trong chuỗi level.                                        |
| `level_group`                   | `STRING`     | Nhóm level. Ví dụ: nhóm level thường, nhóm level sự kiện, nhóm level đặc biệt.  |
| `level_type`                    | `STRING`     | Loại level. Ví dụ: level thường, level hướng dẫn, level đặc biệt.               |
| `level_name`                    | `STRING`     | Tên level trong tracking, ví dụ `N001`, `N002`, `c001`.                         |
| `level_prefix`                  | `STRING`     | Phần tiền tố của tên level. Ví dụ: `N`, `c`, `E`.                               |
| `level_number`                  | `INT64`      | Số level được tách ra từ tên level nếu có. Ví dụ: `N001` có `level_number = 1`. |
| `sequence_index`                | `INT64`      | Vị trí của level trong chuỗi tổng quát.                                         |
| `previous_level_name`           | `STRING`     | Tên level đứng trước theo chuỗi đã xác định.                                    |
| `next_level_name`               | `STRING`     | Tên level đứng sau theo chuỗi đã xác định.                                      |
| `inferred_previous_level_name`  | `STRING`     | Tên level trước được suy luận từ dữ liệu hoặc quy tắc.                          |
| `inferred_next_level_name`      | `STRING`     | Tên level sau được suy luận từ dữ liệu hoặc quy tắc.                            |
| `is_active`                     | `BOOL`       | Cho biết level này có đang được dùng trong chuỗi hiện tại hay không.            |
| `is_main_progression`           | `BOOL`       | Cho biết level có thuộc tiến trình chính của game hay không.                    |
| `is_required_in_sequence`       | `BOOL`       | Cho biết level có bắt buộc trong chuỗi hay không.                               |
| `is_tutorial_related`           | `BOOL`       | Cho biết level có liên quan đến hướng dẫn hay không.                            |
| `is_special_level`              | `BOOL`       | Cho biết level có phải level đặc biệt hay không.                                |
| `effective_from_app_version`    | `STRING`     | Phiên bản game bắt đầu áp dụng dòng mô tả này.                                  |
| `effective_to_app_version`      | `STRING`     | Phiên bản game kết thúc áp dụng dòng mô tả này.                                 |
| `first_seen_event_date`         | `STRING`     | Ngày đầu tiên level được quan sát trong dữ liệu sự kiện.                        |
| `last_seen_event_date`          | `STRING`     | Ngày cuối cùng level được quan sát trong dữ liệu sự kiện.                       |
| `observed_level_event_count`    | `INT64`      | Tổng số sự kiện level đã quan sát được liên quan đến level này.                 |
| `observed_start_level_count`    | `INT64`      | Số sự kiện `Start_level` đã quan sát được.                                      |
| `observed_end_level_count`      | `INT64`      | Số sự kiện `End_level` đã quan sát được.                                        |
| `observed_tutorial_event_count` | `INT64`      | Số sự kiện hướng dẫn đã quan sát được.                                          |
| `sequence_source`               | `STRING`     | Nguồn xác định chuỗi level.                                                     |
| `confidence_status`             | `STRING`     | Mức độ tin cậy của dòng mô tả.                                                  |
| `analyst_note`                  | `STRING`     | Ghi chú của người phân tích dữ liệu.                                            |
| `created_at`                    | `TIMESTAMP`  | Thời điểm dòng dữ liệu được tạo.                                                |
| `updated_at`                    | `TIMESTAMP`  | Thời điểm dòng dữ liệu được cập nhật gần nhất.                                  |

---

## 9. Trường khóa và trường định danh quan trọng

Hiện tại bảng không khai báo khóa chính trong DDL.

Tuy nhiên, về mặt logic, các trường sau rất quan trọng:

```text id="bsrdkh"
sequence_id
level_name
sequence_index
level_group
level_type
```

Trong đó:

* `sequence_id` nên là mã định danh duy nhất cho mỗi dòng;
* `level_name` dùng để đối chiếu với dữ liệu tracking;
* `sequence_index` dùng để xác định thứ tự level trong chuỗi;
* `level_group` và `level_type` dùng để phân loại level.

Nếu chỉ xét các level đang hoạt động trong tiến trình chính, tổ hợp sau nên được kiểm tra để tránh trùng:

```text id="4e2k2c"
level_group
+ sequence_index
```

hoặc:

```text id="v8119f"
level_name
+ effective_from_app_version
```

Tùy cách team muốn quản lý nhiều phiên bản chuỗi level.

---

## 10. Logic nghiệp vụ chính

### 10.1 Phân loại level

Bảng này dùng nhiều cột để phân loại level:

```text id="n616v6"
level_group
level_type
is_main_progression
is_required_in_sequence
is_tutorial_related
is_special_level
```

Những cột này giúp tách riêng:

* level thuộc tiến trình chính;
* level hướng dẫn;
* level phụ;
* level đặc biệt;
* level không còn hoạt động.

### 10.2 Xác định thứ tự level

Thứ tự level được mô tả qua:

```text id="zfdjwo"
sequence_index
previous_level_name
next_level_name
```

Trong đó:

* `sequence_index` là thứ tự số;
* `previous_level_name` là level trước đó;
* `next_level_name` là level tiếp theo.

### 10.3 Suy luận level trước và sau

Hai cột sau dùng để lưu kết quả suy luận:

```text id="36m9yp"
inferred_previous_level_name
inferred_next_level_name
```

Các cột này hữu ích khi dữ liệu thiết kế chưa hoàn chỉnh hoặc cần đối chiếu với dữ liệu thực tế từ người chơi.

Ví dụ, nếu `previous_level_name` bị thiếu nhưng dữ liệu thực tế cho thấy người chơi thường đi từ `N003` sang `N004`, hệ thống có thể suy luận `N003` là level trước của `N004`.

### 10.4 Theo dõi phạm vi áp dụng theo phiên bản game

Hai cột sau cho biết dòng mô tả level áp dụng trong phạm vi phiên bản nào:

```text id="2wefi2"
effective_from_app_version
effective_to_app_version
```

Điều này quan trọng khi thứ tự level thay đổi giữa các phiên bản game.

### 10.5 Theo dõi dữ liệu quan sát thực tế

Các cột quan sát gồm:

```text id="oi76jn"
first_seen_event_date
last_seen_event_date
observed_level_event_count
observed_start_level_count
observed_end_level_count
observed_tutorial_event_count
```

Những cột này giúp kiểm tra xem level có thật sự xuất hiện trong tracking hay không.

---

## 11. Kiểm tra chất lượng dữ liệu khuyến nghị

### 11.1 Kiểm tra trùng `sequence_id`

```sql id="qmxwby"
SELECT
  sequence_id,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

GROUP BY
  sequence_id

HAVING
  COUNT(*) > 1

ORDER BY
  row_count DESC,
  sequence_id;
```

Kỳ vọng:

```text id="jfrmyh"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là cùng một `sequence_id` đang xuất hiện nhiều lần.

---

### 11.2 Kiểm tra thiếu tên level

```sql id="8b5o7c"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_active IS TRUE
  AND level_name IS NULL

ORDER BY
  sequence_index;
```

Kỳ vọng:

```text id="szjskn"
Không trả ra dòng nào
```

Nếu `level_name` bị thiếu, bảng sẽ khó đối chiếu với dữ liệu sự kiện thực tế.

---

### 11.3 Kiểm tra trùng vị trí trong chuỗi chính

```sql id="2z9m6j"
SELECT
  level_group,
  sequence_index,
  COUNT(*) AS row_count,
  STRING_AGG(level_name, ' | ' ORDER BY level_name) AS level_names

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_active IS TRUE
  AND is_main_progression IS TRUE

GROUP BY
  level_group,
  sequence_index

HAVING
  COUNT(*) > 1

ORDER BY
  level_group,
  sequence_index;
```

Kỳ vọng:

```text id="eo3y90"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là cùng một vị trí trong chuỗi chính đang có nhiều level.

---

### 11.4 Kiểm tra level đang hoạt động nhưng không có dữ liệu quan sát

```sql id="399xu9"
SELECT
  level_name,
  level_group,
  level_type,
  sequence_index,
  observed_level_event_count,
  observed_start_level_count,
  observed_end_level_count,
  observed_tutorial_event_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_active IS TRUE
  AND COALESCE(observed_level_event_count, 0) = 0

ORDER BY
  sequence_index,
  level_name;
```

Kỳ vọng tùy trường hợp:

* nếu level đã ra trong bản test, nên có dữ liệu quan sát;
* nếu level chưa được người chơi chạm tới, có thể chấp nhận chưa có dữ liệu;
* nếu level đáng ra đã xuất hiện nhưng không có sự kiện, cần kiểm tra tracking hoặc mapping.

---

### 11.5 Kiểm tra quan hệ level trước và level sau

```sql id="h5p7hc"
SELECT
  current_level.level_name,
  current_level.previous_level_name,
  previous_level.level_name AS matched_previous_level_name,
  current_level.next_level_name,
  next_level.level_name AS matched_next_level_name

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq` AS current_level

LEFT JOIN
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq` AS previous_level
ON
  current_level.previous_level_name = previous_level.level_name

LEFT JOIN
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq` AS next_level
ON
  current_level.next_level_name = next_level.level_name

WHERE
  current_level.is_active IS TRUE
  AND (
    (
      current_level.previous_level_name IS NOT NULL
      AND previous_level.level_name IS NULL
    )
    OR
    (
      current_level.next_level_name IS NOT NULL
      AND next_level.level_name IS NULL
    )
  )

ORDER BY
  current_level.sequence_index,
  current_level.level_name;
```

Kỳ vọng:

```text id="rxalkc"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là bảng đang tham chiếu tới level trước hoặc level sau không tồn tại trong bảng.

---

## 12. Cách sử dụng khuyến nghị

### 12.1 Lấy chuỗi level chính đang hoạt động

```sql id="5zkgq9"
SELECT
  sequence_index,
  level_name,
  level_group,
  level_type,
  previous_level_name,
  next_level_name,
  is_required_in_sequence,
  is_tutorial_related,
  is_special_level,
  confidence_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_active IS TRUE
  AND is_main_progression IS TRUE

ORDER BY
  sequence_index;
```

### 12.2 Lấy các level có liên quan đến hướng dẫn

```sql id="heufsj"
SELECT
  sequence_index,
  level_name,
  level_group,
  level_type,
  observed_tutorial_event_count,
  first_seen_event_date,
  last_seen_event_date,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_tutorial_related IS TRUE

ORDER BY
  sequence_index,
  level_name;
```

### 12.3 Lấy các level đặc biệt

```sql id="pouvim"
SELECT
  sequence_index,
  level_name,
  level_group,
  level_type,
  first_seen_event_date,
  last_seen_event_date,
  observed_level_event_count,
  analyst_note

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_special_level IS TRUE

ORDER BY
  sequence_index,
  level_name;
```

### 12.4 So sánh số lần bắt đầu và kết thúc level

```sql id="cq9ic1"
SELECT
  level_name,
  sequence_index,
  observed_start_level_count,
  observed_end_level_count,

  observed_start_level_count - observed_end_level_count
    AS start_end_event_gap

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`

WHERE
  is_active IS TRUE

ORDER BY
  start_end_event_gap DESC,
  sequence_index;
```

Truy vấn này giúp tìm level có chênh lệch lớn giữa số lần bắt đầu và số lần kết thúc.

---

## 13. Lưu ý khi cập nhật bảng

Khi cập nhật `dim_level_seq`, cần chú ý:

1. Không để trùng `sequence_id`.
2. Không để trùng `sequence_index` trong cùng một chuỗi chính đang hoạt động.
3. Không để `level_name` trống với các level đang hoạt động.
4. Khi thay đổi `previous_level_name` hoặc `next_level_name`, cần đảm bảo level được tham chiếu có tồn tại.
5. Nếu cập nhật theo phiên bản game, cần điền rõ:

   * `effective_from_app_version`;
   * `effective_to_app_version`.
6. Nếu level được suy luận từ dữ liệu thay vì lấy từ thiết kế chính thức, cần ghi rõ trong:

   * `sequence_source`;
   * `confidence_status`;
   * `analyst_note`.
7. Nếu level chưa có dữ liệu quan sát, cần phân biệt rõ:

   * level chưa được người chơi chạm tới;
   * level chưa được release;
   * tracking bị thiếu;
   * mapping sai.

---

## 14. Rủi ro nếu bảng sai dữ liệu

| Lỗi trong bảng                                   | Ảnh hưởng                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| Sai `sequence_index`                             | Phân tích tiến trình level có thể sai thứ tự.                            |
| Sai `previous_level_name` hoặc `next_level_name` | Phân tích rơi từ level này sang level khác có thể sai.                   |
| Thiếu `level_name`                               | Không thể đối chiếu với dữ liệu sự kiện.                                 |
| Sai `level_type`                                 | Có thể phân loại nhầm level thường, level hướng dẫn hoặc level đặc biệt. |
| Sai `is_main_progression`                        | Có thể đưa nhầm level phụ vào phân tích tiến trình chính.                |
| Sai `is_tutorial_related`                        | Có thể không tách đúng các level bị ảnh hưởng bởi hướng dẫn.             |
| Sai phạm vi phiên bản game                       | Có thể áp dụng thứ tự level của phiên bản này sang phiên bản khác.       |
| Sai số lượng sự kiện quan sát                    | Có thể đánh giá sai mức độ xuất hiện của level trong dữ liệu thực tế.    |

---

## 15. Mức độ quan trọng

```text id="h3qqsx"
Mức độ quan trọng: Trung bình đến cao
```

Lý do:

* hiện tại bảng chưa được các view phân tích chính sử dụng trực tiếp;
* nhưng bảng này có vai trò nền tảng nếu mở rộng phân tích toàn bộ chuỗi level;
* bảng này giúp đối chiếu giữa thiết kế level và dữ liệu tracking thực tế;
* bảng này có thể trở thành nguồn chuẩn cho các phân tích tiến trình dài hạn.

Nếu team mở rộng phân tích từ 10 level đầu sang toàn bộ game, mức độ quan trọng của bảng này sẽ tăng lên cao.

---

## 16. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc bảng trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="ur3bro"
sql/ddl/core_foundation/dim_level_seq.sql
```

Cấu trúc bảng hiện tại:

```sql id="x0ylbn"
CREATE TABLE `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_level_seq`
(
  sequence_id STRING,
  level_group STRING,
  level_type STRING,
  level_name STRING,
  level_prefix STRING,
  level_number INT64,
  sequence_index INT64,
  previous_level_name STRING,
  next_level_name STRING,
  inferred_previous_level_name STRING,
  inferred_next_level_name STRING,
  is_active BOOL,
  is_main_progression BOOL,
  is_required_in_sequence BOOL,
  is_tutorial_related BOOL,
  is_special_level BOOL,
  effective_from_app_version STRING,
  effective_to_app_version STRING,
  first_seen_event_date STRING,
  last_seen_event_date STRING,
  observed_level_event_count INT64,
  observed_start_level_count INT64,
  observed_end_level_count INT64,
  observed_tutorial_event_count INT64,
  sequence_source STRING,
  confidence_status STRING,
  analyst_note STRING,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

---

## Liên kết liên quan

- Framework: [[framework_pipeline]]

- Upstream: Không có

- Downstream: Chưa có
