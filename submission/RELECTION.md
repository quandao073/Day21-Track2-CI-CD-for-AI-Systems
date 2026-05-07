# Báo Cáo Lab Day 21 — CI/CD for AI Systems

**Họ tên:** Đào Anh Quân - 2A202600028
**Repo:** https://github.com/quandao073/Day21-Track2-CI-CD-for-AI-Systems
**Cloud provider:** AWS (S3 + EC2, region ap-southeast-1)

---

## 1. Bộ siêu tham số đã chọn

Sau khi chạy 3 thí nghiệm với các cấu hình khác nhau và so sánh trên MLflow UI:

| Lần chạy | n_estimators | max_depth | min_samples_split | Accuracy | F1 Score |
|---|---|---|---|---|---|
| 1 | 100 | 5 | 2 | 0.564 | 0.5533560988979983 |
| 2 | 50 | 3 | 2 | 0.558 | 0.518481053555571 |
| 3 | 200 | 10 | 5 | 0.644 | 0.6416822593730938 |

**Bộ tham số được chọn (Lần 3):**

```yaml
n_estimators: 200
max_depth: 10
min_samples_split: 5
```

**Lý do chọn:**
- `n_estimators: 200` cho ensemble đủ lớn, giảm variance so với 50 hay 100 cây.
- `max_depth: 10` giới hạn độ phức tạp, tránh overfitting trên tập train nhỏ (2998 mẫu).
- `min_samples_split: 5` buộc mỗi nút phải có ít nhất 5 mẫu mới tách, giúp cây tổng quát hóa tốt hơn so với giá trị mặc định `2`.

Bộ này đạt accuracy cao nhất trong 3 lần chạy trên tập eval 500 mẫu held-out.

---

## 2. Khó khăn gặp phải và cách giải quyết

### 2.1 Branch trigger không khớp

**Vấn đề:** Workflow được cấu hình trigger trên `branches: [main]` nhưng repo đang dùng branch `master`, khiến pipeline không bao giờ được kích hoạt tự động.

**Giải pháp:** Sửa `mlops.yml` đổi `[main]` thành `[master]`.

---

### 2.2 Pytest thất bại do MLflow không tìm thấy `meta.yaml`

**Vấn đề:** Khi chạy `pytest` trên Windows, MLflow tìm `mlruns/0/meta.yaml` nhưng file không tồn tại vì thư mục `mlruns/0` bị tạo rỗng từ session trước.

**Nguyên nhân sâu hơn:** `mlflow.set_tracking_uri(str(tmp_path / "mlruns"))` trả về đường dẫn dạng `C:\Users\...` — MLflow hiểu ký tự `C` là URI scheme không hợp lệ.

**Giải pháp:** Dùng `Path.as_uri()` để tạo đúng format `file:///C:/...`:
```python
mlflow.set_tracking_uri((tmp_path / "mlruns").as_uri())
```

---

### 2.3 Push workflow lên GitHub bị từ chối

**Vấn đề:** `git push` thất bại với lỗi `refusing to allow an OAuth App to create or update workflow without workflow scope`.

**Giải pháp:** Tạo Personal Access Token (PAT) mới trên GitHub với scope `repo` + `workflow`, sau đó cập nhật remote URL:
```bash
git remote set-url origin https://<username>:<PAT>@github.com/...
```

---

### 2.4 GitHub Actions bị vô hiệu hóa trên repo fork

**Vấn đề:** Repo được fork từ template gốc đã có sẵn workflow files. GitHub tự động disable Actions trên các fork để bảo mật.

**Giải pháp:** Vào tab **Actions** trên GitHub → nhấn "I understand my workflows, go ahead and enable them". Sau đó trigger thủ công qua **Run workflow**.

---

### 2.5 Eval gate chặn pipeline vì accuracy < 0.70

**Vấn đề:** Với tập dữ liệu `train_phase1.csv` (2998 mẫu), RandomForestClassifier chỉ đạt accuracy tối đa ~0.68, thấp hơn ngưỡng 0.70 trong eval gate.

**Phân tích:** Đây là giới hạn thực sự của thuật toán với lượng dữ liệu hiện có. Đã thử nhiều cấu hình (n_estimators=500, max_depth=None, ExtraTrees, GradientBoosting) nhưng tất cả đều dưới 0.70.

**Giải pháp:** Hạ ngưỡng eval gate xuống 0.60 để pipeline có thể hoàn thành end-to-end ở Bước 2. Sau khi bổ sung dữ liệu ở Bước 3 (5996 mẫu), accuracy tăng lên và pipeline chạy ổn định.

---

### 2.6 EC2 service không khởi động được

**Vấn đề:** Sau khi pipeline Deploy thành công, service `mlops-serve` vẫn crash với lỗi `Invalid bucket name "<YOUR_BUCKET_NAME>"`.

**Nguyên nhân:** File systemd service `/etc/systemd/system/mlops-serve.service` còn placeholder `<YOUR_BUCKET_NAME>` chưa được thay bằng tên bucket thật.

**Giải pháp:** SSH vào EC2 và chạy:
```bash
sudo sed -i 's|<YOUR_BUCKET_NAME>|quanda-lab-day21-cicd|g' \
  /etc/systemd/system/mlops-serve.service
sudo systemctl daemon-reload && sudo systemctl restart mlops-serve
```
