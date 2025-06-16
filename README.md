# Pipeline Jenkins cho CI/CD Ứng dụng Microservices

Repository này chứa pipeline Jenkins để tự động hoá quy trình build, test, kiểm tra chất lượng mã nguồn và triển khai ứng dụng microservices bằng Docker và Kubernetes (EKS).

---

## Công nghệ sử dụng

- Jenkins (Pipeline dạng Declarative)
- Docker
- Kubernetes (Amazon EKS)
- AWS Secrets Manager
- SonarQube (phân tích mã nguồn)
- Trivy / Snyk (kiểm tra lỗ hổng bảo mật)
- GitHub hoặc AWS CodeCommit
- Công cụ CLI: `kubectl`, `aws`, `docker`

---

## Cấu trúc Repository
```bash
├── task-service/
| ├── task/
├── user-service/
│ ├── auth/
│ └── user/
├── k8s/
│ └── auth-deployment.yaml
│ └── user-deployment.yaml
│ └── task-deployment.yaml
└── Jenkinsfile
```

---

## Cài đặt môi trường

### 1. Yêu cầu

- Cài đặt `aws-cli`, `docker`, `kubectl`
- Đã tạo EKS cluster và cấu hình `kubectl` để kết nối
- Máy Jenkins có các plugin:
  - Pipeline
  - Docker Pipeline
  - AWS Credentials
  - Kubernetes CLI
  - SonarQube Scanner
  - Snyk Security
- Các thông tin bí mật cần khai báo trong Jenkins:
  - `aws-key`: AWS Access Key ID + Secret
  - `docker`: Tài khoản Docker Hub
  - `sonar-token`: Token để kiểm tra chất lượng SonarQube
  - `MYSQL-DOTENV`: file `.env` dùng để tạo ConfigMap

---

## Cách sử dụng

### Cách 1: Chạy thủ công từ Jenkins

1. Truy cập Jenkins → New Item → Pipeline
2. Trỏ đến repository này
3. Bấm "Build with Parameters", nhập:
   - `AWS_REGION`: Vùng AWS (vd: `us-east-1`)
   - `REPO_DOCKER_USER`: Tên tài khoản Docker Hub
   - `TAG`: Tag Docker image (vd: `v1.0`)
   - `CLUSTER_NAME`: Tên EKS cluster

### Cách 2: Đẩy code lên Git

- Khi push lên nhánh `main`, pipeline sẽ tự động chạy khi đã cấu hình webhook

---

## Các bước trong Pipeline

| Giai đoạn             | Mô tả                                                                 |
|-----------------------|----------------------------------------------------------------------|
| Git Checkout          | Tải mã nguồn mới nhất từ repository                                 |
| Check Services        | Kiểm tra dịch vụ nào có sự thay đổi để chỉ build phần đó            |
| Setup ENV             | Khai báo biến môi trường, image name, path...                       |
| SonarQube Scan        | Phân tích mã nguồn với SonarQube                                    |
| Quality Gate          | Đợi SonarQube phản hồi kết quả phân tích                            |
| Docker Build          | Build Docker image và push lên Docker Hub                           |
| Scan Image            | Dùng Trivy và Snyk để quét bảo mật cho image                        |
| Remove Container      | Xoá container/image local để giải phóng dung lượng                  |
| Deploy to Kubernetes  | Tạo ConfigMap và triển khai YAML tương ứng lên EKS                  |

---

## Kiểm tra sau triển khai

Sau khi pipeline chạy thành công:

- Docker image đã được đẩy lên Docker Hub
- Deployment YAML đã được apply vào cluster EKS

