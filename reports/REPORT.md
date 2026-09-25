# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đỗ Lý Minh Hải

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Bộ dữ liệu được chia theo trục thời gian thay vì chia ngẫu nhiên, đồng thời có vùng đệm giữa các phần. Cách chia này phù hợp với dữ liệu từ camera cố định vì các frame liên tiếp có thể rất giống nhau và cùng một chiếc xe có thể xuất hiện trong nhiều frame. Nếu chia ngẫu nhiên, thông tin gần như trùng lặp có thể xuất hiện ở cả train và test, làm kết quả test có nguy cơ cao hơn khả năng tổng quát thực tế.

Tập kiểm thử gồm **20 ảnh với 403 box tham chiếu**. Khi đánh giá, **14 box có kích thước dưới 16 px được bỏ qua**; ngưỡng IoU là **0.5**, còn Precision, Recall và F1 được tính tại **confidence = 0.25**.

Do test set nhỏ và một phần box rất nhỏ bị loại khỏi phép chấm, các metric cần được xem như kết quả trên bộ test hiện tại, không nên diễn giải như một ước lượng tuyệt đối cho mọi điều kiện triển khai.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Vòng 0 sử dụng **YOLOv8n cold start từ COCO với các lớp car + bus + truck**, chưa sử dụng ảnh hoặc box được gán nhãn của bài toán này.

Kết quả trên test set:

| Metric | Vòng 0 |
|---|---:|
| Train images | 0 |
| Train boxes | 0 |
| AP50 | **0.771** |
| P@0.25 | **0.925** |
| R@0.25 | **0.489** |
| F1 | **0.640** |
| Recall small | **0.182** |
| Recall medium | **0.547** |
| Recall large | **0.561** |

Kết quả cho thấy Recall của xe nhỏ thấp hơn rõ rệt so với xe vừa và xe lớn. Vì vậy các trường hợp xe ở xa, kích thước rất nhỏ hoặc chỉ còn phần đèn là những trường hợp cần chú ý khi rà soát nhãn.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Chiến lược selection kết hợp ba thành phần:

```text
score = W_U · U + W_A · A + W_D · D
```

Trong đó:

- **U**: mức độ bất định của mô hình.
- **A**: mức độ cần xem xét thêm dựa trên các prediction/box còn lưỡng lự.
- **D**: mức độ đa dạng theo thời gian so với các ảnh đã chọn.
- Trọng số sử dụng: **W_U = 0.5, W_A = 0.3, W_D = 0.2**.

Ngoài điểm score, `MIN_GAP_S` được dùng để tránh chọn nhiều frame quá gần nhau về thời gian. Điều này quan trọng với camera cố định vì hai frame cách nhau rất ít có thể chứa gần như cùng một cảnh. Nếu chọn cả hai, chi phí rà nhãn tăng nhưng lượng thông tin mới có thể không tương ứng.

Trong vòng selection, các frame như `frame_0182.jpg`, `frame_0326.jpg` và `frame_0099.jpg` được dùng làm các ví dụ về những ảnh được ưu tiên để con người kiểm tra. Một frame có score cao nhưng không nhất thiết phải được chọn nếu nó quá gần một frame khác đã được chọn. Ví dụ `frame_0372.jpg` ở gần `frame_0369.jpg` về thời gian, nên việc bỏ qua một trong hai giúp ngân sách rà nhãn được phân bố sang các thời điểm khác.

Điểm selection chỉ dùng để **ưu tiên ảnh cần con người kiểm tra**. Score cao không có nghĩa chắc chắn rằng fine-tune trên ảnh đó sẽ cải thiện model. Hiệu quả của một vòng active learning phải được kiểm tra bằng cách train/fine-tune và đánh giá lại trên cùng một test set độc lập.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

### Vòng 0 — cold start

Vòng 0 không có dữ liệu train của bài toán:

- **Train images:** 0
- **Train boxes:** 0
- **AP50:** 0.771
- **P@0.25:** 0.925
- **R@0.25:** 0.489
- **F1:** 0.640
- **R small / medium / large:** 0.182 / 0.547 / 0.561

### Vòng 1 — fine-tune sau khi rà nhãn

Vòng 1 sử dụng **12 ảnh train với 275 box** sau khi con người rà và sửa pre-label bằng CVAT.

Thống kê thao tác sửa nhãn gồm:

- **135 accepted**
- **18 edited**
- **16 deleted**
- **122 added**

Tổng số box sau rà nhãn là **275**. Đáng chú ý, số box được thêm mới (122) lớn hơn nhiều số box bị xóa (16), cho thấy pre-label ban đầu còn bỏ sót một lượng đáng kể đối tượng cần gán nhãn.

### So sánh kết quả các vòng

| Vòng | Model | Ảnh train | Box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 | YOLOv8n cold start (COCO car+bus+truck) | 0 | 0 | **0.771** | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | YOLOv8n fine-tune vòng 1 | 12 | 275 | **0.663** | **-0.108** | 0.989 | 0.216 | 0.354 | 0.000 | 0.199 | 0.683 |

So với cold start:

- AP50 giảm từ **0.771 xuống 0.663**, tức giảm khoảng **0.108 điểm**.
- Precision tăng từ **0.925 lên 0.989**.
- Recall giảm từ **0.489 xuống 0.216**.
- F1 giảm từ **0.640 xuống 0.354**.
- Recall small giảm từ **0.182 xuống 0.000**.
- Recall medium giảm từ **0.547 xuống 0.199**.
- Recall large tăng từ **0.561 lên 0.683**.

Như vậy, vòng 1 không cho thấy sự cải thiện trên AP50 hoặc Recall của toàn bộ test set. Precision tăng mạnh nhưng đi kèm với Recall giảm mạnh, đặc biệt ở nhóm xe nhỏ và xe vừa. Điều này cho thấy model sau fine-tune trở nên thận trọng hơn trong việc phát hiện, nhưng đồng thời bỏ sót nhiều đối tượng hơn.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Sau vòng 1, kết quả test chưa cho thấy cải thiện so với cold start:

> **AP50: 0.771 → 0.663 (Δ = -0.108)**

Recall cũng giảm từ **0.489 xuống 0.216**, trong đó Recall của xe nhỏ giảm xuống **0.000**. Vì vậy, thay vì tiếp tục chọn và gán nhãn ngay, bước hợp lý tiếp theo là kiểm tra lại dữ liệu và nguyên nhân của sự suy giảm trước khi thực hiện vòng fine-tune tiếp theo.

Các nhóm cần ưu tiên kiểm tra gồm:

1. **Xe rất nhỏ/ở xa:** Recall small của vòng 1 bằng 0, cho thấy đây là nhóm cần được kiểm tra kỹ.
2. **Xe vừa:** Recall cũng giảm đáng kể từ 0.547 xuống 0.199.
3. **Xe bị cắt ở mép ảnh hoặc bị che:** đây là các trường hợp khó xác định chính xác ranh giới bounding box.
4. **Các box đã sửa `added/edited/deleted`:** cần kiểm tra tính nhất quán với guideline trước khi sử dụng thêm chúng để train.
5. **Test reference:** một số trường hợp khó nên được rà lại để đảm bảo reference box thực sự phù hợp với quy tắc gán nhãn.

Kết quả cũng có các giới hạn quan trọng:

- Test set chỉ có **20 ảnh và 403 box tham chiếu**.
- **14 box dưới 16 px** bị bỏ qua trong phép chấm.
- Camera cố định khiến các frame gần nhau có thể rất giống nhau, vì vậy selection cần kiểm soát khoảng cách thời gian.
- Kết quả AP50/Recall hiện tại chỉ phản ánh bộ test và cách chấm được sử dụng trong bài; không nên suy rộng trực tiếp thành chất lượng triển khai thực tế.

Nếu thực hiện vòng tiếp theo, quy trình nên là:

```text
Kiểm tra các box đã sửa
        ↓
Kiểm tra các trường hợp small / bị cắt / bị che
        ↓
Rà lại một số reference box khó
        ↓
Chọn thêm ảnh có uncertainty cao nhưng đủ đa dạng theo thời gian
        ↓
Gán nhãn và kiểm tra chất lượng
        ↓
Fine-tune
        ↓
Đánh giá lại trên cùng test set
```

Chỉ nên tiếp tục mở rộng dữ liệu sau khi xác định được nguyên nhân khiến Recall và AP50 giảm ở vòng 1. Điều này giúp tránh việc tăng chi phí gán nhãn nhưng chưa có bằng chứng rằng dữ liệu bổ sung đang cải thiện model.

