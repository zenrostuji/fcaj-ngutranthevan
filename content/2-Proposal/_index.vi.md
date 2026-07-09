---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# AI Meeting Assistant Platform
## Giải pháp AWS Serverless + VPC lai cho họp Chat & Voice có ghi âm và tự động giao việc

### 1. Tóm tắt điều hành
AI Meeting Assistant Platform là một nền tảng web (giao diện Next.js Dashboard) cho phép người dùng tổ chức các phiên **Chat & Voice** có ghi âm và tự động biến mỗi phiên họp thành công việc cụ thể. Âm thanh và bản ghi được tải lên Amazon S3, kích hoạt một pipeline hướng sự kiện: một Speech Processing Lambda chuyển giọng nói thành văn bản thông qua một AI API bên ngoài, và một Task Lambda tách và giao việc thu được, lưu vào DynamoDB, rồi thông báo cho nhóm. Hệ thống được triển khai tại vùng AWS **ap-southeast-1 (Singapore)**, kết hợp tầng serverless/managed cho các API nghiệp vụ và pipeline AI với tầng VPC + EC2 (Auto Scaling, 2 Availability Zone) cho phần Chat & Voice thời gian thực. Thiết kế hướng đến tính sẵn sàng cao, bảo mật nhiều lớp, tối ưu chi phí lưu trữ và giám sát tập trung, phục vụ quy mô ban đầu khoảng 100 người dùng hoạt động hàng tháng (MAU).

### 2. Tuyên bố vấn đề
*Vấn đề hiện tại*
Ghi chú họp và công việc phát sinh sau họp thường được xử lý thủ công: phải nghe lại bản ghi, gỡ băng, quyết định ai làm gì, rồi nhắc nhở để công việc thực sự được giao. Cách làm này chậm, thiếu nhất quán và dễ bỏ sót. Các công cụ họp có sẵn thì hoặc tốn kém, khó tùy biến, hoặc không gắn kết việc gỡ băng — tách việc — giao việc trong cùng một luồng, và ít công cụ phù hợp với một nhóm nhỏ.

*Giải pháp*
Nền tảng tiếp nhận toàn bộ lưu lượng qua Amazon CloudFront (được AWS WAF bảo vệ) và định tuyến các lời gọi API qua Amazon API Gateway; API Gateway kiểm tra JWT do Amazon Cognito cấp trước khi gọi logic nghiệp vụ trên API Lambda với dữ liệu trong Amazon DynamoDB. Phần Chat & Voice thời gian thực chạy trên Amazon EC2 trong VPC trải 2 Availability Zone, phía trước là Application Load Balancer và được Auto Scaling group quản lý; các phiên chat được ghi vào một bảng DynamoDB riêng thông qua VPC Interface Endpoint để lưu lượng không ra khỏi mạng AWS. Khi âm thanh/bản ghi của phiên họp được tải lên Amazon S3, sự kiện S3 kích hoạt Amazon EventBridge, từ đó khởi chạy workflow AWS Step Functions. Trong workflow, một Speech Processing Lambda gọi AI API bên ngoài để chuyển giọng nói thành văn bản, và một Task Lambda gọi AI API bên ngoài để tách và giao việc, rồi lưu các công việc vào DynamoDB. Amazon SNS thông báo cho người dùng, các đối tượng cũ được chuyển sang S3 Glacier theo chính sách vòng đời, và Amazon CloudWatch giám sát toàn hệ thống. Các tính năng chính gồm: họp có ghi âm, tự động gỡ băng, và tự động tách & giao việc.

*Lợi ích và hoàn vốn đầu tư (ROI)*
Nền tảng loại bỏ vòng lặp thủ công "nghe — gỡ băng — tách việc — giao việc — nhắc việc", biến mỗi phiên họp có ghi âm thành danh sách công việc được theo dõi một cách tự động. Nhóm có một nơi duy nhất để họp và quản lý công việc phát sinh, cải thiện tính nhất quán và khả năng truy vết, đồng thời tạo kho lưu trữ âm thanh và bản ghi họp có thể tra cứu về sau. Thiết kế lai giúp chi phí dễ dự đoán: hạ tầng AWS ở mức **152,47 USD/tháng** (1.829,64 USD cho 12 tháng), phần gọi mô hình AI thêm khoảng **37,33 USD/tháng** ở kịch bản trung bình (khuyến nghị), tổng cộng khoảng **189,80 USD/tháng** (~2.277,54 USD/năm). Thời gian hoàn vốn phụ thuộc vào số giờ xử lý họp thủ công mà nhóm tiết kiệm được mỗi tháng.

### 3. Kiến trúc giải pháp
Nền tảng được tổ chức thành các nhóm dịch vụ: Edge, Managed (ngoài VPC), VPC (Chat & Voice thời gian thực), AI Processing Pipeline (Step Functions, ngoài VPC) và Monitoring. Luồng end-to-end đi theo các bước được đánh số trên sơ đồ: người dùng đăng nhập qua Cognito và tải dashboard qua CloudFront (WAF lọc, tài nguyên tĩnh từ S3); lời gọi API đi qua API Gateway tới API Lambda và DynamoDB; âm thanh/bản ghi được tải lên S3, kích hoạt pipeline EventBridge → Step Functions → Speech Processing Lambda / Task Lambda → AI bên ngoài, công việc được lưu vào DynamoDB; và phần Chat & Voice thời gian thực được định tuyến qua Internet Gateway tới ALB và các EC2 (Auto Scaling trên 2 AZ), với phiên chat được lưu vào một bảng DynamoDB riêng qua VPC Interface Endpoint. CloudWatch và SNS đảm nhận giám sát và cảnh báo.

![AI Meeting Assistant Architecture](Architecture_1.png)

*Dịch vụ AWS sử dụng*
- *Amazon CloudFront*: CDN phân phối dashboard Next.js và cache nội dung tĩnh.
- *AWS WAF*: Lọc request độc hại (SQLi, XSS, bot) ở biên.
- *Amazon S3*: Tài nguyên tĩnh của web, âm thanh/bản ghi tải lên, và lưu trữ lạnh Glacier.
- *Amazon API Gateway*: Cổng vào duy nhất cho mọi request API.
- *Amazon Cognito*: Xác thực người dùng và cấp/kiểm tra JWT (~100 MAU).
- *AWS Lambda*: API Lambda cho logic nghiệp vụ, cùng Speech Processing Lambda (STT) và Task Lambda (tách/giao việc) — 3 hàm.
- *Amazon DynamoDB*: Dữ liệu nghiệp vụ (API Data), phiên chat (Save Chats) và công việc đã tách (Task) — 3 bảng.
- *Application Load Balancer*: Cân bằng tải Chat & Voice trên 2 AZ.
- *Amazon EC2 + Auto Scaling*: Server Chat & Voice thời gian thực trên 2 AZ, tự phục hồi theo tải.
- *Amazon VPC (Internet Gateway, NAT Gateway, Interface Endpoint)*: Mạng nội bộ, kiểm soát chiều ra (mỗi AZ một NAT Gateway) và truy cập DynamoDB riêng tư.
- *Amazon EventBridge*: Kích hoạt pipeline AI khi có sự kiện "object created" trên S3.
- *AWS Step Functions*: Điều phối workflow Speech Processing và Task.
- *External AI API*: Speech-to-Text và tách công việc (ví dụ Gemini / OpenAI / Claude).
- *Amazon SNS*: Thông báo (email) và cảnh báo giám sát.
- *Amazon CloudWatch*: Tập trung logs, metrics và alarms.

*Thiết kế thành phần*
- *Edge*: CloudFront + WAF xử lý HTTPS đầu vào và phục vụ tài nguyên tĩnh từ S3.
- *Xác thực*: Cognito cấp JWT (bước 1); API Gateway kiểm tra token (bước 6) trước khi gọi Lambda.
- *API nghiệp vụ*: API Lambda đọc/ghi bảng DynamoDB API Data (bước 7–8).
- *Tải lên*: Âm thanh/bản ghi được tải lên S3 (bước 9).
- *Pipeline AI*: Sự kiện object-created trên S3 → EventBridge → Step Functions (bước 10–11); Speech Processing Lambda thực hiện Speech-to-Text qua AI API bên ngoài (bước 12); Task Lambda tách & giao việc qua AI API bên ngoài (bước 13) và lưu công việc vào DynamoDB (bước 15).
- *Chat & Voice thời gian thực*: Chat request đi CloudFront → Internet Gateway → ALB → EC2 (Auto Scaling, bước 17–19); phiên chat được lưu vào bảng DynamoDB riêng qua Interface Endpoint (bước 20–21).
- *Vòng đời lưu trữ*: Đối tượng chuyển sang S3 Glacier theo chính sách 30/120 ngày (bước 16).
- *Giám sát*: CloudWatch thu thập logs/metrics/alarms và gửi cảnh báo qua SNS (email).

### 4. Triển khai kỹ thuật
*Các giai đoạn triển khai*
Dự án được triển khai theo 4 giai đoạn:
1. *Nghiên cứu và vẽ kiến trúc*: Nghiên cứu thiết kế lai serverless + VPC và pipeline AI, hoàn thành sơ đồ kiến trúc (giai đoạn lên kế hoạch).
2. *Tính toán chi phí và kiểm tra tính khả thi*: Dùng AWS Pricing Calculator và ước tính chi phí AI (theo token); kiểm chứng tính khả thi với mục tiêu ~100 MAU (Tháng 1).
3. *Điều chỉnh kiến trúc để tối ưu chi phí/giải pháp*: Tinh chỉnh thiết kế — ví dụ bố trí NAT/endpoint, vòng đời S3, batching và caching cho lời gọi AI — để đảm bảo hiệu quả chi phí (Tháng 2).
4. *Phát triển, kiểm thử, triển khai*: Xây dựng ứng dụng Next.js, dịch vụ Chat & Voice trên EC2 và các dịch vụ AWS bằng CDK/SDK, sau đó kiểm thử và đưa vào vận hành (Tháng 2–3).

*Yêu cầu kỹ thuật*
- *Frontend & API*: Kiến thức thực tế về Next.js (host qua CloudFront/S3), API Gateway, Cognito (JWT) và Lambda + DynamoDB cho logic nghiệp vụ.
- *Tầng thời gian thực*: EC2 trong VPC trải 2 AZ, Application Load Balancer, Auto Scaling group, NAT Gateway cho chiều ra được kiểm soát và VPC Interface Endpoint để truy cập DynamoDB riêng tư.
- *Pipeline AI*: Thông báo sự kiện S3, EventBridge, Step Functions, một Speech Processing Lambda (STT) và một Task Lambda (tách/giao việc) gọi AI API bên ngoài, cùng SNS cho thông báo.
- *Tích hợp*: Dùng AWS CDK/SDK để khởi tạo và kết nối dịch vụ, và chính sách vòng đời S3 để kiểm soát chi phí.

### 5. Lộ trình & Mốc triển khai
- *Lên kế hoạch (Tháng 0)*: Nghiên cứu thiết kế, vẽ kiến trúc và chốt danh sách dịch vụ.
- *Triển khai (Tháng 1–3)*:
    - Tháng 1: Thiết lập tài khoản/vùng (ap-southeast-1), tìm hiểu dịch vụ và xác nhận ước tính chi phí.
    - Tháng 2: Xây dựng và điều chỉnh kiến trúc (VPC/EC2, API serverless, pipeline AI).
    - Tháng 3: Triển khai, kiểm thử end-to-end và đưa vào sử dụng.
- *Sau triển khai*: Vận hành, giám sát và tinh chỉnh liên tục.

### 6. Ước tính ngân sách
Có thể xem chi phí trên [AWS Pricing Calculator](https://calculator.aws/) hoặc tải [tệp ước tính ngân sách](../attachments/Cost.pdf). Chi phí gọi mô hình AI được trình bày chi tiết trong bản ước tính Gemini 2.5 Flash kèm theo.

*Chi phí hạ tầng (AWS, ap-southeast-1)*
- Amazon EC2 (t3.medium, 1 instance, 30 GB EBS): 41,82 USD/tháng.
- Amazon VPC (NAT Gateway + IPv4 public): 49,67 USD/tháng.
- Elastic Load Balancing (1 ALB): 19,20 USD/tháng.
- Amazon CloudWatch (logs, metrics, alarms, 1 dashboard): 19,05 USD/tháng.
- AWS WAF (1 Web ACL, 5 rule + 1 managed group): 11,60 USD/tháng.
- Amazon DynamoDB (API Data + Task + Save Chats): 4,76 USD/tháng.
- Amazon CloudFront (~20 GB ra, 1 triệu request): 3,66 USD/tháng.
- Amazon API Gateway: 1,25 USD/tháng.
- Amazon S3 (Frontend + Audio + Glacier): 1,28 USD/tháng.
- Amazon SNS: 0,18 USD/tháng.
- AWS Lambda (3 hàm), Amazon Cognito (100 MAU), AWS Step Functions: 0,00 USD/tháng.

*Tổng phụ (AWS)*: **152,47 USD/tháng, 1.829,64 USD/12 tháng.**

*Chi phí mô hình AI (Gemini 2.5 Flash, ước tính)*
Bao gồm các lời gọi AI API bên ngoài trong pipeline (Speech-to-Text và tách công việc):
- Mức thấp: ~15,53 USD/tháng.
- Mức trung bình (mốc khuyến nghị): ~37,33 USD/tháng.
- Mức cao: ~96,53 USD/tháng.

*Tổng cộng (kịch bản trung bình khuyến nghị)*
- Hàng tháng: ~189,80 USD (AWS 152,47 + AI 37,33).
- 12 tháng: ~2.277,54 USD.

### 7. Đánh giá rủi ro
*Ma trận rủi ro*
- Lỗi AZ hoặc EC2 (gián đoạn Chat & Voice): Ảnh hưởng cao, xác suất thấp.
- Vượt chi phí AI (output dài / audio nặng): Ảnh hưởng trung bình, xác suất trung bình.
- Sự cố bảo mật (lưu lượng độc hại, lạm dụng thông tin đăng nhập): Ảnh hưởng cao, xác suất thấp.
- AI API bên ngoài gián đoạn: Ảnh hưởng trung bình, xác suất thấp.

*Chiến lược giảm thiểu*
- Sẵn sàng: VPC đa AZ, ALB + Auto Scaling với EC2 tự phục hồi, và mỗi AZ một NAT Gateway.
- Chi phí AI: Giới hạn max output token, dùng batch API và context caching, cân nhắc Flash-Lite cho tác vụ đơn giản.
- Bảo mật: WAF ở biên, phân quyền Cognito/JWT, EC2 nằm trong private subnet, và VPC Endpoint để lưu lượng DynamoDB không ra Internet.
- Phụ thuộc: Retry/backoff và xử lý dead-letter trong Step Functions cho các lời gọi AI bên ngoài.

*Kế hoạch dự phòng*
- Quay lại ghi chú thủ công nếu pipeline AI gặp sự cố, xử lý lại các file tải lên sau.
- Dùng vòng đời S3 và cảnh báo ngân sách CloudWatch để kiểm soát lưu trữ và chi phí.
- Chạy lại (re-drive) các execution Step Functions để xử lý lại các file âm thanh/bản ghi bị lỗi.

### 8. Kết quả kỳ vọng
*Cải tiến kỹ thuật*: Các phiên họp có ghi âm được gỡ băng và biến thành công việc được giao một cách tự động, thay thế quy trình thủ công. Tầng thời gian thực luôn sẵn sàng trên 2 AZ, và pipeline serverless co giãn theo mức sử dụng.
*Giá trị dài hạn*: Kho lưu trữ âm thanh, bản ghi và công việc đã tách có thể tra cứu, cùng một mẫu kiến trúc lai (serverless + VPC) có thể tái sử dụng và phát triển cùng nhóm.
