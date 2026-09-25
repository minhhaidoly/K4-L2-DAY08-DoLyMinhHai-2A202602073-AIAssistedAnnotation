# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:  

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà năm ảnh, tôi ưu tiên `frame_0182.jpg` (hạng 1, điểm 0.9591, 72.8s), `frame_0369.jpg` (hạng 2, điểm 0.9324, 147.6s), `frame_0380.jpg` (hạng 3, điểm 0.9170, 152.0s), `frame_0326.jpg` (hạng 4, điểm 0.9155, 130.4s) và `frame_0312.jpg` (hạng 7, điểm 0.9100, 124.8s). Năm ảnh này đều có điểm cao và mức độ model còn phân vân lớn, nên có nhiều khung cần kiểm tra. Tôi không chọn `frame_0331.jpg` (hạng 5, điểm 0.9154) vì nó rất gần `frame_0326.jpg` về thời điểm (132.4s so với 130.4s), nên hai ảnh có thể chứa cảnh gần giống nhau; dành ngân sách cho ảnh ở thời điểm khác sẽ tránh rà lại gần cùng một cảnh.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: `frame_0182.jpg` (hạng 1, điểm 0.9591, 18 box còn mơ hồ), `frame_0326.jpg` (hạng 4, điểm 0.9155, 15 box còn mơ hồ) và `frame_0099.jpg` (hạng 8, điểm 0.9063, 14 box còn mơ hồ). Cả ba đều nằm trong 12 ảnh được chọn để rà soát ở vòng 1; CSV cho thấy model có nhiều box chưa chắc chắn ở các ảnh này.


Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: Một frame có điểm cao nhưng không chọn là `frame_0372.jpg` (hạng 6, điểm 0.9101, 148.8s). Ảnh này không được ưu tiên vì rất gần `frame_0369.jpg` (147.6s), chỉ cách khoảng 1.2 giây, nên hai ảnh có khả năng gần trùng cảnh. Bỏ `frame_0372.jpg` giúp dành lượt rà soát cho các thời điểm khác thay vì kiểm tra hai ảnh gần nhau.


Điều phép chọn này chưa chứng minh về chất lượng mô hình: Phép chọn này chỉ cho biết những ảnh mà chiến lược selection đánh giá là đáng ưu tiên để con người rà soát dựa trên điểm uncertainty và các tiêu chí trong CSV. Điểm cao không chứng minh rằng sửa ảnh đó sẽ làm chất lượng mô hình tăng. Chất lượng mô hình phải được đánh giá bằng kết quả trên tập test độc lập sau khi huấn luyện lại.