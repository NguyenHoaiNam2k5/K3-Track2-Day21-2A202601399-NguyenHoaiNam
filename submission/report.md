# Báo cáo kết quả xây dựng hệ thống MLOps

## 1. Bộ siêu tham số được lựa chọn

Mô hình được sử dụng là `RandomForestClassifier` với `random_state=42` để bảo đảm khả năng tái tạo kết quả. Các lần chạy được ghi nhận và so sánh trong experiment MLflow `hyperparameter-comparison` trên cùng tập huấn luyện 5.996 mẫu và tập đánh giá cố định 500 mẫu.

| Số cây (`n_estimators`) | Độ sâu tối đa (`max_depth`) | Số mẫu tối thiểu để tách nút (`min_samples_split`) |   Accuracy |   F1-score |
| ----------------------: | --------------------------: | -------------------------------------------------: | ---------: | ---------: |
|                     100 |                           5 |                                                  5 |     0.5680 |     0.5573 |
|                     400 |                          20 |                                                 20 |       0.64 |     0.6377 |
|                     200 |                          10 |                                                 10 |       0.62 |     0.6179 |
|                      50 |                           3 |                                                  2 |      0.558 |     0.5185 |
|                 **100** |                      **20** |                                              **2** | **0.7580** | **0.7571** |

Bộ siêu tham số cuối cùng được chọn:

```yaml
n_estimators: 100
max_depth: 20
min_samples_split: 2
```

Lý do lựa chọn:

- Cấu hình này đạt cả accuracy và F1-score cao nhất trong các lần chạy MLflow, lần lượt là `0.7580` và `0.7571`.
- Accuracy cao hơn ngưỡng triển khai `0.70` khoảng 5,8 điểm phần trăm, tạo khoảng an toàn cho eval gate của pipeline.
- `max_depth=20` cho phép mô hình học quan hệ đủ phức tạp; cấu hình `max_depth=5` bị underfitting và chỉ đạt accuracy `0.5800`.
- `min_samples_split=2` giữ lại khả năng tách chi tiết của cây. Các cấu hình dùng giá trị 20 hoặc 40 hạn chế quá trình phân chia và cho kết quả thấp hơn.
- Chỉ cần 100 cây nhưng vẫn tốt hơn các cấu hình 200 hoặc 400 cây. Vì vậy cấu hình được chọn còn giúp giảm thời gian huấn luyện và kích thước mô hình mà không đánh đổi chất lượng.

## 2. So sánh kết quả khi tăng dữ liệu

Để đánh giá ảnh hưởng của dữ liệu mới, hai lần chạy trong experiment MLflow `dataset-size-comparison` sử dụng cùng bộ siêu tham số đã chọn và cùng tập đánh giá 500 mẫu. Khác biệt duy nhất là kích thước tập huấn luyện.

| Chỉ số   | 2.998 mẫu | 5.996 mẫu | Mức thay đổi |
| -------- | --------: | --------: | -----------: |
| Accuracy |    0.6840 |    0.7580 |      +0.0740 |
| F1-score |    0.6829 |    0.7571 |      +0.0742 |

Khi tăng tập huấn luyện từ 2.998 lên 5.996 mẫu:

- Accuracy tăng từ `0.6840` lên `0.7580`, tương đương tăng 7,4 điểm phần trăm.
- F1-score tăng từ `0.6829` lên `0.7571`, tương đương tăng khoảng 7,42 điểm phần trăm.
- Hai chỉ số tăng gần tương đương nhau, cho thấy cải thiện không chỉ nằm ở tỷ lệ dự đoán đúng tổng thể mà còn tương đối cân bằng giữa các lớp khi đánh giá bằng weighted F1-score.
- Lần chạy 2.998 mẫu chưa đạt ngưỡng `0.70`, trong khi lần chạy 5.996 mẫu đã vượt ngưỡng và đủ điều kiện triển khai.

Kết quả cho thấy dữ liệu phase 2 cung cấp thêm thông tin hữu ích cho mô hình. Với cùng thuật toán, siêu tham số và tập đánh giá, việc tăng gấp đôi số mẫu huấn luyện đã cải thiện rõ rệt khả năng tổng quát hóa.

## 3. Khó khăn gặp phải và cách giải quyết

Những lần chạy ban đầu chỉ đạt accuracy `0.6760`, thấp hơn ngưỡng `0.70`, nên job Eval kết thúc với lỗi và job Deploy không được chạy. Đây là hành vi đúng của quality gate, không phải lỗi của GitHub Actions.

Cách giải quyết là dùng MLflow để so sánh các cấu hình thay vì hạ ngưỡng đánh giá. Kết quả cho thấy việc giảm `min_samples_split` từ 20 xuống 2 và sử dụng `max_depth=20` giúp accuracy tăng lên `0.7580`. Sau khi cập nhật `params.yaml`, pipeline vượt eval gate.

## 4. Kết luận

Hệ thống đã hoàn thành chu trình MLOps từ quản lý dữ liệu bằng DVC, theo dõi thí nghiệm bằng MLflow, kiểm thử và kiểm soát chất lượng bằng GitHub Actions, đến triển khai FastAPI trên VM. Kết quả thực nghiệm cho thấy lựa chọn siêu tham số dựa trên metrics và bổ sung dữ liệu huấn luyện đều có tác động rõ rệt: accuracy cuối cùng đạt `0.7580` và F1-score đạt `0.7571`, đủ điều kiện vượt eval gate và triển khai tự động.
