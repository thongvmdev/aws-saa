# Kế hoạch học AWS Solutions Architect – Associate (SAA)

*Cập nhật 18/11/2026. Theo trang Exam Guides chính thức của AWS, đề hiện tại là **SAA-C03** và chưa có thông báo đề thay thế.*

## 1. Vị trí của SAA trong lộ trình

SAA là mũi nhọn phụ, song song với ưu tiên chính (Python, eval, AI Reliability). SAA lấp lỗ hổng "chưa có kinh nghiệm cloud". Hiện toàn bộ hạ tầng của bạn chạy trên VPS tự quản: CI/CD tự dựng, MySQL, Postgres, Redis, Weaviate, nginx, và bạn tự trực sự cố. Như vậy bạn đã có kinh nghiệm vận hành thật. Thứ còn thiếu là **ngôn ngữ thiết kế cloud**.

- **Không học:** SysOps/CloudOps, DevOps Pro, DVA. Stack của bạn là container, không thiên về serverless.
- **Chưa học:** SAP (Professional). Khóa SAP yêu cầu có SAA trước. Đề SAP đang chuyển từ C02 sang C03 (ngày thi cuối của C02 là 16/11/2026), trong khi khóa Udemy vẫn ghi là C02.
- **Sau SAA:** cân nhắc **AWS Generative AI Developer – Professional (AIP-C01)**, khớp với hướng AI Engineer hơn SAP.
- **Không học trước SAA: Kubernetes.** Đề SAA chỉ hỏi EKS ở mức biết nó là gì và khi nào chọn nó thay vì ECS, không cần viết manifest hay vận hành cluster. Lab dùng ECS Fargate, đơn giản hơn và đủ cho Family Finance. K8s là một mảng lớn riêng, học trước dễ sa vào chi tiết vận hành cluster trước khi có bức tranh kiến trúc. K8s để sau khi thi (xem mục 11).

**Kiến thức nền cần có trước:** mạng cơ bản (IP, CIDR, subnet, DNS, HTTP/TLS, NAT). Nếu phần này còn mơ hồ, dành vài buổi ôn trước khi bắt đầu, vì tuần 2 (VPC) sẽ dễ hơn rất nhiều. Docker là nền còn lại, và bạn đã có.

**Thứ tự tổng thể:** ôn mạng cơ bản (nếu cần) → SAA → K8s trên home server (k3s) → EKS nếu muốn.

**Thời gian:** 8 tuần, 8–10 giờ mỗi tuần, tức khoảng 70 giờ. Cần chấp nhận một điều: trong 8 tuần này SAA sẽ chiếm phần lớn thời gian, phần AI chỉ còn khoảng một buổi cuối tuần. Nếu muốn giữ nhịp cho AI, hãy giãn ra 10 tuần bằng cách tách tuần 3–4 và tuần 7 thành hai tuần mỗi phần.

## 2. Kiến thức SAA và giá trị cho sự nghiệp

**Bốn mảng của đề:** bảo mật, khả năng chịu lỗi, hiệu năng, tối ưu chi phí.

- **Mạng:** VPC, subnet public/private, route table, security group, VPN, Direct Connect, Transit Gateway.
- **Bảo mật:** IAM theo nguyên tắc quyền tối thiểu, Organizations/SCP, KMS, Secrets Manager.
- **Compute và lưu trữ:** chọn giữa EC2, container và serverless; chọn database và storage theo kiểu truy cập.
- **Tách rời hệ thống:** SQS, SNS, EventBridge.
- **Chịu lỗi:** Multi-AZ, RPO/RTO, các chiến lược DR từ backup-restore đến active-active.
- **Hiệu năng và chi phí:** cache, CDN, autoscaling, Savings Plans, Spot, đọc hóa đơn.

**Giá trị cho sự nghiệp:**

- Đổi cách nhà tuyển dụng nhìn hồ sơ: từ "frontend 7 năm" sang "làm được cả backend lẫn hạ tầng".
- Có ngôn ngữ chung để trả lời câu hỏi system design kiểu "đưa hệ thống agent lên production cho nhiều người dùng".
- Hỗ trợ trực tiếp mảng AI Reliability: chịu lỗi khi provider sập, kiểm soát chi phí, cách ly dữ liệu nhạy cảm.
- Dùng được ngay ở công việc hiện tại: tham gia thảo luận kiến trúc trong team, từ đó nhận thêm việc backend hoặc AI.

**Giới hạn:** chứng chỉ chỉ giúp qua vòng lọc CV. Để thắng ở vòng phỏng vấn, bạn cần bằng chứng: kiến trúc Family Finance dựng bằng CDK, bài so sánh VPS và AWS, và tài liệu thiết kế cho tải lớn.

## 3. Áp dụng vào Family Finance

**Kiến trúc mục tiêu:**

| Thành phần hiện tại | Trên AWS | Ghi chú |
|---|---|---|
| nginx | ALB + ACM | Idle timeout mặc định 60 giây, phải chỉnh cho SSE và gửi heartbeat. Nếu đặt CloudFront phía trước, kiểm tra thêm origin timeout |
| Web, API (NestJS) | ECS Fargate, private subnet | Chuyển image từ GHCR sang ECR, kéo qua VPC endpoint để không tốn phí NAT |
| **LangGraph server** | ECS service riêng | Stream chat kéo dài tới 30 phút, nên timeout phải khớp với service token |
| MySQL (ledger) | RDS MySQL Multi-AZ | Dữ liệu tiền, không chấp nhận mất |
| MySQL cho SQL skill | **Read replica** | Cách ly các truy vấn do model sinh ra. Xem mục đọc dữ liệu cũ bên dưới |
| Postgres (checkpoint) | RDS Postgres Single-AZ + backup | Chấp nhận RPO cao hơn. Mất checkpoint thì các phiên đang dừng ở bước human-in-the-loop sẽ hỏng, nên vẫn cần backup |
| Redis | ElastiCache (Redis/Valkey ≥ 7) | Cần bản 7 trở lên cho `EXPIRE NX/GT` và `EVAL_RO` |
| **Weaviate** | Cần quyết định (ADR) | Đây là dịch vụ có trạng thái, không hợp với Fargate. Lựa chọn: EC2 + EBS, Weaviate Cloud, OpenSearch hoặc pgvector |
| File PDF import | S3 + presigned URL | |
| Scheduled job | EventBridge Scheduler | |
| Secret, token | Secrets Manager | |

**Những điểm đáng giá để viết thành tài liệu thiết kế:**

- **Đọc phải dữ liệu cũ (read-your-writes):** sau khi người dùng duyệt một lệnh ghi ở bước human-in-the-loop, agent đọc từ replica có thể nhận dữ liệu cũ. Cách xử lý: đọc từ primary trong một khoảng thời gian sau khi ghi, hoặc chấp nhận dữ liệu cũ một chút và ghi rõ quyết định.
- **Phân loại dữ liệu theo độ quan trọng** để quyết định thành phần nào cần Multi-AZ.
- **Chi phí NAT:** VPC endpoint chỉ giảm phí cho lưu lượng đến dịch vụ AWS (S3, ECR, Secrets Manager, CloudWatch Logs). Lưu lượng gọi OpenAI, Groq hay Cloudflare **vẫn đi qua NAT**.

**Các bẫy cần nhớ:**

- **Redis cluster mode:** phần invalidation theo tag set động đến nhiều key và sẽ gặp lỗi CROSSSLOT. Dùng hash tag `{householdId}` để các key cùng household nằm chung một slot. Rate limiter dùng một ZSET cho mỗi user nên không bị ảnh hưởng.
- **SSE dài qua ALB** cần chỉnh timeout và gửi heartbeat.
- **Region:** Singapore là region gần nhất. Dữ liệu tài chính hộ gia đình có thể chịu quy định bảo vệ dữ liệu cá nhân của Việt Nam, nên cần kiểm tra yêu cầu về vị trí lưu trữ.

**Chi phí:** kiến trúc đầy đủ có thể tốn vài trăm USD mỗi tháng, chủ yếu do phí nền cố định (NAT, ALB, RDS Multi-AZ, ElastiCache). Vì vậy lựa chọn VPS hiện tại là hợp lý. Biết phân tích khi nào cloud đáng tiền chính là tư duy SAA muốn rèn.

## 4. Công cụ

- **CDK (TypeScript):** đặt thành một package trong Turborepo. Mỗi lần lab thì `cdk deploy`, xong là `cdk destroy`.
- **Terraform:** viết lại sau khi thi để bổ sung vào CV.
- **k6:** mock LLM để không đốt token, giới hạn autoscaling, chạy k6 trên EC2 cùng region, bật Budgets. Một buổi test thường chỉ tốn vài USD.

**Checklist an toàn tài khoản lab:**

- [ ] Bật MFA cho root, đăng nhập bằng IAM Identity Center, không tạo access key dài hạn.
- [ ] Bật Budgets và Cost Anomaly Detection. Nhớ rằng Budgets chỉ cảnh báo, không tự tắt tài nguyên.
- [ ] Đặt `RemovalPolicy.DESTROY` cho staging. RDS mặc định tạo snapshot khi xóa, log group mặc định được giữ lại.
- [ ] `cdk destroy` không xóa bucket và repo do bootstrap tạo ra.
- [ ] Mỗi IPv4 công khai và mỗi NAT Gateway đều tính tiền theo giờ. Staging chỉ dùng 1 NAT.
- [ ] Kiểm tra điều kiện Free Tier hiện tại. Từ giữa 2025, tài khoản mới nhận credit thay cho Free Tier 12 tháng.

## 5. SAA và tư duy thiết kế cho tải lớn

SAA cho bạn các mảnh ghép (scale ngang, hàng đợi, cache, Multi-AZ) và phản xạ hỏi "chỗ này chết thì sao". SAA không dạy ước lượng năng lực, sharding, mô hình nhất quán hay back-pressure. Bổ sung bằng *Designing Data-Intensive Applications* và *System Design Interview*.

Bài tập chính: với Family Finance, ở mức tải 10x, 100x, 1000x thì cái gì vỡ trước? Nhiều khả năng LLM vỡ trước database, và đó đúng là mảng AI Reliability.

## 6. Mô hình học ba lớp

1. **Home server:** chỉ dựng những thứ bạn **chưa từng vận hành**, ví dụ MySQL replication. Bỏ lab nginx load balancer và Redis vì bạn đã chạy chúng thật.
2. **LocalStack** (tùy chọn): tập viết CDK và code gọi AWS API miễn phí. Nó chỉ giả lập giao diện, không mô phỏng cơ chế.
3. **AWS staging thật:** dựng bằng CDK vài giờ để thấy hành vi thật, IAM và chi phí, rồi destroy. RDS Multi-AZ mất 15–20 phút mỗi lần dựng hoặc xóa, nên gom các lab RDS vào một buổi.

## 7. Tài liệu (đã chốt)

| Loại | Tài liệu | Ghi chú |
|---|---|---|
| Khóa chính | **Ultimate AWS Certified Solutions Architect Associate 2026** (Stephane Maarek, Udemy) | 27h25, 399 bài, nhắm SAA-C03, có slide PDF và 1 đề đầy đủ. Không mua thêm khóa video nào khác |
| Đề luyện | **Tutorials Dojo** (Jon Bonso) | Thước đo chính để biết đã sẵn sàng hay chưa |
| Đề chính thức | Official Practice Question Set trên **AWS Skill Builder** | Miễn phí |
| Đối chiếu | Exam Guide SAA-C03 và trang FAQ của từng dịch vụ | Để kiểm tra lại các con số và giới hạn |

Khóa Maarek giảng hướng tới đề thi, ít đi sâu vào cơ chế. Bạn bù phần cơ chế bằng lab và học cùng AI. Các con số trong video (giá, giới hạn) có thể đã cũ, nên luôn kiểm tra lại trên tài liệu của AWS.

Link:

- Khóa học: https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/
- Exam Guides: https://docs.aws.amazon.com/aws-certification/latest/examguides/aws-certification-exam-guides.html
- Trang chứng chỉ: https://aws.amazon.com/certification/certified-solutions-architect-associate/

## 8. Học từng phần cùng AI

Dùng AI để **bổ sung** cho khóa học, không thay thế. AI dễ sai ở chi tiết, không biết đề hay ra gì, và câu hỏi nó tự ra thường dễ hơn đề thật.

**Quy trình cho mỗi phần của khóa:**

1. Xem video ở 1.25–1.5x để biết đề hay hỏi gì.
2. Đưa slide PDF của phần đó cho AI và yêu cầu chỉ trả lời dựa trên slide cùng tài liệu AWS.
3. Hỏi sâu về cơ chế và áp vào Family Finance.
4. Bảo AI ra câu hỏi tình huống kiểu SAA và chấm nghiêm.
5. Tự viết lab bằng CDK, AI chỉ review.
6. Ghi câu sai vào sổ tay theo chủ đề.

**Thiết lập:** tạo một Claude Project riêng cho SAA, để sẵn slide, kế hoạch này, sổ tay câu sai và mô tả kiến trúc Family Finance. Slide chỉ dùng cho việc học cá nhân, không chia sẻ ra ngoài.

**Prompt mẫu:**

- *Cơ chế:* "Đây là slide phần [X]. Giải thích cơ chế bên dưới, không chỉ định nghĩa. Chỗ nào slide không nói, hãy ghi rõ là kiến thức ngoài slide."
- *Áp dụng:* "Hệ thống của tôi là [mô tả]. Nếu chuyển lên [dịch vụ] thì cái gì vỡ, có những phương án nào và đánh đổi gì?"
- *Luyện đề:* "Ra 5 câu tình huống kiểu SAA-C03, mỗi câu có 2 đáp án đều chạy được và chỉ phân biệt bằng từ khóa như 'chi phí thấp nhất' hoặc 'ít vận hành nhất'. Chưa đưa đáp án. Sau khi tôi trả lời, chấm nghiêm và giải thích vì sao 3 đáp án còn lại sai."
- *Tự kiểm tra:* "Tôi sẽ giải thích lại [X]. Chỉ ra chỗ tôi hiểu sai hoặc còn thiếu. Đừng khen."

## 9. Lộ trình 8 tuần

Tối trong tuần (khoảng 1 giờ mỗi tối) xem video và học cùng AI. Cuối tuần làm lab. Các chủ đề ghi "lý thuyết" không có trong Family Finance nhưng đề hỏi nhiều, nên chỉ học bằng video và đề luyện, không làm lab.

| Tuần | Chủ đề | Lab cuối tuần |
|---|---|---|
| 1 | IAM, cấu trúc tài khoản, Budgets. *Lý thuyết:* Organizations/SCP, Control Tower | Tạo tài khoản theo checklist ở mục 4; cài CDK, bootstrap, deploy rồi destroy một stack nhỏ |
| 2 | VPC, Route 53. *Lý thuyết:* VPN, Direct Connect, Transit Gateway | CDK: VPC public/private, security group nối chuỗi, 1 NAT; thử VPC endpoint cho S3 và ECR |
| 3 | EC2, ASG, ELB, ECS/Fargate, ECR | NestJS API và LangGraph server trên Fargate sau ALB; thử idle timeout với SSE kèm heartbeat; đẩy image lên ECR |
| 4 | RDS, Aurora, read replica, DynamoDB | Home server: MySQL replication. AWS (gom một buổi): RDS kèm read replica, thử failover Multi-AZ. Viết ADR chọn chỗ chạy Weaviate và cách xử lý read-your-writes |
| 5 | S3, EBS/EFS, ElastiCache, CloudFront. *Lý thuyết:* Kinesis, Athena, Glue, Redshift, DMS, DataSync, Snow family | Upload qua presigned URL; chuyển sang ElastiCache với hash tag. **Bắt đầu làm đề Tutorials Dojo** |
| 6 | SQS, SNS, EventBridge, DR (RPO/RTO). *Lý thuyết:* Storage Gateway, AWS Backup | Scheduled job bằng EventBridge Scheduler; viết một trang chiến lược DR cho Family Finance |
| 7 | KMS, WAF, GuardDuty, CloudWatch, CloudTrail, tối ưu chi phí. *Lý thuyết:* các dịch vụ AI/ML có sẵn | Gộp thành staging hoàn chỉnh; load test bằng k6 với mock LLM; xem Cost Explorer |
| 8 | Luyện đề | Làm đề trọn bộ theo thời gian thật; ôn các mảng yếu theo sổ tay |

## 10. Luyện đề và đăng ký thi

- Bắt đầu làm đề từ tuần 5 để sớm thấy mảng nào đang yếu.
- Giá trị nằm ở phần **đọc giải thích**, kể cả với câu làm đúng. Với mỗi câu, hiểu vì sao ba đáp án còn lại sai.
- Đề SAA thường có hai đáp án đều chạy được. Điểm phân biệt nằm ở từ khóa như "chi phí thấp nhất", "ít vận hành nhất", "độ trễ thấp nhất".
- Nếu điểm Tutorials Dojo thấp hơn điểm khi luyện với AI, nghĩa là AI đang hỏi quá dễ.
- **Đăng ký thi** khi điểm đề luyện ổn định từ 80% trở lên qua vài bộ liên tiếp. Đề gồm 65 câu trong 130 phút.
- **Trước khi đặt lịch**, xin chế độ hỗ trợ ESL (+30 phút) cho người không dùng tiếng Anh bản ngữ.
- Kiểm tra lại phiên bản đề trên trang Exam Guides ngay trước khi đăng ký.
- Thi đậu sẽ nhận voucher giảm 50% cho lần thi tiếp theo.

## 11. Giữ đúng ưu tiên và việc sau khi thi

SAA không được chiếm hết thời gian của Python và eval. Hai mảng gặp nhau ở tuần 7, khi load test với mock LLM cũng chính là bài tập AI Reliability.

**Sau khi thi, đây là lúc kiến thức SAA biến thành giá trị trên CV:**

1. Viết bài **so sánh VPS và AWS** cho Family Finance (chi phí, độ phức tạp, khi nào nên chuyển) và đăng lên blog Hanu NUS.
2. Viết **tài liệu thiết kế cho tải lớn** (10x, 100x, 1000x) để mang đi phỏng vấn.
3. **Thêm Bedrock** vào lớp multi-provider sẵn có: gọi model qua PrivateLink, xác thực bằng IAM thay cho API key, xử lý quota và throttling, thử cross-region inference. Đây là dự án vừa thuộc SAA vừa thuộc AI Reliability.
4. Viết lại phần hạ tầng bằng **Terraform**.
5. **Học Kubernetes trên home server:** dựng cluster nhẹ bằng k3s, deploy lại Family Finance lên đó, rồi so với bản ECS trên AWS. Sau SAA, K8s sẽ dễ hiểu hơn vì bạn đã nắm load balancing, autoscaling, service discovery và networking; K8s chỉ là cách khác để làm những việc đó. Đáng học vì nhiều công ty vừa và lớn ở Việt Nam chạy K8s, và mảng AI infra (serving model, quản lý GPU) cũng xoay quanh K8s. Sau đó học EKS nếu muốn.
6. Cân nhắc mục tiêu tiếp theo: **AIP-C01** (Generative AI Developer – Professional).
