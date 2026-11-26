## Section 3: Getting started with AWS

_0 / 3 | 13min_

- [x] 8. [AWS Cloud Overview - Regions & AZ](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/learn/lecture/26097978) · 8min
- [x] 9. [[Important] AWS Console UI Update](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/learn/lecture/47890667) · 1min
- [x] 10. [Tour of the AWS Console & Services in AWS](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/learn/lecture/26097980) · 4min

---

# Section 3, video 8: AWS Global Infrastructure

Bài này dạy **mô hình địa lý của AWS** và cách chọn nơi đặt ứng dụng. Phần đầu (lịch sử, doanh thu, khách hàng) chỉ là bối cảnh, không thi.

## 1. Bốn khái niệm cần nắm

| Khái niệm                      | Bản chất                                                                         | Slide nói gì                                                                                                                               |
| ------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Region**                     | Cụm data center ở một khu vực địa lý (us-east-1, ap-southeast-2...)              | Hầu hết service **gắn với một region**. Dùng service ở region khác thì coi như dùng mới, tài nguyên không tự sang                          |
| **Availability Zone (AZ)**     | Một hoặc nhiều data center rời nhau, có nguồn điện, mạng, kết nối dự phòng riêng | Mỗi region thường có 3 AZ (tối thiểu 3, tối đa 6). Các AZ **cô lập với thảm họa** nhưng nối nhau bằng mạng băng thông cao, độ trễ cực thấp |
| **Edge location / PoP**        | Điểm biên để đưa nội dung tới người dùng gần nhất                                | Hơn 400 PoP, 90 thành phố, 40 quốc gia. Sẽ học kỹ ở phần Global (CloudFront...)                                                            |
| **Global vs Regional service** | Phạm vi của service                                                              | Global: IAM, Route 53, CloudFront, WAF. Regional: EC2, Elastic Beanstalk, Lambda, Rekognition                                              |

Lưu ý: các con số trên là từ slide, hãy kiểm tra lại trên trang AWS Global Infrastructure vì chúng thay đổi theo thời gian.

## 2. Cơ chế bên dưới (vì sao thiết kế như vậy)

- **Region là ranh giới cô lập lớn nhất.** Dữ liệu và tài nguyên không tự rời region nếu bạn không cấu hình. Đây là điều giúp đáp ứng yêu cầu compliance (dữ liệu ở lại nước nào).
- **AZ là ranh giới lỗi (failure domain).** Slide nhấn mạnh: sự cố ở AZ-a được thiết kế để **không lan** sang AZ-b, AZ-c. Vì vậy tư duy HA của AWS là _rải tài nguyên qua nhiều AZ trong cùng một region_.
- **Vì sao chia hai tầng?** Cùng region thì độ trễ giữa AZ đủ thấp để replicate đồng bộ (ví dụ database Multi-AZ). Khác region thì xa hơn, thường replicate bất đồng bộ và tốn tiền hơn. Phần này là kiến thức ngoài slide, ta sẽ gặp lại ở RDS/S3.
- **Ngoài slide:** AWS không cho biết AZ có bao nhiêu data center, và tên AZ như `ap-southeast-2a` được ánh xạ khác nhau giữa các account. Nên kiểm tra lại trong docs AWS.

## 3. Bốn tiêu chí chọn region (hay ra thi)

1. **Compliance:** luật bắt dữ liệu ở trong nước.
2. **Latency:** gần người dùng.
3. **Service availability:** không phải region nào cũng có đủ service.
4. **Pricing:** giá khác nhau giữa các region.

Từ khóa của đề: compliance thường là tiêu chí **loại trừ cứng** (đứng trước các tiêu chí còn lại), tiếp theo mới tới latency, rồi giá.

## 4. So sánh với VPS bạn đang vận hành

- VPS của bạn gần như là **1 máy trong 1 data center**: tương đương "1 AZ, không dự phòng". Nếu nhà cung cấp mất điện hoặc mạng, toàn bộ Family Finance chết.
- Bạn chọn VPS ở đâu dựa trên latency và giá, nhưng ít khi cân nhắc compliance hay service availability. AWS buộc bạn nghĩ cả 4 yếu tố.
- Cái bạn **chưa có** ở VPS là khái niệm AZ: một "vùng lỗi" độc lập mà bạn có thể thuê thêm bằng vài cú click.

## 5. Áp vào Family Finance

Mình chưa đọc code trong thư mục P-project nên đây là phân tích ở mức kiến trúc, bạn sửa nếu sai:

- **Chọn region:** người dùng ở Việt Nam. Theo hiểu biết của mình (ngoài slide, cần kiểm tra lại), AWS chưa có region tại Việt Nam, nên ứng viên gần nhất thường là Singapore (`ap-southeast-1`). Mình chưa đo latency thực tế, bạn nên tự đo.
- **Compliance:** đây là dữ liệu tài chính gia đình. Cần kiểm tra quy định lưu trữ dữ liệu cá nhân và dữ liệu tại Việt Nam có áp dụng hay không. Mình không chắc về chi tiết pháp lý nên không kết luận.
- **Service availability:** stack của bạn có LangGraph và LLM. Nếu dùng Bedrock, model nào có ở region nào là câu hỏi thật, vì có thể ép bạn chọn region khác với region "gần nhất".
- **Cái gì vỡ nếu giữ 1 AZ:** mất AZ là mất cả MySQL, Postgres, Redis, Weaviate cùng lúc.
- **Phương án và đánh đổi:** (a) 1 AZ: rẻ nhất, vận hành đơn giản, chấp nhận downtime. (b) Multi-AZ cho database: chi phí tăng khoảng gấp đôi cho phần đó, đổi lại tự failover. (c) Multi-region: gần như không đáng với ứng dụng gia đình, chi phí và độ phức tạp lớn hơn rất nhiều so với lợi ích.

## 6. Mình hỏi lại bạn

1. Nếu Family Finance chạy trên **một EC2 duy nhất** ở một AZ, bạn nêu cho mình **2 kịch bản** khiến nó sập mà việc "chọn đúng region" không cứu được. Cách khắc phục từng cái là gì?
2. Slide nói IAM là global còn EC2 là regional. Theo bạn, **vì sao** IAM nên là global? Nếu IAM là regional thì chuyện gì khó chịu xảy ra?

## 7. Câu luyện kiểu SAA-C03

> Một công ty vận hành ứng dụng web trên EC2 ở một region. Yêu cầu mới: ứng dụng phải **tiếp tục chạy khi một data center bị sự cố**, đồng thời **chi phí tăng ít nhất có thể** và **không thay đổi region**. Giải pháp nào phù hợp?
>
> **A.** Chạy thêm các instance EC2 ở một region thứ hai và dùng Route 53 để chuyển hướng.
> **B.** Phân bổ các instance EC2 vào nhiều AZ trong cùng region, đặt sau một load balancer.
> **C.** Mua một instance lớn hơn trong cùng AZ để tăng độ bền.
> **D.** Dùng edge location để lưu bản sao ứng dụng gần người dùng.

Bạn trả lời và giải thích lý do chọn, mình chưa đưa đáp án. Sau đó mình sẽ chấm và giải thích vì sao từng phương án còn lại sai.

---

# Kiến trúc AWS cho app tài chính (Next.js + NestJS + MySQL)

## 1. Chọn Region

Chọn **Singapore (ap-southeast-1)**: gần Việt Nam nhất (khoảng 30-50ms), đủ service, có 3 AZ. Các region mới như Thái Lan (ap-southeast-7) hay Malaysia (ap-southeast-5) cũng gần nhưng ít service hơn và giá có thể khác.

**Lưu ý:** AWS chưa có region ở Việt Nam. Với app tài chính, bạn nên kiểm tra quy định lưu trữ dữ liệu tại Việt Nam (Luật An ninh mạng, Nghị định 53/2022) và yêu cầu của NHNN nếu có liên quan thanh toán hoặc ví điện tử. Mình không phải luật sư, nên nên xác nhận với bên pháp lý.

## 2. Sơ đồ tổng quan

```
User (VN)
   │
Route 53 ──► CloudFront (edge HCM/Hà Nội) + WAF + Shield
   │               │ static assets ──► S3 (Next.js static, uploads)
   ▼
┌──────────────────── VPC (ap-southeast-1) ────────────────────┐
│  Public subnets (2-3 AZ):   ALB,  NAT Gateway                │
│            │                                                  │
│  Private app subnets (2-3 AZ):                                │
│     ECS Fargate: Next.js (SSR)  ──►  NestJS API              │
│            │                           │         │            │
│            │                      SQS / worker   │            │
│  Private DB subnets (isolated):        │         │            │
│     RDS MySQL Multi-AZ (Primary AZ-a ⇄ Standby AZ-b)         │
│     ElastiCache Redis (Multi-AZ, cache/session/rate-limit)    │
└───────────────────────────────────────────────────────────────┘
   Hỗ trợ: Secrets Manager, KMS, CloudWatch, CloudTrail, GuardDuty, ECR
```

```
                      ALB (nằm ở cả 2 AZ)
                 ┌─────────┴──────────┐
                 ▼                    ▼
   AZ-a                        AZ-b
   Private app subnet A        Private app subnet B
   ┌───────────────┐           ┌───────────────┐
   │ NestJS task 1 │           │ NestJS task 2 │
   │ NestJS task 3 │           │ NestJS task 4 │
   └───────┬───────┘           └───────┬───────┘
           └──────────┬────────────────┘
                      ▼
   Private DB subnet:  RDS Primary (AZ-a) ══sync══► Standby (AZ-b)
```

## 3. Từng lớp

**Edge / Frontend**

- CloudFront đứng trước ALB và S3 để cache static (`/_next/static`), giảm latency cho user VN.
- AWS WAF: managed rules (SQLi, XSS), rate limit, chặn bot cho endpoint login/OTP.
- Next.js chạy SSR trên ECS Fargate. Nếu chủ yếu là static/CSR thì có thể host trên S3 + CloudFront cho rẻ.

**Backend**

- NestJS chạy trên ECS Fargate (hoặc EKS nếu team đã quen K8s), auto scaling theo CPU/request, tối thiểu 2 task ở 2 AZ.
- Job nền (gửi mail, đối soát, báo cáo) dùng SQS + worker task riêng, hoặc EventBridge Scheduler cho cron.

**Database: MySQL Multi-AZ**

| Lựa chọn                            | Failover         | Ghi chú                                                      |
| ----------------------------------- | ---------------- | ------------------------------------------------------------ |
| RDS MySQL Multi-AZ (1 standby)      | ~60-120s         | Đơn giản, rẻ nhất, standby không đọc được                    |
| RDS Multi-AZ DB cluster (2 standby) | ~35s             | Standby đọc được, replica lag thấp                           |
| **Aurora MySQL**                    | ~30s hoặc ít hơn | Storage tự replicate 6 bản qua 3 AZ, scale read tốt, đắt hơn |

Gợi ý: bắt đầu với **RDS MySQL Multi-AZ**, lên Aurora khi cần scale read hoặc RTO thấp hơn. Cấu hình thêm:

- Đặt trong isolated subnet, không public, Security Group chỉ cho phép từ SG của NestJS.
- Encryption at rest (KMS) và in transit (TLS), kết nối qua RDS Proxy nếu NestJS scale nhiều task.
- Automated backup (PITR 7-35 ngày), snapshot định kỳ, copy snapshot sang region khác để DR.
- Read replica cho báo cáo nặng.

**Cache:** ElastiCache Redis (Multi-AZ) cho session, rate limit, cache dữ liệu đọc nhiều, lock chống xử lý trùng giao dịch.

## 4. Bảo mật (quan trọng với tài chính)

- Secrets (DB password, API key) lưu trong **Secrets Manager**, không để trong env file hay image.
- IAM task role theo nguyên tắc least privilege. Bật MFA cho mọi tài khoản người dùng AWS.
- VPC endpoints (S3, ECR, Secrets Manager, CloudWatch) để traffic nội bộ không đi qua internet.
- CloudTrail, AWS Config và GuardDuty để audit và phát hiện bất thường.
- Audit log giao dịch ở tầng app, lưu bất biến (S3 Object Lock).
- Nếu xử lý thẻ thanh toán: cần PCI DSS, nên dùng cổng thanh toán (VNPay, MoMo, Stripe...) để tránh tự lưu dữ liệu thẻ.

## 5. CI/CD và vận hành

- GitHub Actions hoặc CodePipeline: build Docker, push ECR, deploy ECS (blue/green qua CodeDeploy).
- Hạ tầng bằng IaC (Terraform hoặc AWS CDK).
- CloudWatch Logs/Alarms, X-Ray hoặc OpenTelemetry để trace. Cảnh báo qua SNS tới Slack.
- Tách môi trường dev, staging, prod, tốt nhất là tách AWS account (AWS Organizations).

## 6. Lộ trình theo giai đoạn

1. **MVP:** 1 VPC, ECS Fargate (2 task), RDS Multi-AZ, ALB, CloudFront, S3.
2. **Tăng trưởng:** thêm Redis, RDS Proxy, WAF, SQS, read replica.
3. **Scale / compliance:** Aurora, DR sang region khác, tách account, SIEM.

Nếu bạn cho mình biết quy mô (số user, TPS dự kiến), ngân sách và có xử lý thanh toán trực tiếp hay không, mình có thể ước tính chi phí và vẽ sơ đồ chi tiết hơn.
