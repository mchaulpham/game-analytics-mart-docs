# dim_f10_level_map

## 1. Mục đích

`dim_f10_level_map` là bảng ánh xạ 10 level đầu tiên trong tiến trình chơi game theo từng phiên bản game.

Bảng này giúp hệ thống biết, với mỗi phiên bản game:

* level nào là bước 1, bước 2, bước 3, v.v.;
* level đó có tên kỹ thuật là gì;
* level đó thuộc loại level nào;
* level đó có đang được sử dụng hay không;
* level đó có bắt buộc trong tiến trình 10 level đầu hay không;
* ánh xạ này đến từ nguồn nào;
* độ tin cậy của ánh xạ là gì.

Nói ngắn gọn, đây là bảng chuẩn để các view phân tích hiểu đúng thứ tự 10 level đầu.

---

## 2. Vai trò trong hệ thống

`dim_f10_level_map` thuộc nhóm:

```text id="8t4s18"
core_foundation
```

Tức là nhóm nền tảng lõi.

Bảng này không phải là bảng kết quả phân tích cuối cùng. Nó là bảng tham chiếu dùng để các view khác tính toán chính xác phễu người chơi mới, phễu 10 level đầu, thời gian đi qua từng level và chẩn đoán thiết kế level.

Trong tài liệu này, “bảng tham chiếu” nghĩa là bảng dùng để mô tả, phân loại hoặc ánh xạ dữ liệu, thay vì trực tiếp chứa kết quả phân tích hành vi người chơi.

---

## 3. Câu hỏi phân tích có thể trả lời

Bảng này giúp trả lời các câu hỏi nền tảng sau:

1. Với một phiên bản game cụ thể, 10 level đầu tiên gồm những level nào?
2. Level nào là bước 1, bước 2, bước 3 trong tiến trình 10 level đầu?
3. Một level có thuộc tiến trình bắt buộc hay không?
4. Một ánh xạ level có đang được dùng trong phân tích hiện tại hay không?
5. Ánh xạ này có độ tin cậy cao hay cần kiểm tra lại?
6. Level trước và level sau của một bước tiến trình là gì?
7. Khi phiên bản game thay đổi, thứ tự hoặc thiết kế level đầu game có thay đổi hay không?

---

## 4. Độ chi tiết của mỗi dòng dữ liệu

Mỗi dòng trong `dim_f10_level_map` đại diện cho một ánh xạ giữa:

```text id="h1zo73"
một phiên bản game
+ một bước trong tiến trình 10 level đầu
+ một level cụ thể
```

Có thể hiểu đơn giản:

```text id="50gwx4"
Một dòng = một level được đặt vào một vị trí trong 10 bước đầu của một phiên bản game
```

Ví dụ về logic:

```text id="r3vj2c"
app_version = 0.3.61
progression_step = 1
level_name = N001
```

Nghĩa là trong phiên bản `0.3.61`, level `N001` được xem là bước 1 trong tiến trình 10 level đầu.

---

## 5. Loại đối tượng

```text id="4ygr5d"
Loại đối tượng: BASE TABLE
```

Trong BigQuery, `BASE TABLE` nghĩa là bảng vật lý có lưu dữ liệu thật.

Khác với view, bảng này không tự tính lại kết quả từ câu truy vấn mỗi lần được gọi. Dữ liệu trong bảng cần được tạo hoặc cập nhật bởi quy trình bên ngoài, ví dụ do người phân tích dữ liệu cập nhật, hoặc do một câu lệnh tạo bảng đã được chạy trước đó.

---

## 6. Nguồn dữ liệu phụ thuộc

`dim_f10_level_map` là bảng vật lý. Bản thân bảng này không phụ thuộc trực tiếp vào view nào trong kiến trúc hiện tại.

Tuy nhiên, về mặt nghiệp vụ, bảng này là bảng do team phân tích hoặc team thiết kế dữ liệu duy trì để phản ánh thứ tự 10 level đầu theo từng phiên bản game.

Các trường như sau cho thấy bảng có yếu tố quản trị dữ liệu thủ công hoặc bán thủ công:

* `mapping_source`;
* `confidence_status`;
* `analyst_note`;
* `created_at`;
* `updated_at`.

---

## 7. Các view đang sử dụng bảng này

`dim_f10_level_map` đang được các view sau sử dụng:

| View sử dụng               | Mục đích sử dụng                                                              |
| -------------------------- | ----------------------------------------------------------------------------- |
| `vw_onboard_funnel_24h`    | Xác định Level 1 theo từng phiên bản game để đo phễu người chơi mới.          |
| `vw_onboard_drop_diag_24h` | Xác định Level 1 để phân loại rơi trong giai đoạn người chơi mới.             |
| `vw_f10_funnel_24h`        | Xác định 10 level đầu để đo phễu tiến trình 24 giờ.                           |
| `vw_f10_timing_24h`        | Xác định thứ tự 10 level đầu để đo thời gian đi qua từng bước.                |
| `vw_f10_design_diag_24h`   | Dùng thứ tự level, tên level và thông tin ánh xạ để chẩn đoán thiết kế level. |

Do bảng này được nhiều view quan trọng sử dụng, nếu ánh xạ sai thì nhiều báo cáo phía sau sẽ sai theo.

---

## 8. Danh sách cột

| Cột                         | Kiểu dữ liệu | Ý nghĩa                                                                                                 |
| --------------------------- | ------------ | ------------------------------------------------------------------------------------------------------- |
| `app_version`               | `STRING`     | Phiên bản game mà ánh xạ level áp dụng.                                                                 |
| `progression_step`          | `INT64`      | Số thứ tự bước trong tiến trình 10 level đầu. Ví dụ: 1, 2, 3.                                           |
| `progression_slot_id`       | `STRING`     | Mã định danh cho vị trí trong tiến trình. Dùng để nhận diện slot của level trong luồng 10 level đầu.    |
| `level_name`                | `STRING`     | Tên level trong tracking, ví dụ `N001`, `N002`.                                                         |
| `level_design_id`           | `STRING`     | Mã thiết kế level. Có thể dùng để liên kết với tài liệu hoặc dữ liệu thiết kế level nội bộ.             |
| `level_type`                | `STRING`     | Loại level. Ví dụ: level thường, level hướng dẫn, level đặc biệt.                                       |
| `previous_progression_step` | `INT64`      | Bước trước đó trong tiến trình.                                                                         |
| `next_progression_step`     | `INT64`      | Bước tiếp theo trong tiến trình.                                                                        |
| `is_active`                 | `BOOL`       | Cho biết ánh xạ này có đang được sử dụng hay không.                                                     |
| `is_required`               | `BOOL`       | Cho biết level này có bắt buộc trong tiến trình phân tích hay không.                                    |
| `mapping_source`            | `STRING`     | Nguồn tạo ra ánh xạ. Ví dụ: do người phân tích nhập, lấy từ dữ liệu thiết kế, hoặc suy luận từ dữ liệu. |
| `confidence_status`         | `STRING`     | Mức độ tin cậy của ánh xạ.                                                                              |
| `analyst_note`              | `STRING`     | Ghi chú của người phân tích dữ liệu.                                                                    |
| `created_at`                | `TIMESTAMP`  | Thời điểm dòng dữ liệu được tạo.                                                                        |
| `updated_at`                | `TIMESTAMP`  | Thời điểm dòng dữ liệu được cập nhật gần nhất.                                                          |

---

## 9. Trường khóa và trường định danh quan trọng

Hiện tại bảng không khai báo khóa chính trong DDL.

Tuy nhiên, về mặt logic phân tích, tổ hợp sau nên được xem là khóa tự nhiên:

```text id="edhsvk"
app_version
+ progression_step
+ level_name
```

Trong nhiều trường hợp, tổ hợp sau cũng nên là duy nhất với các dòng đang hoạt động và bắt buộc:

```text id="2bcp83"
app_version
+ progression_step
```

Lý do: trong một phiên bản game, mỗi bước từ 1 đến 10 chỉ nên ánh xạ tới một level bắt buộc đang hoạt động.

---

## 10. Quy tắc sử dụng trong các view phân tích

Các view phía sau thường chỉ sử dụng các dòng thỏa mãn:

```sql id="9s9ydj"
is_active IS TRUE
AND is_required IS TRUE
```

Với nhóm 10 level đầu, các view thường lọc thêm:

```sql id="4xi3o4"
progression_step BETWEEN 1 AND 10
```

Với phân tích người chơi mới, hệ thống dùng:

```sql id="mgl1ai"
progression_step = 1
```

để xác định Level 1 tương ứng với từng phiên bản game.

---

## 11. Logic nghiệp vụ chính

### 11.1 Xác định 10 level đầu

Bảng này định nghĩa thứ tự 10 level đầu bằng cột:

```text id="lu6irl"
progression_step
```

Ví dụ:

| `progression_step` | Ý nghĩa                                   |
| -----------------: | ----------------------------------------- |
|                  1 | Level đầu tiên trong tiến trình phân tích |
|                  2 | Level thứ hai                             |
|                  3 | Level thứ ba                              |
|                ... | ...                                       |
|                 10 | Level thứ mười                            |

### 11.2 Xác định level trước và level sau

Hai cột sau giúp mô tả quan hệ liền kề giữa các bước:

```text id="0xr8ky"
previous_progression_step
next_progression_step
```

Các cột này hỗ trợ phân tích rơi từ bước hiện tại sang bước tiếp theo.

### 11.3 Chỉ lấy ánh xạ đang hoạt động

Không phải mọi dòng trong bảng đều nhất thiết được dùng cho phân tích hiện tại.

Các dòng có:

```text id="skkz2k"
is_active = TRUE
```

mới được xem là ánh xạ đang dùng.

### 11.4 Chỉ lấy level bắt buộc

Một số level có thể không bắt buộc trong tiến trình phân tích.

Các view chính thường chỉ lấy:

```text id="r5ue1v"
is_required = TRUE
```

để đảm bảo phễu chuyển đổi chỉ tính các level bắt buộc.

---

## 12. Kiểm tra chất lượng dữ liệu khuyến nghị

Vì bảng này là bảng ánh xạ quan trọng, nên cần kiểm tra định kỳ.

### 12.1 Kiểm tra mỗi phiên bản có đủ 10 level bắt buộc đang hoạt động hay không

```sql id="0y9398"
SELECT
  app_version,

  COUNT(*) AS active_required_level_count,

  MIN(progression_step) AS min_progression_step,

  MAX(progression_step) AS max_progression_step

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE

GROUP BY
  app_version

ORDER BY
  app_version;
```

Kỳ vọng thông thường:

```text id="l2lrk3"
active_required_level_count = 10
min_progression_step = 1
max_progression_step = 10
```

Nếu một phiên bản không đủ 10 dòng, các view phân tích 10 level đầu có thể bị thiếu bước.

---

### 12.2 Kiểm tra trùng bước trong cùng phiên bản

```sql id="vtfhec"
SELECT
  app_version,
  progression_step,
  COUNT(*) AS row_count

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE

GROUP BY
  app_version,
  progression_step

HAVING
  COUNT(*) > 1

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="50h8zs"
Không trả ra dòng nào
```

Nếu có kết quả, nghĩa là một phiên bản đang có nhiều hơn một level cho cùng một bước tiến trình.

---

### 12.3 Kiểm tra thiếu tên level

```sql id="6j3n09"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE
  AND level_name IS NULL

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="d2a2qn"
Không trả ra dòng nào
```

Nếu `level_name` bị thiếu, các view phía sau không thể nối đúng với dữ liệu level.

---

### 12.4 Kiểm tra thứ tự bước ngoài phạm vi 1 đến 10

```sql id="n3oqod"
SELECT
  *

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE
  AND (
    progression_step < 1
    OR progression_step > 10
  )

ORDER BY
  app_version,
  progression_step;
```

Kỳ vọng:

```text id="e64wil"
Không trả ra dòng nào
```

---

## 13. Cách sử dụng khuyến nghị

### 13.1 Lấy mapping 10 level đầu đang hoạt động

```sql id="o2ywl1"
SELECT
  app_version,
  progression_step,
  progression_slot_id,
  level_name,
  level_design_id,
  level_type,
  previous_progression_step,
  next_progression_step,
  mapping_source,
  confidence_status

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE
  AND progression_step BETWEEN 1 AND 10

ORDER BY
  app_version,
  progression_step;
```

### 13.2 Lấy Level 1 theo từng phiên bản game

```sql id="jg6kz4"
SELECT
  app_version,
  level_name AS level1_name

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`

WHERE
  is_active IS TRUE
  AND is_required IS TRUE
  AND progression_step = 1

ORDER BY
  app_version;
```

### 13.3 Lấy level kế tiếp của từng bước

```sql id="ccur41"
SELECT
  current_level.app_version,
  current_level.progression_step,
  current_level.level_name,
  current_level.next_progression_step,
  next_level.level_name AS next_level_name

FROM
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map` AS current_level

LEFT JOIN
  `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map` AS next_level
ON
  current_level.app_version = next_level.app_version
  AND current_level.next_progression_step = next_level.progression_step
  AND next_level.is_active IS TRUE
  AND next_level.is_required IS TRUE

WHERE
  current_level.is_active IS TRUE
  AND current_level.is_required IS TRUE

ORDER BY
  current_level.app_version,
  current_level.progression_step;
```

---

## 14. Lưu ý khi cập nhật bảng

Khi cập nhật `dim_f10_level_map`, cần đặc biệt chú ý:

1. Không để trùng `progression_step` trong cùng `app_version` nếu cả hai dòng đều đang hoạt động và bắt buộc.
2. Không đổi `level_name` nếu chưa xác nhận với dữ liệu tracking thực tế.
3. Nếu thay đổi thứ tự level, cần kiểm tra lại các view:

   * `vw_f10_funnel_24h`;
   * `vw_f10_timing_24h`;
   * `vw_f10_funnel_timing_24h`;
   * `vw_f10_design_diag_24h`;
   * `vw_f10_design_board_24h`.
4. Nếu thêm phiên bản game mới, cần đảm bảo có đủ 10 dòng bắt buộc đang hoạt động.
5. Nếu chưa chắc chắn về ánh xạ, nên ghi rõ trong:

   * `confidence_status`;
   * `analyst_note`.

---

## 15. Rủi ro nếu bảng sai dữ liệu

Nếu `dim_f10_level_map` bị sai, các vấn đề sau có thể xảy ra:

| Lỗi trong bảng                     | Ảnh hưởng                                                      |
| ---------------------------------- | -------------------------------------------------------------- |
| Sai Level 1                        | Phễu người chơi mới sẽ tính sai điểm bắt đầu level đầu tiên.   |
| Thiếu level trong 10 bước đầu      | Phễu 10 level đầu sẽ thiếu bước.                               |
| Trùng `progression_step`           | Kết quả phễu có thể bị nhân dòng hoặc sai số lượng người chơi. |
| Sai `level_name`                   | Không nối được với sự kiện level thực tế.                      |
| Sai `is_active` hoặc `is_required` | View phía sau có thể lấy nhầm hoặc bỏ sót level.               |
| Sai `next_progression_step`        | Phân tích rơi sang bước tiếp theo có thể sai.                  |

---

## 16. Mức độ quan trọng

```text id="3o7jfa"
Mức độ quan trọng: Cao
```

Lý do:

* bảng này được nhiều view phân tích chính sử dụng;
* bảng này quyết định cách hệ thống hiểu 10 level đầu;
* sai sót trong bảng này có thể lan sang nhiều báo cáo;
* đây là bảng cần được kiểm tra mỗi khi có phiên bản game mới hoặc thay đổi thứ tự level.

---

## 17. DDL tham chiếu

DDL là câu lệnh định nghĩa cấu trúc bảng trong BigQuery.

Đường dẫn đề xuất để lưu DDL:

```text id="30dyka"
sql/ddl/core_foundation/dim_f10_level_map.sql
```

Cấu trúc bảng hiện tại:

```sql id="8t7j7x"
CREATE TABLE `project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart.dim_f10_level_map`
(
  app_version STRING,
  progression_step INT64,
  progression_slot_id STRING,
  level_name STRING,
  level_design_id STRING,
  level_type STRING,
  previous_progression_step INT64,
  next_progression_step INT64,
  is_active BOOL,
  is_required BOOL,
  mapping_source STRING,
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

- Downstream:
	- [[vw_onboard_funnel_24h]]
	- [[vw_onboard_drop_diag_24h]]
	- [[vw_f10_funnel_24h]]
	- [[vw_f10_timing_24h]]
	- [[vw_f10_design_diag_24h]]