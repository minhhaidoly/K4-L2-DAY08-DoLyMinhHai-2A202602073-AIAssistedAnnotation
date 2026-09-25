# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg

Số xe nhìn thấy bằng mắt: 21

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: 
1. Xe ở sát mép phải phía dưới — chỉ nhìn thấy một phần đầu xe/đèn pha, bị cắt bởi mép ảnh -> dễ bị bỏ sót hoặc bounding box bị thiếu.
2. Xe nhỏ ở vùng phía trên–giữa ảnh— xe rất nhỏ, tối, chỉ nổi bật bởi hai đèn hậu màu đỏ -> dễ bị AI bỏ sót hoặc gộp nhầm với xe ngay phía dưới.
Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
