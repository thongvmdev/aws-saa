# Chiến lược học SAA và áp dụng vào p-resume

Học khóa SAA-C03 trong 5 tháng (01/12/2026 – cuối 04/2027, thi khoảng 05/2027) và đưa dần p-resume lên AWS staging song song.
Kế hoạch chung (vị trí của SAA, tài liệu, học cùng AI, luyện đề): [ke-hoach-hoc-saa.md](ke-hoach-hoc-saa.md). Checklist bài học: [udemy-saa-c03-checklist.md](udemy-saa-c03-checklist.md).

## 1. Chiến lược áp dụng

p-resume (`P-project/p-resume`) là dự án dùng để làm lab trong đợt học này. Lý do chọn: stack đủ các mảnh mà đề SAA hỏi nhiều (container, Postgres, Redis, queue + worker, lưu file, SSE, gọi LLM), và đây là app làm CV, nên kiến trúc AWS của nó vừa là bằng chứng kỹ năng vừa là nội dung đưa thẳng vào CV.

**Ba nguyên tắc:**

1. **Không chuyển production.** p-resume vẫn chạy trên home server. Trên AWS chỉ dựng một bản **staging bằng CDK**: cuối tuần dựng lên làm lab, xong thì `cdk destroy`.
2. **Học xong là làm ngay.** Mỗi section, ghi 2–3 dòng xem phần nào của p-resume dùng đến dịch vụ đó. Có dịch vụ tương ứng thì làm lab trên p-resume, bỏ qua lab mẫu của khóa. Dịch vụ nào p-resume không dùng (Snow, Storage Gateway, Direct Connect…) thì chỉ xem video và làm quiz.
3. **Mỗi mốc để lại bằng chứng:** một ADR trong `docs/adr` của p-resume, có sơ đồ, lý do chọn, chi phí ước tính mỗi tháng và một con số đo được.

**Kiến trúc mục tiêu:**

| Thành phần hiện tại | Trên AWS | Ghi chú |
|---|---|---|
| Traefik + Cloudflare Tunnel | ALB + ACM + Route 53 | Idle timeout mặc định 60 giây: chỉnh cho SSE (AI stream, job events) và gửi heartbeat |
| `resume-web` (SSR), `resume-api` (NestJS) | ECS Fargate | Image chuyển từ GHCR sang ECR |
| `worker` (queue consumers) | ECS service riêng | Tự scale theo độ dài queue. Thử SQS cho một queue sau `QueueModule`, không chuyển hết |
| Postgres 17 + pgvector | RDS Postgres 17 | RDS hỗ trợ pgvector. Single-AZ cho staging, chỉ bật Multi-AZ trong buổi lab failover |
| `redis` (queue, pub/sub, noeviction) và `redis-cache` (LRU) | 2 ElastiCache | Giữ tách hai vai trò như hiện tại, vì eviction policy khác nhau |
| `STORAGE_DIR` (ảnh, PDF, thumbnail) | S3 + presigned URL | Lifecycle rule cho PDF/thumbnail render lại được; CloudFront cho trang chia sẻ CV công khai |
| SMTP (magic link) | SES | |
| Lịch crawl job (P19) | EventBridge Scheduler | |
| API key LLM theo từng node | Secrets Manager | Thêm Bedrock làm provider cho một node: chỉ cần đổi env |
| `backup.sh` (pg_dump + tar volume) | RDS PITR + AWS Backup | Diễn tập khôi phục ở region khác, đo RTO/RPO thật |
| GitHub Actions + self-hosted runner | GitHub OIDC → IAM role | Bỏ access key dài hạn |
| Log container | CloudWatch Logs + alarm | WAF rate limit cho các route AI |

**Những thứ không nên làm:**

- Đừng thay Better Auth bằng Cognito. Cognito chỉ cần biết đủ để thi.
- Đừng chuyển hết queue sang SQS. Thử một queue để hiểu khác biệt với Redis là đủ.
- Đừng mở Multi-AZ hay NAT Gateway mặc định. Chỉ bật trong buổi lab cần đến rồi tắt ngay.

## 2. Lộ trình 5 tháng (12/2026 – 05/2027)

Học theo đúng thứ tự section của khóa. Mỗi tháng khoảng 5–6 giờ video: tối trong tuần xem 20–30 phút và học cùng AI, cuối tuần làm lab 2–3 giờ trên p-resume (theo kiến trúc mục tiêu ở trên). Làm quiz cuối mỗi section ngay khi xem xong.

| Tháng | Section (giờ video) | Phần p-resume đưa lên AWS |
|---|---|---|
| **12/2026** | S1–8: IAM, EC2, EBS/EFS, ELB/ASG (6,2h) | Tạo tài khoản theo checklist an toàn ở mục 4 của [kế hoạch chung](ke-hoach-hoc-saa.md), bật **Budgets** trước tiên. Chạy cả stack compose trên **một EC2**, đặt sau **ALB + ACM**. Kiểm tra SSE qua ALB (idle timeout 60 giây) |
| **01/2027** | S9–14: RDS/ElastiCache, Route 53, S3 (5,6h) | Postgres sang **RDS Postgres 17** (pgvector), 2 Redis sang **ElastiCache**, `STORAGE_DIR` sang **S3** (presigned URL, lifecycle rule), subdomain trên **Route 53** |
| **02/2027** (có Tết, nhẹ tay) | S15–20: CloudFront, SQS/SNS, ECS, Serverless (5,1h) | api, web, worker lên **ECS Fargate** + **ECR**; worker tự scale theo queue; thử **SQS** cho một queue; **CloudFront** cho trang chia sẻ CV; **EventBridge Scheduler** cho lịch crawl job; **SES** cho magic link |
| **03/2027** | S21–26: Database, Analytics, ML, Monitoring, IAM nâng cao, Security (5,2h) | **CloudWatch Logs** + alarm; API key LLM vào **Secrets Manager**; GitHub Actions dùng **OIDC role**; **WAF rate limit** cho route AI; thêm **Bedrock** làm provider cho một node. **Bắt đầu làm đề Tutorials Dojo theo mảng** |
| **04/2027** | S27–33: VPC, DR, kiến trúc, ôn thi (5,3h) | Dựng lại VPC: private subnet, so sánh chi phí VPC endpoint với NAT. `backup.sh` sang **RDS PITR + AWS Backup**, diễn tập khôi phục ở region khác và đo RTO/RPO. Viết ADR **"VPS hay AWS cho p-resume"** kèm bảng chi phí |
| **05/2027** | Thi | Làm đề trọn bộ theo thời gian thật, ôn mảng yếu theo sổ tay, rồi đi thi |

Phần VPC (S27) nằm cuối khóa, nên 4 tháng đầu cứ dùng default VPC, đến tháng 4 dựng lại cho đúng. Làm tạm rồi làm lại như vậy giúp nhớ lâu hơn. Nếu phần mạng cơ bản còn mơ hồ, xem trước các bài lý thuyết của S27 ngay trong tháng 12.
