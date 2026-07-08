# Game Analytics Mart Documentation

Repository này lưu tài liệu kỹ thuật cho hệ thống phân tích dữ liệu gameplay của dự án mobile puzzle match-3.

Mục tiêu của repository là giúp team hiểu rõ:

* dữ liệu được lấy từ đâu;
* các bảng và view trong BigQuery được xây dựng như thế nào;
* từng object phục vụ mục đích phân tích gì;
* object nào phụ thuộc object nào;
* cách đọc kết quả phân tích onboarding, 10 level đầu, retention, độ khó level, progression và sức khỏe pipeline.

Trong tài liệu này:

* **GA4** là Google Analytics 4, nguồn tracking sự kiện từ game.
* **BigQuery** là nơi lưu và truy vấn dữ liệu.
* **Dataset** là vùng chứa dữ liệu trong BigQuery.
* **View** là bảng ảo trong BigQuery, không lưu dữ liệu vật lý mà lưu logic truy vấn.
* **Mart** là lớp dữ liệu đã được xử lý để phục vụ phân tích.
* **Pipeline** là luồng xử lý dữ liệu từ nguồn thô đến bảng phân tích cuối.
* **Cohort** là nhóm user được gom theo cùng một điều kiện ban đầu, ví dụ cùng ngày `first_open` hoặc cùng `app_version` tại `first_open`.
* **Retention** là chỉ số đo user có quay lại app hoặc quay lại gameplay sau một khoảng thời gian nhất định hay không.

---

## 1. Phạm vi repository

Repository này tập trung vào dataset:

```text
project-feb1f7ca-3dbf-419f-aa8.game_analytics_mart
```

Nguồn dữ liệu thô chính:

```text
project-feb1f7ca-3dbf-419f-aa8.analytics_524104373.events_*
```

Repository này không lưu dữ liệu raw, không lưu dữ liệu người dùng thật, không lưu credential và không lưu file private key.

Repository chỉ nên lưu:

* tài liệu đặc tả;
* DDL của bảng/view;
* sơ đồ phụ thuộc;
* quy ước phân tích;
* hướng dẫn kiểm tra dữ liệu;
* truy vấn mẫu phục vụ phân tích.

---

## 2. Trạng thái hiện tại

Data mart hiện có 22 object chính:

| Nhóm                      | Số lượng | Vai trò                                                                     |
| ------------------------- | -------: | --------------------------------------------------------------------------- |
| Core foundation           |        8 | Lớp nền: mapping, event sạch, tutorial flag, level attempts.                |
| Current live analysis     |        8 | Phân tích onboarding, first 10 levels, funnel tổng hợp, timing, diagnostics. |
| Monitoring level analysis |        5 | Theo dõi reach, difficulty, progression và pipeline health.                 |
| Retention analysis        |        1 | Phân tích retention theo cohort, app version và hành vi quay lại gameplay.  |

Tổng cộng:

```text
22 objects
```

---

## 3. Cấu trúc thư mục đề xuất

```text
.
├── README.md
├── docs/
│   ├── pipeline/
│   │   └── framework_pipeline.md
│   │
│   ├── graph/
│   │   └── mart_dependency_graph.md
│   │
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
│       │
│       ├── current_live_analysis/
│       │   ├── vw_onboard_funnel_24h.md
│       │   ├── vw_onboard_drop_diag_24h.md
│       │   ├── vw_f10_funnel_24h.md
│       │   ├── vw_f10_timing_24h.md
│       │   ├── vw_f10_funnel_timing_24h.md
│       │   ├── vw_f10_design_diag_24h.md
│       │   ├── vw_f10_design_board_24h.md
│       │   └── vw_funnel_first_open_to_level10.md
│       │
│       ├── monitoring_level_analysis/
│       │   ├── vw_user_level_reach.md
│       │   ├── vw_level_difficulty_daily.md
│       │   ├── vw_level_difficulty_summary.md
│       │   ├── vw_level_progression_version.md
│       │   └── vw_pipeline_health_daily.md
│       │
│       └── retention_analysis/
│           └── vw_d1_retention.md
│
└── sql/
    └── ddl/
        ├── core_foundation/
        ├── current_live_analysis/
        ├── monitoring_level_analysis/
        └── retention_analysis/
```

---

## 4. Danh sách object

### 4.1 Core foundation

| Object                   | Loại  | Mục đích                                                       |
| ------------------------ | ----- | -------------------------------------------------------------- |
| `dim_f10_level_map`      | Table | Mapping 10 level đầu theo app version.                         |
| `dim_level_seq`          | Table | Mapping sequence tổng quát của normal level.                   |
| `vw_level_start_eff`     | View  | Chuẩn hóa effective level start, bao gồm pseudo start khi cần. |
| `vw_onboard_events`      | View  | Làm sạch event onboarding.                                     |
| `vw_level_events`        | View  | Làm sạch `Start_level` và `End_level`.                         |
| `vw_tutorial_events`     | View  | Làm sạch tutorial events.                                      |
| `vw_level_tutorial_flag` | View  | Gắn cờ `Start_level` có gần tutorial hay không.                |
| `vw_level_attempts`      | View  | Ghép `Start_level` với `End_level` thành level attempt.        |

---

### 4.2 Current live analysis

| Object                                | Loại | Mục đích                                                                                                                         |
| ------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------- |
| `vw_onboard_funnel_24h`               | View | Phân tích onboarding funnel trong 24 giờ.                                                                                        |
| `vw_onboard_drop_diag_24h`            | View | Chẩn đoán điểm rơi onboarding.                                                                                                   |
| `vw_f10_funnel_24h`                   | View | Phân tích funnel 10 level đầu trong 24 giờ.                                                                                      |
| `vw_f10_timing_24h`                   | View | Phân tích thời gian đi qua 10 level đầu.                                                                                         |
| `vw_f10_funnel_timing_24h`            | View | Kết hợp funnel và timing của 10 level đầu.                                                                                       |
| `vw_f10_design_diag_24h`              | View | Chẩn đoán thiết kế 10 level đầu.                                                                                                 |
| `vw_f10_design_board_24h`             | View | Bảng ưu tiên hành động cho thiết kế level.                                                                                       |
| `vw_funnel_first_open_to_level10`     | View | Funnel tổng hợp từ `first_open` đến Level 10 trong 24 giờ, gồm onboarding, early progression, drop-off, win/fail/no-end, timing và move usage. |

---

### 4.3 Monitoring level analysis

| Object                         | Loại | Mục đích                                               |
| ------------------------------ | ---- | ------------------------------------------------------ |
| `vw_user_level_reach`          | View | Tổng hợp trạng thái người chơi theo từng normal level. |
| `vw_level_difficulty_daily`    | View | Theo dõi độ khó level theo ngày.                       |
| `vw_level_difficulty_summary`  | View | Tổng hợp độ khó level trên toàn bộ dữ liệu.            |
| `vw_level_progression_version` | View | Phân tích progression theo app version.                |
| `vw_pipeline_health_daily`     | View | Kiểm tra sức khỏe kỹ thuật của pipeline hằng ngày.     |

---

### 4.4 Retention analysis

| Object            | Loại | Mục đích                                                                                                  |
| ----------------- | ---- | --------------------------------------------------------------------------------------------------------- |
| `vw_d1_retention` | View | Phân tích D1 App Retention, D1 Gameplay Retention và D1 Gameplay Activation among Returners theo `app_version`. |

---

## 5. Sơ đồ phụ thuộc tổng quát

```mermaid
flowchart TD

  RAW["GA4 raw events_*"]

  DIM_F10["dim_f10_level_map"]
  DIM_SEQ["dim_level_seq"]

  START_EFF["vw_level_start_eff"]
  ONBOARD_EVENTS["vw_onboard_events"]
  LEVEL_EVENTS["vw_level_events"]
  TUTORIAL_EVENTS["vw_tutorial_events"]

  LEVEL_TUTORIAL_FLAG["vw_level_tutorial_flag"]
  LEVEL_ATTEMPTS["vw_level_attempts"]

  ONBOARD_FUNNEL["vw_onboard_funnel_24h"]
  ONBOARD_DROP["vw_onboard_drop_diag_24h"]

  F10_FUNNEL["vw_f10_funnel_24h"]
  F10_TIMING["vw_f10_timing_24h"]
  F10_FUNNEL_TIMING["vw_f10_funnel_timing_24h"]
  F10_DESIGN_DIAG["vw_f10_design_diag_24h"]
  F10_DESIGN_BOARD["vw_f10_design_board_24h"]

  FUNNEL_FO_L10["vw_funnel_first_open_to_level10"]

  USER_REACH["vw_user_level_reach"]
  DIFF_DAILY["vw_level_difficulty_daily"]
  DIFF_SUMMARY["vw_level_difficulty_summary"]
  PROGRESSION_VERSION["vw_level_progression_version"]
  PIPELINE_HEALTH["vw_pipeline_health_daily"]

  D1_RETENTION["vw_d1_retention"]

  RAW --> START_EFF
  RAW --> ONBOARD_EVENTS
  RAW --> LEVEL_EVENTS
  RAW --> TUTORIAL_EVENTS

  LEVEL_EVENTS --> LEVEL_TUTORIAL_FLAG
  TUTORIAL_EVENTS --> LEVEL_TUTORIAL_FLAG

  LEVEL_TUTORIAL_FLAG --> LEVEL_ATTEMPTS
  LEVEL_EVENTS --> LEVEL_ATTEMPTS

  DIM_F10 --> ONBOARD_FUNNEL
  ONBOARD_EVENTS --> ONBOARD_FUNNEL
  START_EFF --> ONBOARD_FUNNEL

  DIM_F10 --> ONBOARD_DROP
  ONBOARD_EVENTS --> ONBOARD_DROP
  START_EFF --> ONBOARD_DROP

  DIM_F10 --> F10_FUNNEL
  START_EFF --> F10_FUNNEL

  DIM_F10 --> F10_TIMING
  START_EFF --> F10_TIMING
  LEVEL_EVENTS --> F10_TIMING

  F10_FUNNEL --> F10_FUNNEL_TIMING
  F10_TIMING --> F10_FUNNEL_TIMING

  DIM_F10 --> F10_DESIGN_DIAG
  START_EFF --> F10_DESIGN_DIAG
  LEVEL_ATTEMPTS --> F10_DESIGN_DIAG
  F10_FUNNEL_TIMING --> F10_DESIGN_DIAG

  F10_DESIGN_DIAG --> F10_DESIGN_BOARD

  RAW --> FUNNEL_FO_L10
  DIM_F10 --> FUNNEL_FO_L10

  LEVEL_ATTEMPTS --> USER_REACH
  LEVEL_ATTEMPTS --> DIFF_DAILY
  LEVEL_ATTEMPTS --> DIFF_SUMMARY

  USER_REACH --> PROGRESSION_VERSION

  RAW --> D1_RETENTION

  LEVEL_EVENTS --> PIPELINE_HEALTH
  TUTORIAL_EVENTS --> PIPELINE_HEALTH
  LEVEL_TUTORIAL_FLAG --> PIPELINE_HEALTH
  LEVEL_ATTEMPTS --> PIPELINE_HEALTH
  USER_REACH --> PIPELINE_HEALTH
  DIFF_DAILY --> PIPELINE_HEALTH
  PROGRESSION_VERSION --> PIPELINE_HEALTH
```

---

## 6. Thứ tự đọc tài liệu khuyến nghị

Nếu đọc lần đầu, nên theo thứ tự sau:

1. `docs/pipeline/framework_pipeline.md`
2. `docs/graph/mart_dependency_graph.md`
3. `docs/objects/core_foundation/dim_f10_level_map.md`
4. `docs/objects/core_foundation/dim_level_seq.md`
5. `docs/objects/core_foundation/vw_level_start_eff.md`
6. `docs/objects/core_foundation/vw_onboard_events.md`
7. `docs/objects/core_foundation/vw_level_events.md`
8. `docs/objects/core_foundation/vw_tutorial_events.md`
9. `docs/objects/core_foundation/vw_level_tutorial_flag.md`
10. `docs/objects/core_foundation/vw_level_attempts.md`
11. Các view trong `current_live_analysis`
12. `docs/objects/current_live_analysis/vw_funnel_first_open_to_level10.md`
13. Các view trong `monitoring_level_analysis`
14. `docs/objects/retention_analysis/vw_d1_retention.md`

---

## 7. Quy ước đặt tên

### 7.1 Prefix object

| Prefix  | Ý nghĩa                                                |
| ------- | ------------------------------------------------------ |
| `dim_`  | Bảng dimension hoặc mapping.                           |
| `vw_`   | View trong BigQuery.                                   |
| `mart_` | Bảng vật lý phục vụ phân tích, nếu có trong tương lai. |

---

### 7.2 Viết tắt chuẩn

| Viết tắt  | Ý nghĩa                                                       |
| --------- | ------------------------------------------------------------- |
| `f10`     | First 10 levels.                                              |
| `d1`      | Day 1, tức ngày sau cohort date hoặc mốc đầu tùy định nghĩa.  |
| `onboard` | Onboarding.                                                   |
| `diag`    | Diagnostics, tức chẩn đoán.                                   |
| `seq`     | Sequence, tức chuỗi thứ tự level.                             |

---

### 7.3 Hậu tố thời gian

| Hậu tố     | Ý nghĩa                                         |
| ---------- | ----------------------------------------------- |
| `_24h`     | Chỉ số trong vòng 24 giờ.                       |
| `_daily`   | Tổng hợp theo ngày.                             |
| `_summary` | Tổng hợp toàn bộ dữ liệu hoặc tổng hợp cấp cao. |

---

### 7.4 Quy ước chính tả metric

Dùng thống nhất từ **retention** cho tên tài liệu, tên metric và mô tả phân tích.

---

## 8. Quy ước tài liệu object spec

Mỗi file đặc tả object nên có cấu trúc sau:

```text
# <object_name>

## 1. Mục đích
## 2. Vai trò trong hệ thống
## 3. Câu hỏi phân tích có thể trả lời
## 4. Độ chi tiết của mỗi dòng dữ liệu
## 5. Loại đối tượng
## 6. Nguồn dữ liệu phụ thuộc
## 7. Danh sách cột
## 8. Logic xử lý chính
## 9. Kiểm tra chất lượng dữ liệu khuyến nghị
## 10. Cách sử dụng khuyến nghị
## 11. Lưu ý khi sử dụng
## 12. Rủi ro nếu view sai logic
## 13. Mức độ quan trọng
## 14. DDL tham chiếu
## Liên kết liên quan
```

Mục `## Liên kết liên quan` nên được thêm vào cuối mỗi file để hỗ trợ backlink và graph view.

---

## 9. Các nhóm phân tích chính

### 9.1 Onboarding

Các view liên quan:

* `vw_onboard_funnel_24h`
* `vw_onboard_drop_diag_24h`
* `vw_funnel_first_open_to_level10`

Mục tiêu:

* đo người chơi có đi từ `first_open` đến `Open_first`, `Vuot_home`, Level 1 hay không;
* chẩn đoán điểm rơi trong onboarding;
* kiểm tra tác động của onboarding tới early progression trong 24 giờ đầu;
* phát hiện các điểm user mở app nhưng chưa bắt đầu gameplay.

Lưu ý:

* `vw_onboard_funnel_24h` và `vw_onboard_drop_diag_24h` là các view chuyên biệt cho onboarding.
* `vw_funnel_first_open_to_level10` là view tổng hợp, phù hợp khi cần đọc onboarding cùng với Level 1 đến Level 10 trong cùng một funnel.

---

### 9.2 First 10 levels

Các view liên quan:

* `vw_f10_funnel_24h`
* `vw_f10_timing_24h`
* `vw_f10_funnel_timing_24h`
* `vw_f10_design_diag_24h`
* `vw_f10_design_board_24h`
* `vw_funnel_first_open_to_level10`

Mục tiêu:

* đo phễu 10 level đầu;
* đo thời gian đi qua từng level;
* phát hiện level có rủi ro thiết kế;
* tạo bảng ưu tiên hành động;
* đo drop-off từ `first_open`, từ `Open_first`, từ Level 1 và từ step liền trước;
* đọc win/fail/no-end, move usage, duration và attempt trong bối cảnh early funnel.

Lưu ý:

* `vw_funnel_first_open_to_level10` phù hợp khi cần một bảng tổng hợp để thao tác phân tích hằng ngày.
* Các view `vw_f10_*` hiện có vẫn hữu ích khi cần đọc các logic chuyên biệt đã tách sẵn như timing, diagnostics hoặc design board.

---

### 9.3 Level difficulty

Các view liên quan:

* `vw_level_difficulty_daily`
* `vw_level_difficulty_summary`
* `vw_funnel_first_open_to_level10`

Mục tiêu:

* theo dõi win rate, fail rate, duration và move usage;
* phân loại level theo độ khó;
* tách tracking risk khỏi difficulty risk;
* phát hiện level có friction sớm trong 10 level đầu;
* hỗ trợ quyết định chỉnh số move, layout, obstacle, tutorial hoặc difficulty ramp.

Lưu ý:

* Với phân tích difficulty toàn bộ level, ưu tiên dùng `vw_level_difficulty_daily` hoặc `vw_level_difficulty_summary`.
* Với phân tích difficulty trong hành trình từ `first_open` đến Level 10, có thể dùng `vw_funnel_first_open_to_level10`.

---

### 9.4 Level progression

Các view liên quan:

* `vw_user_level_reach`
* `vw_level_progression_version`
* `vw_funnel_first_open_to_level10`

Mục tiêu:

* đo người chơi đã tới và thắng từng normal level;
* đo tỷ lệ đi từ level hiện tại sang level kế tiếp;
* đo rơi sau khi thắng;
* theo dõi theo app version;
* phát hiện churn point trong early progression.

Lưu ý:

* `vw_user_level_reach` phù hợp để đọc trạng thái user theo từng normal level.
* `vw_funnel_first_open_to_level10` phù hợp để đọc progression nghiêm ngặt theo thứ tự từ Level 1 đến Level 10 trong 24 giờ đầu.

---

### 9.5 Pipeline health

View liên quan:

* `vw_pipeline_health_daily`

Mục tiêu:

* kiểm tra event bị thiếu;
* kiểm tra app version bị thiếu;
* kiểm tra tham số bắt buộc;
* kiểm tra attempt matching;
* kiểm tra progression mart;
* cảnh báo lỗi kỹ thuật theo ngày.

---

### 9.6 Retention analysis

View liên quan:

* `vw_d1_retention`

Mục tiêu:

* đo D1 App Retention theo `app_version`;
* đo D1 Gameplay Retention theo `app_version`;
* đo D1 Gameplay Activation among Returners, tức trong nhóm user đã quay lại app vào D1, bao nhiêu user thật sự bắt đầu chơi level;
* so sánh chất lượng cohort giữa các app version;
* hỗ trợ đánh giá tác động của onboarding, early level design và chất lượng phiên chơi đầu tiên lên khả năng quay lại ngày hôm sau.

Lưu ý:

* `vw_d1_retention` định nghĩa D1 theo calendar day local, không phải rolling 24h window.
* User được gán vào `app_version` tại event `first_open` đầu tiên.
* D1 activity có thể xảy ra ở app version khác, nhưng user vẫn thuộc cohort version tại `first_open`.

---

## 10. Một số lưu ý quan trọng

### 10.1 Không dùng mọi view cho cùng một mục đích

Ví dụ:

* muốn phân tích từng attempt, dùng `vw_level_attempts`;
* muốn phân tích từng user theo level, dùng `vw_user_level_reach`;
* muốn phân tích funnel 10 level đầu theo các view chuyên biệt, dùng `vw_f10_funnel_24h`, `vw_f10_timing_24h`, `vw_f10_funnel_timing_24h`, `vw_f10_design_diag_24h` hoặc `vw_f10_design_board_24h`;
* muốn phân tích funnel tổng hợp từ `first_open` đến Level 10, dùng `vw_funnel_first_open_to_level10`;
* muốn phân tích D1 retention theo cohort và app version, dùng `vw_d1_retention`;
* muốn kiểm tra lỗi tracking, dùng `vw_pipeline_health_daily`.

---

### 10.2 Phân biệt raw start và effective start

Một số view dùng `vw_level_start_eff`, tức là effective level start.

Một số view dùng `vw_level_attempts`, tức là dữ liệu bắt đầu level đã ghép attempt.

Một số view, như `vw_funnel_first_open_to_level10`, có logic effective level start riêng trong chính view, bao gồm raw `Start_level` và pseudo `Start_level` cho một số case đặc biệt.

Cần phân biệt rõ nguồn level start khi so sánh số liệu giữa các view.

---

### 10.3 Không kết luận difficulty khi tracking chưa ổn

Nếu `matched_end_rate`, `end_level_coverage_rate_pct` hoặc coverage của `End_level` thấp, cần kiểm tra tracking trước khi kết luận level khó hay dễ.

Một level có nhiều `start_without_end_user_count` có thể là:

* user thật sự bỏ level;
* tracking thiếu `End_level`;
* level window bị sai;
* event bị lệch `app_version` hoặc `level_name`;
* user bị mất kết nối hoặc tắt app trước khi gửi event.

---

### 10.4 Các view 24 giờ chỉ nên đọc trên cohort đã đủ 24 giờ

Các view có hậu tố `_24h` thường dùng cửa sổ 24 giờ kể từ mốc bắt đầu.

Không nên so sánh người chơi chưa đủ thời gian quan sát với người chơi đã đủ thời gian.

`vw_funnel_first_open_to_level10` chỉ lấy user có:

```text
first_open_time_utc <= CURRENT_TIMESTAMP() - 24 hours
```

Mục đích là đảm bảo mỗi user trong cohort đã có đủ 24 giờ quan sát.

---

### 10.5 Phân biệt D1 calendar day và rolling 24h window

`vw_d1_retention` định nghĩa D1 theo calendar day local:

```text
d1_date = cohort_date + 1 day
```

Trong khi đó, `vw_funnel_first_open_to_level10` dùng cửa sổ quan sát 24 giờ kể từ `first_open_time_utc`:

```text
[first_open_time_utc, first_open_time_utc + 24 hours]
```

Hai logic này phục vụ hai câu hỏi khác nhau:

* D1 calendar day dùng để đo user có quay lại vào ngày hôm sau hay không.
* Rolling 24h window dùng để đo user làm được gì trong 24 giờ đầu sau lần mở app đầu tiên.

Không nên so sánh hai loại window này như cùng một định nghĩa thời gian.

---

### 10.6 Phân biệt app version tại first_open và app version tại event follow-up

Một số view gán user vào `app_version` tại `first_open`.

Cần đọc kỹ từng đặc tả để biết event follow-up có bắt buộc cùng `app_version` hay không.

Ví dụ:

* `vw_d1_retention` vẫn tính D1 activity dù user quay lại ở app version khác, nhưng user vẫn thuộc cohort version tại `first_open`.
* `vw_funnel_first_open_to_level10` yêu cầu nhiều event trong funnel khớp với `app_version` tại `first_open`.

Điều này có thể làm số liệu giữa retention và funnel khác nhau khi user update app giữa D0 và D1, hoặc giữa các event trong 24 giờ đầu.

---

### 10.7 Cẩn thận với sequential funnel nghiêm ngặt

`vw_funnel_first_open_to_level10` dùng logic sequential progression nghiêm ngặt.

Điều này có nghĩa là user chỉ được tính là reach Level N nếu các level trước đó đã có start hợp lệ và không bị sai thứ tự.

Ưu điểm:

* funnel sạch hơn;
* dễ phát hiện drop-off thật theo hành trình thiết kế;
* giảm rủi ro đếm nhầm user do event sai thứ tự.

Rủi ro:

* nếu tracking thiếu một step trước đó, các step sau có thể không được tính dù user thực tế đã chơi;
* nếu `level_name`, `app_version` hoặc timestamp bị lệch, funnel có thể thấp hơn reach thực tế.

Vì vậy, khi thấy drop bất thường trong `vw_funnel_first_open_to_level10`, nên kiểm tra thêm pipeline health và raw event sample trước khi kết luận thiết kế level có vấn đề.

---

### 10.8 Cẩn thận với định nghĩa active event trong D1 App Retention

Trong `vw_d1_retention`, D1 App Retention hiện tính user retained nếu user có ít nhất 1 active event vào `d1_date`.

Active event hiện được hiểu là event không thuộc nhóm:

```text
first_open, app_remove
```

Đây là định nghĩa có thể dùng trong giai đoạn đầu, nhưng cần đọc thận trọng nếu GA4 có nhiều event tự động hoặc event nền không phản ánh việc user thật sự quay lại chơi game.

Vì vậy, khi đánh giá chất lượng gameplay, nên đọc song song:

* D1 App Retention;
* D1 Gameplay Retention;
* D1 Gameplay Activation among Returners.

---

## 11. Quy trình bảo trì tài liệu

Khi thay đổi một object trong BigQuery:

1. Cập nhật DDL trong thư mục `sql/ddl/`.
2. Cập nhật object spec tương ứng trong `docs/objects/`.
3. Cập nhật `docs/graph/mart_dependency_graph.md` nếu dependency thay đổi.
4. Kiểm tra lại mục `## Liên kết liên quan` ở các spec liên quan.
5. Ghi rõ thay đổi nếu logic phân tích hoặc định nghĩa metric thay đổi.
6. Nếu thêm object mới, cập nhật lại tổng số object trong README.
7. Nếu thêm nhóm phân tích mới, cập nhật lại cấu trúc thư mục, danh sách object và các nhóm phân tích chính.

---

## 12. Không đưa vào repository

Không đưa các nội dung sau vào repository:

* private key;
* credential;
* raw export chứa dữ liệu người dùng;
* file chứa email, số điện thoại, hoặc thông tin cá nhân;
* dữ liệu đầy đủ từ GA4;
* token truy cập;
* thông tin nội bộ không cần thiết cho phân tích.

---

## 13. Trạng thái tài liệu

Hiện tại đã có đặc tả cho:

```text
22 / 22 objects
```

Đã có thêm:

```text
Framework Pipeline
Mart Dependency Graph
Backlink sections cho từng object spec
Retention analysis group
Unified first_open to Level 10 funnel spec
D1 retention spec
```

Các object mới đã được bổ sung vào hệ thống tài liệu tổng quan:

* `vw_funnel_first_open_to_level10`
* `vw_d1_retention`

Repository này có thể tiếp tục mở rộng thêm:

* dashboard spec;
* metric dictionary;
* data quality checklist;
* release note theo app version;
* changelog thay đổi SQL;
* naming convention chi tiết hơn;
* test query cho từng view;
* retention mở rộng cho D3, D7, D14, D30;
* dashboard hướng product cho onboarding, early funnel, retention, progression, difficulty và pipeline health.
