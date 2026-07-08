# Khung tổng thể luồng dữ liệu phân tích game

## 1. Mục đích tài liệu

Tài liệu này mô tả kiến trúc tổng thể của bộ dữ liệu phân tích game trong BigQuery.

Mục tiêu là giúp các thành viên trong team có thể hiểu:

* dữ liệu được lấy từ đâu;
* dữ liệu được làm sạch và chuẩn hóa như thế nào;
* mỗi nhóm bảng/view phục vụ mục đích phân tích nào;
* các bảng và view phụ thuộc lẫn nhau ra sao;
* nên đọc và sử dụng các bảng theo thứ tự nào;
* đâu là view phù hợp cho từng nhóm câu hỏi phân tích như onboarding, first 10 levels, level difficulty, progression, retention và pipeline health.

Tài liệu này là tài liệu gốc để đọc trước khi đi vào từng bản đặc tả chi tiết của 22 bảng và view trong hệ thống.

---

## 2. Một số thuật ngữ cần thống nhất

Trong tài liệu này, phần lớn nội dung sẽ được viết bằng tiếng Việt. Một số thuật ngữ tiếng Anh vẫn được giữ lại vì đây là thuật ngữ chuẩn trong BigQuery, GA4 hoặc phân tích dữ liệu. Khi xuất hiện lần đầu, các thuật ngữ này được giải thích rõ như sau.

| Thuật ngữ  | Cách hiểu trong tài liệu này                                                                                                                                          |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DDL        | Viết tắt của “Data Definition Language”. Trong ngữ cảnh này, DDL là câu lệnh định nghĩa cấu trúc bảng hoặc view trong BigQuery. Ví dụ: `CREATE TABLE`, `CREATE VIEW`. |
| View       | Bảng ảo trong BigQuery. View không lưu dữ liệu vật lý, mà lưu một câu truy vấn. Mỗi lần gọi view, BigQuery sẽ chạy lại câu truy vấn đó để trả kết quả.                |
| Dataset    | Vùng chứa dữ liệu trong BigQuery. Một dataset có thể chứa nhiều bảng và view.                                                                                         |
| Event      | Sự kiện được gửi từ game lên GA4. Ví dụ: `first_open`, `Start_level`, `End_level`, `Tutorial_start`.                                                                  |
| Funnel     | Phễu chuyển đổi. Đây là chuỗi bước mà người chơi cần đi qua. Ví dụ: `first_open` → `Open_first` → bắt đầu Level 1.                                                    |
| Cohort     | Nhóm người dùng theo cùng điều kiện bắt đầu phân tích. Ví dụ: nhóm người dùng có `first_open` cùng ngày hoặc cùng app version.                                        |
| Retention  | Tỷ lệ người dùng quay lại sau một khoảng thời gian. Ví dụ: D1 Retention là tỷ lệ user quay lại vào ngày kế tiếp sau ngày `first_open`.                                |
| Activation | Tỷ lệ người dùng thực hiện một hành động quan trọng sau khi đã vào app. Trong tài liệu này có trường hợp đo user quay lại D1 và có bắt đầu chơi level.                |
| Grain      | Độ chi tiết của mỗi dòng dữ liệu. Ví dụ: một dòng là một lần chơi level, một người chơi theo một level, một ngày theo một level, hoặc một app version theo funnel step. |
| Dependency | Quan hệ phụ thuộc giữa các bảng hoặc view. Nếu view A đọc dữ liệu từ view B, thì A phụ thuộc vào B.                                                                   |
| Pipeline   | Luồng xử lý dữ liệu từ dữ liệu thô đến các bảng phân tích cuối cùng.                                                                                                  |
| Upstream   | Nguồn dữ liệu đầu vào của một bảng/view. Ví dụ: nếu view A đọc từ view B, thì view B là upstream của view A.                                                          |
| Downstream | Bảng/view nằm phía sau trong luồng xử lý và phụ thuộc vào dữ liệu từ bảng/view hiện tại.                                                                              |

---

## 3. Phạm vi hệ thống

Nguồn dữ liệu thô chính là dữ liệu GA4 được xuất sang BigQuery:

```text
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Bộ dữ liệu phân tích được xây dựng trong dataset:

```text
project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart
```

Hệ thống hiện có 22 đối tượng dữ liệu:

| Nhóm                        | Số lượng | Loại đối tượng         | Vai trò chính |
| --------------------------- | -------: | ---------------------- | ------------- |
| Nền tảng lõi                |        8 | 2 bảng vật lý, 6 view  | Làm sạch, chuẩn hóa, mapping và tạo dữ liệu nền cho phân tích level. |
| Phân tích hiện tại          |        8 | 8 view                 | Phân tích onboarding, first 10 levels, funnel tổng hợp, timing và chẩn đoán thiết kế. |
| Giám sát và phân tích level |        5 | 5 view                 | Theo dõi reach, difficulty, progression và pipeline health. |
| Phân tích retention         |        1 | 1 view                 | Phân tích D1 Retention theo cohort và app version. |
| Tổng cộng                   |       22 | 2 bảng vật lý, 20 view | Bộ data mart phục vụ phân tích gameplay prototype. |

Trong đó:

* bảng vật lý là bảng có lưu dữ liệu thật;
* view là bảng ảo được tạo từ câu truy vấn;
* toàn bộ kết quả phân tích hiện tại đều là view, chưa có bảng kết quả được lên lịch ghi dữ liệu vật lý;
* một số view phân tích đọc trực tiếp từ GA4 raw thay vì đi qua toàn bộ tầng foundation. Đây là thiết kế có chủ đích để phục vụ những logic cohort/funnel riêng.

---

## 4. Quy ước đặt tên

Hệ thống hiện dùng quy ước đặt tên như sau:

| Mẫu tên      | Ý nghĩa                                                                                |
| ------------ | -------------------------------------------------------------------------------------- |
| `dim_`       | Bảng chiều hoặc bảng ánh xạ. Đây là bảng dùng để mô tả, phân loại hoặc ánh xạ dữ liệu. |
| `vw_`        | View trong BigQuery. Đây là bảng ảo, không lưu dữ liệu vật lý.                         |
| `f10`        | Viết gọn của “first 10”, nghĩa là 10 level đầu tiên trong tiến trình chơi.             |
| `onboard`    | Giai đoạn người chơi mới bắt đầu vào game và đi qua các bước đầu tiên.                 |
| `retention`  | Nhóm phân tích người chơi quay lại sau ngày cài/mở app đầu tiên.                       |
| `diag`       | Viết gọn của “diagnostics”, nghĩa là chẩn đoán hoặc phát hiện vấn đề.                  |
| `seq`        | Viết gọn của “sequence”, nghĩa là chuỗi hoặc thứ tự level.                             |
| `_24h`       | Logic phân tích trong cửa sổ 24 giờ.                                                   |
| `_daily`     | Kết quả tổng hợp theo ngày.                                                            |
| `_summary`   | Kết quả tổng hợp nhiều ngày hoặc tổng hợp chung.                                       |

Các tên bảng, tên view, tên cột và tên sự kiện được giữ nguyên theo hệ thống để tránh sai lệch khi đối chiếu với BigQuery.

Lưu ý chuẩn chính tả:

* dùng nhất quán `retention` cho toàn bộ tài liệu, dashboard và query;
* nếu phát hiện tên file hoặc text cũ còn sai chính tả, cần sửa về `retention` để đồng bộ tài liệu, dashboard và query.

---

## 5. Kiến trúc tổng thể

Luồng dữ liệu có năm nhóm chính:

```text
Dữ liệu thô từ GA4
  ↓
Tầng nền tảng lõi
  ↓
Tầng phân tích hiện tại
  ↓
Tầng giám sát và phân tích level

Dữ liệu thô từ GA4
  ↓
Tầng phân tích retention
```

Cách hiểu từng tầng:

1. **Dữ liệu thô từ GA4**

   Đây là dữ liệu ban đầu do game gửi lên GA4, sau đó được xuất sang BigQuery.

2. **Tầng nền tảng lõi**

   Tầng này làm sạch dữ liệu, chuẩn hóa sự kiện, chuẩn hóa level, ghép sự kiện bắt đầu và kết thúc level, đồng thời đánh dấu các lần chơi có liên quan đến hướng dẫn.

3. **Tầng phân tích hiện tại**

   Tầng này phục vụ các phân tích đang cần dùng trực tiếp cho dự án: phễu người chơi mới, 10 level đầu, thời gian đi qua level, chẩn đoán thiết kế level và funnel tổng hợp từ `first_open` đến Level 10.

4. **Tầng giám sát và phân tích level**

   Tầng này phục vụ việc theo dõi chất lượng dữ liệu, độ khó level, tiến trình người chơi và sức khỏe toàn bộ luồng dữ liệu.

5. **Tầng phân tích retention**

   Tầng này phục vụ phân tích user quay lại sau ngày đầu tiên, đặc biệt là D1 App Retention, D1 Gameplay Retention và Gameplay Activation among Returners.

Lưu ý quan trọng:

* không phải mọi view phân tích đều bắt buộc đọc qua toàn bộ tầng foundation;
* `vw_funnel_first_open_to_level10` đọc trực tiếp GA4 raw và dùng thêm `dim_f10_level_map` để tạo funnel tổng hợp;
* `vw_d1_retention` đọc trực tiếp GA4 raw để xác định cohort theo `first_open` và activity ngày D1.

---

## 6. Tầng dữ liệu thô từ GA4

Nguồn dữ liệu thô:

```text
analytics_524104373.events_*
```

Đây là các bảng sự kiện GA4 dạng chia theo ngày. Mỗi bảng có hậu tố ngày ở dạng `YYYYMMDD`.

Ví dụ:

```text
events_20260531
events_20260601
events_20260602
```

Các view lõi đọc trực tiếp từ nguồn này gồm:

| View                 | Mục đích                                                  |
| -------------------- | --------------------------------------------------------- |
| `vw_level_start_eff` | Lấy và chuẩn hóa các sự kiện bắt đầu level.               |
| `vw_onboard_events`  | Lấy các sự kiện phục vụ phân tích người chơi mới.         |
| `vw_level_events`    | Lấy và làm sạch các sự kiện `Start_level` và `End_level`. |
| `vw_tutorial_events` | Lấy các sự kiện liên quan đến hướng dẫn trong game.       |

Ngoài các view lõi, hiện có hai view phân tích cũng đọc trực tiếp từ GA4 raw:

| View                              | Mục đích                                                                 |
| --------------------------------- | ------------------------------------------------------------------------ |
| `vw_funnel_first_open_to_level10` | Tạo funnel tổng hợp từ `first_open` đến Level 10 trong 24 giờ đầu.       |
| `vw_d1_retention`                 | Tính D1 Retention theo `cohort_date`, `d1_date` và `app_version`.        |

Các view đọc trực tiếp từ GA4 raw cần được theo dõi kỹ khi tracking thay đổi, vì logic của chúng phụ thuộc trực tiếp vào event name, event params, `user_pseudo_id`, `event_timestamp` và `app_info.version`.

---

## 7. Tầng nền tảng lõi

Tầng nền tảng lõi gồm 8 đối tượng.

### 7.1 `dim_f10_level_map`

Đây là bảng ánh xạ 10 level đầu tiên theo từng phiên bản game.

Bảng này giúp hệ thống biết:

* level nào là bước 1, bước 2, bước 3, v.v.;
* level đó thuộc phiên bản game nào;
* level đó có đang được dùng hay không;
* level đó có bắt buộc trong tiến trình 10 level đầu hay không;
* độ tin cậy của ánh xạ này là gì.

Bảng này được dùng nhiều trong các phân tích:

* phễu người chơi mới;
* phễu 10 level đầu;
* funnel tổng hợp từ `first_open` đến Level 10;
* thời gian đi qua 10 level đầu;
* chẩn đoán thiết kế level.

### 7.2 `dim_level_seq`

Đây là bảng mô tả thứ tự level tổng quát hơn.

Bảng này chứa thông tin như:

* nhóm level;
* loại level;
* số thứ tự level;
* level trước;
* level sau;
* level có thuộc tiến trình chính hay không;
* level có liên quan đến hướng dẫn hay không;
* level có phải level đặc biệt hay không;
* số lần quan sát được trong dữ liệu.

Hiện tại bảng này chưa được các view khác sử dụng trực tiếp, nhưng có vai trò quan trọng nếu sau này cần mở rộng phân tích toàn bộ tiến trình level.

### 7.3 `vw_level_start_eff`

View này tạo ra sự kiện bắt đầu level dạng hiệu lực.

Lý do cần view này là vì không phải mọi trường hợp bắt đầu level đều đến từ sự kiện `Start_level` thuần túy. Một số phiên bản hoặc một số tình huống hướng dẫn có thể cần suy luận sự kiện bắt đầu level từ `Tutorial_start`.

View này tạo ra một mã định danh ổn định cho mỗi lần bắt đầu level:

```text
effective_start_event_id
```

View cũng phân biệt:

* lần bắt đầu level thật từ `Start_level`;
* lần bắt đầu level được suy luận từ `Tutorial_start`;
* quy tắc dùng để suy luận.

### 7.4 `vw_onboard_events`

View này lấy các sự kiện phục vụ phân tích giai đoạn người chơi mới.

Các sự kiện chính gồm:

* `first_open`: lần đầu người chơi mở game;
* `Open_first`: bước mở đầu trong luồng vào game;
* `Vuot_home`: sự kiện liên quan đến vào màn hình chính hoặc vượt qua đoạn đầu;
* `app_remove`: người chơi gỡ ứng dụng.

View này là nguồn chính cho các phân tích người chơi mới.

### 7.5 `vw_level_events`

View này làm sạch các sự kiện level chính:

```text
Start_level
End_level
```

View này trích xuất và chuẩn hóa các tham số quan trọng:

* `level_name`;
* `attempt_no`;
* `success`;
* `fail_reason`;
* `move_used`;
* `move_left`;
* `duration_sec`;
* `gold`;
* `lives_left`;
* `booster_used`;
* `booster_pre_used`.

View này giữ cả cột đã ép kiểu và cột gốc dạng thô. Việc giữ cột thô giúp kiểm tra chất lượng dữ liệu khi có lỗi ép kiểu hoặc tracking không nhất quán.

### 7.6 `vw_tutorial_events`

View này lấy các sự kiện hướng dẫn:

* `Tutorial_start`;
* `Tutorial_step_start`;
* `Tutorial_end`.

Các trường chính gồm:

* thời điểm sự kiện;
* người chơi;
* tên level;
* tên hướng dẫn;
* giá trị của bước hướng dẫn.

View này được dùng để xác định những lần chơi level bị ảnh hưởng bởi hướng dẫn.

### 7.7 `vw_level_tutorial_flag`

View này đánh dấu các lần `Start_level` xảy ra gần thời điểm bắt đầu hướng dẫn.

Nếu một lần bắt đầu level xảy ra gần `Tutorial_start`, view sẽ đánh dấu:

```text
is_near_tutorial_start = TRUE
```

Mục đích là tách riêng dữ liệu chơi level thông thường và dữ liệu có thể bị ảnh hưởng bởi hướng dẫn.

Điều này rất quan trọng khi phân tích độ khó level, vì một level có hướng dẫn có thể có hành vi khác với một level chơi bình thường.

### 7.8 `vw_level_attempts`

View này ghép sự kiện bắt đầu level với sự kiện kết thúc level.

Một lần bắt đầu level được ghép với một lần kết thúc level dựa trên:

* cùng người chơi;
* cùng level;
* cùng số lần thử;
* thời điểm kết thúc lớn hơn hoặc bằng thời điểm bắt đầu;
* thời điểm kết thúc không quá 6 giờ sau thời điểm bắt đầu.

Kết quả là một dòng dữ liệu đại diện cho một lần người chơi bắt đầu chơi level, có thể có hoặc không có sự kiện kết thúc tương ứng.

Đây là view lõi nhất để phân tích:

* tỷ lệ thắng;
* tỷ lệ thua;
* số lần chơi không có kết thúc;
* thời lượng chơi;
* số bước đi đã dùng;
* ảnh hưởng của hướng dẫn;
* độ khó level.

---

## 8. Tầng phân tích hiện tại

Tầng này gồm 8 view, phục vụ trực tiếp cho các câu hỏi phân tích đang cần trong giai đoạn hiện tại của dự án.

### 8.1 Nhóm phân tích người chơi mới

#### `vw_onboard_funnel_24h`

View này đo phễu người chơi mới trong vòng 24 giờ.

Các bước chính:

```text
first_open
  ↓
Open_first
  ↓
bắt đầu Level 1
```

View này trả về các chỉ số như:

* số người chơi mở game lần đầu;
* số người chơi đã đủ 24 giờ để quan sát;
* số người chơi đi tới `Open_first`;
* số người chơi bắt đầu Level 1;
* tỷ lệ chuyển đổi giữa các bước;
* số người bị rơi ở từng bước.

#### `vw_onboard_drop_diag_24h`

View này đi sâu hơn vào các nhóm rơi khỏi phễu người chơi mới.

View phân loại người chơi vào các nhóm như:

* có `first_open` nhưng không có `Open_first` và không bắt đầu Level 1;
* có `Open_first` nhưng không bắt đầu Level 1;
* bắt đầu Level 1 nhưng thiếu `Open_first`;
* đi đủ từ `Open_first` đến Level 1.

View cũng đo thêm các tín hiệu như:

* có `Vuot_home` trong 24 giờ hay không;
* có `app_remove` trong 24 giờ hay không;
* thời gian từ `first_open` đến các mốc quan trọng.

### 8.2 Nhóm phân tích 10 level đầu

#### `vw_f10_funnel_24h`

View này đo số người chơi đi qua 10 level đầu trong vòng 24 giờ kể từ lần bắt đầu Level 1.

View này dựa vào bảng ánh xạ:

```text
dim_f10_level_map
```

Các chỉ số chính:

* số người trong nhóm bắt đầu Level 1;
* số người đã đủ 24 giờ để quan sát;
* số người đi tới từng level trong 10 level đầu;
* tỷ lệ chuyển đổi từ level trước sang level sau;
* tỷ lệ rơi từ level trước sang level sau;
* tỷ lệ chuyển đổi cộng dồn từ Level 1.

#### `vw_f10_timing_24h`

View này đo thời gian người chơi đi qua các bước trong 10 level đầu.

Các nhóm thời gian chính:

* từ Level 1 đến từng level;
* từ level trước đến level hiện tại;
* thời gian ở trong level dựa trên sự kiện kết thúc;
* thời lượng báo cáo từ tham số `duration_sec`.

View này giúp phát hiện:

* level nào làm người chơi mất nhiều thời gian;
* đoạn nào trong 10 level đầu có độ trễ bất thường;
* dữ liệu thời lượng có đủ độ phủ hay không.

#### `vw_f10_funnel_timing_24h`

View này kết hợp kết quả từ:

```text
vw_f10_funnel_24h
vw_f10_timing_24h
```

Mục đích là tạo một view tổng hợp vừa có số lượng người chơi theo từng bước, vừa có các chỉ số thời gian tương ứng.

### 8.3 Nhóm chẩn đoán thiết kế level

#### `vw_f10_design_diag_24h`

View này là view chẩn đoán chính cho thiết kế 10 level đầu.

View kết hợp nhiều nhóm chỉ số:

* số người tới từng level;
* tỷ lệ rơi sang level kế tiếp;
* tỷ lệ rơi sau khi thắng level;
* số lần thử trung bình;
* tỷ lệ thắng;
* tỷ lệ thua;
* tỷ lệ không có `End_level`;
* mức độ ảnh hưởng của hướng dẫn;
* thời gian chơi;
* số bước đi đã dùng;
* lý do thua phổ biến.

View này phân loại rủi ro thành các nhóm như:

* rủi ro do độ khó;
* rủi ro do rơi sau khi thắng;
* rủi ro kết hợp giữa độ khó và chuyển tiếp sau thắng;
* rủi ro do dữ liệu tracking;
* cần tiếp tục theo dõi.

#### `vw_f10_design_board_24h`

View này chuyển kết quả chẩn đoán thành bảng ưu tiên xử lý.

Mỗi dòng là một level trong 10 level đầu, kèm theo:

* mức ưu tiên;
* loại vấn đề chính;
* tín hiệu chính cần chú ý;
* gợi ý hướng kiểm tra;
* chỉ số liên quan để team thiết kế level có thể ra quyết định.

View này phù hợp để dùng trong cuộc họp review level.

### 8.4 View funnel tổng hợp từ `first_open` đến Level 10

#### `vw_funnel_first_open_to_level10`

View này là view funnel tổng hợp từ lần mở app đầu tiên đến Level 10 trong cửa sổ 24 giờ sau `first_open`.

Các bước chính:

```text
first_open
  ↓
Open_first
  ↓
Level 1
  ↓
Level 2
  ↓
...
  ↓
Level 10
```

View này được thiết kế để làm nguồn dữ liệu thao tác chính cho các phân tích:

* onboarding funnel;
* early progression funnel;
* drop-off theo từng step;
* conversion theo từng step;
* conversion từ `first_open`;
* conversion từ Level 1;
* level completion;
* win/fail/no-end;
* difficulty hoặc friction proxy;
* timing giữa các step;
* moves used;
* attempt 1 win và first end is win.

Grain của view là:

```text
1 app_version × 1 funnel step
```

View này đọc trực tiếp từ GA4 raw và dùng thêm:

```text
dim_f10_level_map
```

để mapping Level 1 đến Level 10 theo app version.

Điểm cần đọc thận trọng:

* view yêu cầu user đã đủ ít nhất 24 giờ quan sát sau `first_open`;
* view dùng sequential funnel nghiêm ngặt, nghĩa là user chỉ được tính ở Level N nếu đã có đủ các level trước đó theo đúng thứ tự;
* nhiều event follow-up được yêu cầu cùng `app_version` với version tại `first_open`;
* nếu tracking thiếu ở một step trước, các step sau có thể bị loại khỏi funnel dù user thực tế có thể đã chơi tiếp.

View này nên được dùng khi team cần một bảng tổng hợp để đọc nhanh toàn bộ hành trình từ mở app đến Level 10. Các view cũ như `vw_onboard_funnel_24h`, `vw_f10_funnel_24h`, `vw_f10_timing_24h` vẫn hữu ích khi cần phân tích chuyên biệt theo từng phần.

---

## 9. Tầng giám sát và phân tích level

Tầng này gồm 5 view.

### 9.1 `vw_user_level_reach`

View này tổng hợp dữ liệu theo người chơi và level.

Mỗi dòng thể hiện một người chơi đã từng chạm tới một level thường.

View này cho biết:

* lần đầu người chơi bắt đầu level;
* người chơi đã thắng level hay chưa;
* lần thắng đầu tiên;
* số lần thử;
* số lần có kết thúc;
* số lần bắt đầu nhưng không có kết thúc;
* level đã đủ 24 giờ để quan sát việc đi tiếp hay chưa.

Đây là nền tảng cho phân tích tiến trình từ level này sang level kế tiếp.

### 9.2 `vw_level_difficulty_daily`

View này tổng hợp độ khó level theo ngày.

Mỗi dòng là một ngày và một level.

View chỉ tính các level thường và loại bỏ các lần bắt đầu gần hướng dẫn.

Các chỉ số chính:

* số lần bắt đầu bình thường;
* số lần có kết thúc;
* tỷ lệ ghép được `End_level`;
* số lần thắng;
* số lần thua;
* tỷ lệ thắng;
* thời lượng chơi;
* số bước đi đã dùng;
* số bước còn lại;
* phân loại độ khó;
* ghi chú phân tích.

### 9.3 `vw_level_difficulty_summary`

View này tổng hợp độ khó level trên toàn bộ dữ liệu hiện có.

Khác với view theo ngày, view này không chia theo ngày. Mỗi dòng là một level.

View chỉ giữ các level có đủ số mẫu tối thiểu để tránh kết luận quá sớm từ dữ liệu quá ít.

### 9.4 `vw_level_progression_version`

View này phân tích tiến trình người chơi từ level hiện tại sang level kế tiếp theo từng phiên bản game.

View này giúp trả lời:

* bao nhiêu người đã tới level hiện tại;
* bao nhiêu người đã đủ thời gian quan sát;
* bao nhiêu người đi sang level tiếp theo;
* bao nhiêu người thắng level nhưng không đi tiếp;
* tỷ lệ rơi từ bắt đầu level hiện tại sang bắt đầu level tiếp theo;
* tỷ lệ rơi từ thắng level hiện tại sang bắt đầu level tiếp theo.

### 9.5 `vw_pipeline_health_daily`

View này là view kiểm tra sức khỏe dữ liệu hằng ngày.

View này theo dõi:

* số lượng sự kiện level;
* số lượng `Start_level`;
* số lượng `End_level`;
* số lượng sự kiện hướng dẫn;
* các trường bị thiếu;
* tỷ lệ ghép được `End_level`;
* tỷ lệ bắt đầu level gần hướng dẫn;
* tỷ lệ dữ liệu bị lệch phiên bản game;
* chất lượng các view downstream.

Trong tài liệu này, “downstream” nghĩa là các view nằm phía sau trong luồng xử lý và phụ thuộc vào dữ liệu từ view trước đó.

---

## 10. Tầng phân tích retention

Tầng này hiện có 1 view.

### 10.1 `vw_d1_retention`

View này dùng để phân tích D1 Retention theo `app_version`.

View phục vụ 3 metric chính:

1. D1 App Retention
2. D1 Gameplay Retention
3. D1 Gameplay Activation among Returners

Trong đó:

* **D1 App Retention**: user quay lại app vào ngày D1 và có ít nhất 1 active event.
* **D1 Gameplay Retention**: user quay lại app vào ngày D1 và có ít nhất 1 event `Start_level`.
* **D1 Gameplay Activation among Returners**: trong nhóm user đã quay lại app vào D1, bao nhiêu phần trăm user thật sự bắt đầu chơi level.

View đọc trực tiếp từ GA4 daily export:

```text
analytics_524104373.events_*
```

View không dùng `event_params` cho logic D1 hiện tại.

Grain của view có hai loại:

* `daily`: mỗi row là 1 `app_version` × 1 `cohort_date` × 1 `d1_date`;
* `total`: mỗi row là tổng hợp toàn bộ cohort đã đủ maturity của 1 `app_version`.

Logic ngày:

* `cohort_date` là ngày local mà user có `first_open` đầu tiên;
* `d1_date = cohort_date + 1 day`;
* timezone dùng để tính ngày là `Asia/Ho_Chi_Minh`;
* D1 ở view này là calendar day, không phải rolling 24h window.

App version assignment:

* user được gán vào `app_version` tại event `first_open` đầu tiên;
* nếu user quay lại D1 ở app version khác, user vẫn thuộc cohort app version tại `first_open`.

View này phù hợp để đánh giá:

* chất lượng cohort theo app version;
* khả năng kéo user quay lại sau ngày đầu tiên;
* mức độ user quay lại nhưng không chơi level;
* tác động của onboarding, first session và early level design lên retention ngày kế tiếp.

---

## 11. Sơ đồ phụ thuộc tổng thể

Sơ đồ phụ thuộc có thể đọc như sau:

```text
Dữ liệu thô GA4
analytics_524104373.events_*
  ├── vw_level_start_eff
  ├── vw_onboard_events
  ├── vw_level_events
  ├── vw_tutorial_events
  ├── vw_funnel_first_open_to_level10
  └── vw_d1_retention
```

Từ các view lõi:

```text
vw_level_events
vw_tutorial_events
  └── vw_level_tutorial_flag

vw_level_events
vw_level_tutorial_flag
  └── vw_level_attempts
```

Nhóm người chơi mới:

```text
dim_f10_level_map
vw_onboard_events
vw_level_start_eff
  ├── vw_onboard_funnel_24h
  └── vw_onboard_drop_diag_24h
```

Nhóm 10 level đầu:

```text
dim_f10_level_map
vw_level_start_eff
  └── vw_f10_funnel_24h

dim_f10_level_map
vw_level_start_eff
vw_level_events
  └── vw_f10_timing_24h

vw_f10_funnel_24h
vw_f10_timing_24h
  └── vw_f10_funnel_timing_24h
```

Nhóm funnel tổng hợp từ `first_open` đến Level 10:

```text
GA4 raw events_*
dim_f10_level_map
  └── vw_funnel_first_open_to_level10
```

Nhóm chẩn đoán thiết kế level:

```text
dim_f10_level_map
vw_level_start_eff
vw_level_attempts
vw_f10_funnel_timing_24h
  └── vw_f10_design_diag_24h
        └── vw_f10_design_board_24h
```

Nhóm giám sát và phân tích level:

```text
vw_level_attempts
  ├── vw_user_level_reach
  ├── vw_level_difficulty_daily
  └── vw_level_difficulty_summary

vw_user_level_reach
  └── vw_level_progression_version

vw_level_events
vw_tutorial_events
vw_level_tutorial_flag
vw_level_attempts
vw_user_level_reach
vw_level_difficulty_daily
vw_level_progression_version
  └── vw_pipeline_health_daily
```

Nhóm retention:

```text
GA4 raw events_*
  └── vw_d1_retention
```

---

## 12. Những nguyên tắc phân tích chính

### 12.1 Không phân tích độ khó từ dữ liệu bị ảnh hưởng bởi hướng dẫn nếu không tách riêng

Các lần bắt đầu level gần `Tutorial_start` có thể không phản ánh hành vi chơi tự nhiên.

Vì vậy, các view độ khó như `vw_level_difficulty_daily` và `vw_level_difficulty_summary` loại bỏ các lần bắt đầu gần hướng dẫn.

### 12.2 Phải kiểm tra tỷ lệ ghép được `End_level`

Nếu nhiều lần `Start_level` không ghép được với `End_level`, không nên kết luận mạnh về độ khó level.

Trường hợp này có thể do:

* người chơi thoát game giữa level;
* thiếu tracking `End_level`;
* lỗi gửi sự kiện;
* sai `attempt_no`;
* thời gian ghép bị vượt quá cửa sổ 6 giờ.

### 12.3 Cần dùng nhóm đã đủ thời gian quan sát

Các view có hậu tố `_24h` thường dùng nhóm người chơi đã đủ 24 giờ kể từ mốc bắt đầu phân tích.

Mục đích là tránh kết luận sai khi người chơi chưa có đủ thời gian đi tiếp.

Ví dụ:

* `vw_onboard_funnel_24h` dùng cửa sổ 24 giờ cho onboarding;
* `vw_f10_funnel_24h` dùng cửa sổ 24 giờ cho first 10 levels;
* `vw_funnel_first_open_to_level10` dùng cửa sổ 24 giờ sau `first_open`.

### 12.4 Cần tách rơi trước khi thắng và rơi sau khi thắng

Một level có thể gây rơi người chơi theo hai cách khác nhau:

1. người chơi không thắng được level;
2. người chơi đã thắng nhưng không tiếp tục sang level kế tiếp.

Hai trường hợp này có nguyên nhân thiết kế khác nhau, nên cần phân tích riêng.

### 12.5 Phân biệt D1 calendar day và rolling 24h window

`vw_d1_retention` định nghĩa D1 theo calendar day local:

```text
d1_date = cohort_date + 1 day
```

Trong khi đó, `vw_funnel_first_open_to_level10` dùng rolling 24h window:

```text
[first_open_time_utc, first_open_time_utc + 24 hours]
```

Hai logic này phục vụ hai câu hỏi khác nhau, không nên so sánh trực tiếp như cùng một loại window.

Ví dụ:

* D1 calendar day trả lời: “User có quay lại vào ngày hôm sau không?”
* Rolling 24h window trả lời: “Trong 24 giờ đầu sau khi mở app lần đầu, user đi được tới đâu?”

### 12.6 Phân biệt app version tại `first_open` và app version tại event follow-up

Một số view gán user vào `app_version` tại `first_open`.

Cần đọc kỹ từng đặc tả để biết event follow-up có bắt buộc cùng `app_version` hay không.

Ví dụ:

* `vw_d1_retention` vẫn tính D1 activity dù user quay lại ở app version khác, nhưng user vẫn thuộc cohort version tại `first_open`;
* `vw_funnel_first_open_to_level10` yêu cầu nhiều event trong funnel khớp với `app_version` tại `first_open`.

### 12.7 Sequential funnel sạch nhưng có thể nhạy với lỗi tracking

`vw_funnel_first_open_to_level10` dùng sequential funnel nghiêm ngặt.

Điều này giúp funnel sạch và dễ đọc, nhưng nếu tracking thiếu một step trước đó, các step sau có thể không được tính là reached hợp lệ.

Vì vậy, khi thấy drop bất thường ở một step, cần kiểm tra thêm:

* event của step trước có bị thiếu không;
* mapping level có đúng theo app version không;
* `Open_first` có bị thiếu không;
* `Start_level` và `End_level` có được gửi đúng không;
* có thay đổi tracking giữa các app version không.

---

## 13. Cấu trúc thư mục đề xuất trên GitHub

Nên lưu tài liệu theo cấu trúc sau:

```text
game-analytics-mart-docs/
├── README.md
├── docs/
│   ├── pipeline/
│   │   └── framework_pipeline.md
│   ├── graph/
│   │   └── mart_dependency_graph.md
│   └── objects/
│       ├── core_foundation/
│       │   ├── dim_f10_level_map.md
│       │   ├── dim_level_seq.md
│       │   ├── vw_level_start_eff.md
│       │   ├── vw_onboard_events.md
│       │   ├── vw_level_events.md
│       │   ├── vw_tutorial_events.md
│       │   ├── vw_level_tutorial_flag.md
│       │   └── vw_level_attempts.md
│       ├── current_live_analysis/
│       │   ├── vw_onboard_funnel_24h.md
│       │   ├── vw_onboard_drop_diag_24h.md
│       │   ├── vw_f10_funnel_24h.md
│       │   ├── vw_f10_timing_24h.md
│       │   ├── vw_f10_funnel_timing_24h.md
│       │   ├── vw_f10_design_diag_24h.md
│       │   ├── vw_f10_design_board_24h.md
│       │   └── vw_funnel_first_open_to_level10.md
│       ├── monitoring_level_analysis/
│       │   ├── vw_user_level_reach.md
│       │   ├── vw_level_difficulty_daily.md
│       │   ├── vw_level_difficulty_summary.md
│       │   ├── vw_level_progression_version.md
│       │   └── vw_pipeline_health_daily.md
│       └── retention_analysis/
│           └── vw_d1_retention.md
├── sql/
│   └── ddl/
│       ├── core_foundation/
│       ├── current_live_analysis/
│       ├── monitoring_level_analysis/
│       └── retention_analysis/
└── metadata/
    ├── object_inventory.json
    ├── column_schema.json
    └── dependency_graph.json
```

---

## 14. Cấu trúc chuẩn cho mỗi bản đặc tả chi tiết

Mỗi bảng hoặc view nên có một file đặc tả riêng theo mẫu sau:

```text
# <tên bảng hoặc view>

## 1. Mục đích
## 2. Vai trò trong hệ thống
## 3. Câu hỏi phân tích có thể trả lời
## 4. Độ chi tiết của mỗi dòng dữ liệu
## 5. Loại đối tượng
## 6. Nguồn dữ liệu phụ thuộc
## 7. Danh sách cột đầu ra
## 8. Trường khóa và trường định danh quan trọng
## 9. Chỉ số chính
## 10. Logic xử lý dữ liệu
## 11. Bộ lọc và điều kiện chọn dữ liệu
## 12. Lưu ý về chất lượng dữ liệu
## 13. Cách sử dụng khuyến nghị
## 14. Ví dụ truy vấn
## 15. DDL tham chiếu
## 16. Liên kết liên quan
```

Trong đó:

* “độ chi tiết của mỗi dòng dữ liệu” trả lời câu hỏi: một dòng trong bảng đại diện cho cái gì;
* “nguồn dữ liệu phụ thuộc” liệt kê các bảng hoặc view mà đối tượng đang đọc;
* “chỉ số chính” giải thích các cột đo lường như retention, tỷ lệ thắng, tỷ lệ rơi, số người chơi;
* “logic xử lý dữ liệu” mô tả cách dữ liệu được lọc, ghép, tính toán hoặc phân loại;
* “DDL tham chiếu” lưu câu lệnh định nghĩa bảng hoặc view;
* “Liên kết liên quan” giúp graph view và backlink trong repository dễ đọc hơn.

---

## 15. Thứ tự đọc khuyến nghị cho người mới trong team

Người mới nên đọc theo thứ tự:

```text
1. docs/pipeline/framework_pipeline.md
2. docs/graph/mart_dependency_graph.md

3. Nhóm nền tảng lõi:
   - dim_f10_level_map.md
   - dim_level_seq.md
   - vw_level_start_eff.md
   - vw_onboard_events.md
   - vw_level_events.md
   - vw_tutorial_events.md
   - vw_level_tutorial_flag.md
   - vw_level_attempts.md

4. Nhóm phân tích người chơi mới và 10 level đầu:
   - vw_onboard_funnel_24h.md
   - vw_onboard_drop_diag_24h.md
   - vw_f10_funnel_24h.md
   - vw_f10_timing_24h.md
   - vw_f10_funnel_timing_24h.md
   - vw_funnel_first_open_to_level10.md

5. Nhóm chẩn đoán thiết kế level:
   - vw_f10_design_diag_24h.md
   - vw_f10_design_board_24h.md

6. Nhóm giám sát:
   - vw_user_level_reach.md
   - vw_level_difficulty_daily.md
   - vw_level_difficulty_summary.md
   - vw_level_progression_version.md
   - vw_pipeline_health_daily.md

7. Nhóm retention:
   - vw_d1_retention.md
```

Nếu mục tiêu là thao tác phân tích nhanh cho prototype hiện tại, có thể đọc trước các view sau:

```text
1. vw_funnel_first_open_to_level10.md
2. vw_d1_retention.md
3. vw_pipeline_health_daily.md
4. vw_level_difficulty_daily.md
5. vw_f10_design_board_24h.md
```

---

## 16. Khi nào nên dùng view nào?

| Câu hỏi phân tích | View khuyến nghị | Lý do |
| ----------------- | ---------------- | ----- |
| User đi từ `first_open` đến Level 10 như thế nào trong 24 giờ đầu? | `vw_funnel_first_open_to_level10` | View tổng hợp đầy đủ onboarding, level funnel, win/fail/no-end, timing và move usage. |
| User có quay lại ngày hôm sau không? | `vw_d1_retention` | View được thiết kế riêng cho D1 App Retention và D1 Gameplay Retention. |
| Level nào trong 10 level đầu nên ưu tiên review thiết kế? | `vw_f10_design_board_24h` | View đã chuyển tín hiệu phân tích thành bảng ưu tiên hành động. |
| Muốn kiểm tra level difficulty theo ngày | `vw_level_difficulty_daily` | View theo dõi win rate, fail rate, duration, move usage theo ngày và level. |
| Muốn kiểm tra difficulty tổng hợp | `vw_level_difficulty_summary` | View tổng hợp toàn bộ dữ liệu đủ mẫu theo level. |
| Muốn phân tích user đi từ level này sang level tiếp theo | `vw_level_progression_version` | View phân tích progression theo app version. |
| Muốn kiểm tra tracking có đáng tin không | `vw_pipeline_health_daily` | View theo dõi lỗi event, thiếu param, attempt matching và sức khỏe pipeline. |
| Muốn phân tích từng attempt | `vw_level_attempts` | View grain là một lần bắt đầu level được ghép với kết thúc nếu có. |

---

## 17. Trạng thái hiện tại của kiến trúc

Tính đến thời điểm tài liệu này được cập nhật:

```text
Tổng số đối tượng: 22
Bảng vật lý: 2
View: 20
Đối tượng cũ đã loại bỏ: 0
Tham chiếu tới tên cũ: 0
Đối tượng REVIEW: 0
```

Đã bổ sung vào kiến trúc:

* `vw_funnel_first_open_to_level10` trong nhóm `current_live_analysis`;
* `vw_d1_retention` trong nhóm `retention_analysis`;
* nhóm tài liệu mới `retention_analysis`;
* lưu ý phân biệt D1 calendar day và rolling 24h window;
* lưu ý phân biệt app version tại `first_open` và app version tại follow-up event;
* hướng dẫn dùng view theo từng câu hỏi phân tích.

Kiến trúc hiện tại đã đủ sạch để đưa vào tài liệu nội bộ và sử dụng làm nền tảng cho phân tích dữ liệu game trong giai đoạn prototype.
