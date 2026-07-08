---
title: "Bản đề xuất"
date: 2026-05-20
weight: 2
chapter: false
pre: " <b> 2. </b> "
---
# AI Meeting Assistant Platform
## Giải pháp AWS Serverless + VPC lai cho họp Chat & Voice có ghi âm và tự động giao việc

### 1. Tóm tắt điều hành
AI Meeting Assistant Platform là một nền tảng web (giao diện Next.js Dashboard) cho phép người dùng tổ chức các phiên **Chat & Voice** có ghi âm, rồi tự động biến mỗi phiên họp thành danh sách công việc cụ thể. Sau phiên họp, hệ thống đóng gói phần âm thanh và nội dung chat, gửi tới mô hình AI đa phương thức để chuyển giọng nói thành văn bản (Speech-to-Text) và tách công việc (Task Extraction), lưu các công việc thu được và gửi thông báo cho người được giao. Nền tảng cũng hỗ trợ thanh toán trực tuyến qua VNPay để nâng cấp gói. Hệ thống được triển khai tại vùng AWS **ap-southeast-1 (Singapore)**, kết hợp tầng serverless/managed cho các API nghiệp vụ và pipeline AI với tầng VPC + EC2 (Auto Scaling, 2 Availability Zone) cho dịch vụ Chat & Voice thời gian thực. Thiết kế hướng đến tính sẵn sàng cao, bảo mật nhiều lớp, tối ưu chi phí lưu trữ và giám sát tập trung, phục vụ quy mô ban đầu khoảng 100 người dùng hoạt động hàng tháng (MAU).

### 2. Tuyên bố vấn đề
*Vấn đề hiện tại*
Ghi chú họp và công việc phát sinh sau họp thường được xử lý thủ công: phải nghe lại bản ghi, gỡ băng, quyết định ai làm gì, rồi nhắc nhở để công việc thực sự được giao. Cách làm này chậm, thiếu nhất quán và dễ bỏ sót. Các công cụ họp có sẵn thì hoặc tốn kém, khó tùy biến, hoặc không gắn kết việc gỡ băng — tách việc — giao việc trong cùng một luồng, và ít công cụ phù hợp với một nhóm nhỏ đồng thời cần thanh toán trực tuyến.

*Giải pháp*
Nền tảng tiếp nhận toàn bộ lưu lượng qua Amazon CloudFront (được AWS WAF bảo vệ) và định tuyến các lời gọi API qua Amazon API Gateway; API Gateway kiểm tra JWT do Amazon Cognito cấp trước khi gọi logic nghiệp vụ trên AWS Lambda với dữ liệu trong Amazon DynamoDB. Phần Chat & Voice thời gian thực chạy trên Amazon EC2 trong VPC trải 2 Availability Zone, phía trước là Application Load Balancer và được Auto Scaling group quản lý; các phiên chat được ghi vào một bảng DynamoDB riêng thông qua VPC Interface Endpoint để lưu lượng không ra khỏi mạng AWS. Khi phiên họp kết thúc, audio và chat.json được đóng gói thành một "Meeting Bundle" trong Amazon S3. Sự kiện S3 kích hoạt Amazon EventBridge, từ đó khởi chạy workflow AWS Step Functions; một hàm Lambda gửi bundle cho AI đa phương thức bên ngoài (ví dụ Gemini / OpenAI / Claude), AI trả về bản gỡ băng và danh sách công việc đã tách chỉ trong một lần gọi. Một hàm Lambda thứ hai lưu công việc vào DynamoDB và Amazon SNS thông báo cho người được giao. Các bundle cũ được chuyển sang S3 Glacier theo chính sách vòng đời. Các tính năng chính gồm: họp có ghi âm, tự động gỡ băng, tự động tách và giao việc, thông báo giao việc, và thanh toán VNPay.

*Lợi ích và hoàn vốn đầu tư (ROI)*
Nền tảng loại bỏ vòng lặp thủ công "nghe — gỡ băng — tách việc — giao việc — nhắc việc", biến mỗi phiên họp có ghi âm thành danh sách công việc được theo dõi một cách tự động. Nhóm có một nơi duy nhất để họp và quản lý công việc phát sinh, cải thiện tính nhất quán và khả năng truy vết, đồng thời tạo kho lưu trữ bundle họp có thể tra cứu về sau. Thiết kế lai giúp chi phí dễ dự đoán: hạ tầng AWS ở mức **152,47 USD/tháng** (1.829,64 USD cho 12 tháng), phần gọi mô hình AI thêm khoảng **37,33 USD/tháng** ở kịch bản trung bình (khuyến nghị), tổng cộng khoảng **189,80 USD/tháng** (~2.277,54 USD/năm). Hỗ trợ thanh toán VNPay mở đường thu hồi chi phí qua các gói trả phí. Thời gian hoàn vốn phụ thuộc vào số giờ xử lý họp thủ công mà nhóm tiết kiệm được mỗi tháng.

### 3. Kiến trúc giải pháp
Nền tảng được tổ chức thành 5 nhóm dịch vụ: Edge (toàn cầu), Managed (ngoài VPC), VPC (Chat & Voice thời gian thực), AI Processing Pipeline (managed, ngoài VPC) và Monitoring. Request đi vào qua CloudFront với WAF lọc; tài nguyên tĩnh phục vụ từ S3, API động đi qua API Gateway tới Lambda và DynamoDB, còn các phiên thời gian thực được định tuyến qua Internet Gateway tới ALB và các EC2 do Auto Scaling quản lý trên 2 AZ. Bundle họp được ghi vào S3 và kích hoạt pipeline EventBridge → Step Functions → Lambda → AI bên ngoài, kết quả lưu vào DynamoDB và thông báo qua SNS. Toàn hệ thống được CloudWatch giám sát.

![](Architecture_1.png)

*Dịch vụ AWS sử dụng*
- *Amazon CloudFront*: CDN phân phối dashboard Next.js và cache nội dung tĩnh.
- *AWS WAF*: Lọc request độc hại (SQLi, XSS, bot) ở biên.
- *Amazon S3*: Tài nguyên tĩnh của web, Meeting Bundle (audio + chat.json) và lưu trữ lạnh Glacier (3 bucket/tier).
- *Amazon API Gateway*: Cổng vào duy nhất cho mọi request API.
- *Amazon Cognito*: Xác thực người dùng và cấp/kiểm tra JWT (~100 MAU).
- *AWS Lambda*: Logic API nghiệp vụ và các hàm của pipeline AI (gọi model, lưu công việc) — 3 hàm.
- *Amazon DynamoDB*: Dữ liệu nghiệp vụ (API DB), phiên chat (Save Chats) và công việc đã tách (Task DB) — 3 bảng.
- *Application Load Balancer*: Cân bằng tải Chat & Voice trên 2 AZ.
- *Amazon EC2 + Auto Scaling*: Server Chat & Voice thời gian thực (Primary/Standby trên 2 AZ), tự phục hồi theo tải.
- *Amazon VPC (Internet Gateway, NAT Gateway, Interface Endpoint)*: Mạng nội bộ, kiểm soát chiều ra và truy cập DynamoDB riêng tư.
- *Amazon EventBridge*: Kích hoạt pipeline AI khi có sự kiện "object created" của bundle.
- *AWS Step Functions*: Điều phối workflow xử lý AI.
- *Amazon SNS*: Thông báo giao việc và cảnh báo giám sát.
- *Amazon CloudWatch*: Tập trung logs, metrics và alarms.
- *VNPay (bên thứ ba)*: Cổng thanh toán trực tuyến.

*Thiết kế thành phần*
- *Edge*: CloudFront + WAF xử lý HTTPS đầu vào và phục vụ tài nguyên tĩnh từ S3.
- *Xác thực*: Cognito cấp JWT; API Gateway kiểm tra token trước khi gọi Lambda.
- *API nghiệp vụ*: Lambda đọc/ghi bảng DynamoDB nghiệp vụ.
- *Chat & Voice thời gian thực*: ALB định tuyến tới EC2 do Auto Scaling quản lý trên AZ A và AZ B; phiên chat được lưu vào bảng DynamoDB riêng qua Interface Endpoint (mỗi AZ 1) để đảm bảo HA.
- *Pipeline AI*: Sự kiện bundle trên S3 → EventBridge → Step Functions → Lambda gọi AI đa phương thức, AI trả về bản gỡ băng + công việc; một Lambda khác ghi công việc vào DynamoDB và SNS thông báo cho người dùng.
- *Vòng đời lưu trữ*: Bundle họp chuyển sang S3 Glacier theo chính sách 30/120 ngày.
- *Thanh toán*: Người dùng thanh toán qua VNPay; callback được xử lý qua API Gateway và Lambda.
- *Giám sát*: CloudWatch thu thập logs/metrics/alarms; SNS gửi cảnh báo.

### 4. Triển khai kỹ thuật
*Các giai đoạn triển khai*
Dự án được triển khai theo 4 giai đoạn:
1. *Nghiên cứu và vẽ kiến trúc*: Nghiên cứu thiết kế lai serverless + VPC và pipeline AI đa phương thức, hoàn thành sơ đồ kiến trúc (giai đoạn lên kế hoạch).
2. *Tính toán chi phí và kiểm tra tính khả thi*: Dùng AWS Pricing Calculator và ước tính chi phí AI (theo token); kiểm chứng tính khả thi với mục tiêu ~100 MAU (Tháng 1).
3. *Điều chỉnh kiến trúc để tối ưu chi phí/giải pháp*: Tinh chỉnh thiết kế — ví dụ bố trí NAT/endpoint, vòng đời S3, batching và caching cho lời gọi AI — để đảm bảo hiệu quả chi phí (Tháng 2).
4. *Phát triển, kiểm thử, triển khai*: Xây dựng ứng dụng Next.js, dịch vụ Chat & Voice trên EC2 và các dịch vụ AWS bằng CDK/SDK, sau đó kiểm thử và đưa vào vận hành (Tháng 2–3).

*Yêu cầu kỹ thuật*
- *Frontend & API*: Kiến thức thực tế về Next.js (host qua CloudFront/S3), API Gateway, Cognito (JWT) và Lambda + DynamoDB cho logic nghiệp vụ.
- *Tầng thời gian thực*: EC2 trong VPC trải 2 AZ, Application Load Balancer, Auto Scaling group, NAT Gateway cho chiều ra được kiểm soát và VPC Interface Endpoint để truy cập DynamoDB riêng tư.
- *Pipeline AI*: Thông báo sự kiện S3, EventBridge, Step Functions và Lambda để gọi mô hình AI đa phương thức (Speech-to-Text + tách việc trong một lần gọi), cùng SNS cho thông báo.
- *Tích hợp*: Dùng AWS CDK/SDK để khởi tạo và kết nối dịch vụ; tích hợp VNPay cho thanh toán; chính sách vòng đời S3 để kiểm soát chi phí.

### 5. Lộ trình & Mốc triển khai
- *Lên kế hoạch (Tháng 0)*: Nghiên cứu thiết kế, vẽ kiến trúc v1/v2 và chốt danh sách dịch vụ.
- *Triển khai (Tháng 1–3)*:
    - Tháng 1: Thiết lập tài khoản/vùng (ap-southeast-1), tìm hiểu dịch vụ và xác nhận ước tính chi phí.
    - Tháng 2: Xây dựng và điều chỉnh kiến trúc (VPC/EC2, API serverless, pipeline AI, thanh toán).
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
- Amazon DynamoDB (API DB + Task DB + Chat DB): 4,76 USD/tháng.
- Amazon CloudFront (~20 GB ra, 1 triệu request): 3,66 USD/tháng.
- Amazon API Gateway: 1,25 USD/tháng.
- Amazon S3 (Frontend + Audio + Glacier): 1,28 USD/tháng.
- Amazon SNS: 0,18 USD/tháng.
- AWS Lambda (3 hàm), Amazon Cognito (100 MAU), AWS Step Functions: 0,00 USD/tháng.

*Tổng phụ (AWS)*: **152,47 USD/tháng, 1.829,64 USD/12 tháng.**

*Chi phí mô hình AI (Gemini 2.5 Flash, ước tính)*
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
- AI bên thứ ba hoặc VNPay gián đoạn: Ảnh hưởng trung bình, xác suất thấp.

*Chiến lược giảm thiểu*
- Sẵn sàng: VPC đa AZ, ALB + Auto Scaling với failover Primary/Standby, và 2 NAT Gateway + 2 Interface Endpoint (mỗi AZ 1).
- Chi phí AI: Giới hạn max output token, dùng batch API và context caching, cân nhắc Flash-Lite cho tác vụ đơn giản.
- Bảo mật: WAF ở biên, phân quyền Cognito/JWT, EC2 nằm trong private subnet, và VPC Endpoint để lưu lượng DynamoDB/S3 không ra Internet.
- Phụ thuộc: Retry/backoff và xử lý dead-letter trong Step Functions; xử lý callback thanh toán an toàn.

*Kế hoạch dự phòng*
- Quay lại ghi chú thủ công nếu pipeline AI gặp sự cố, xử lý lại bundle sau.
- Dùng vòng đời S3 và cảnh báo ngân sách CloudWatch để kiểm soát lưu trữ và chi phí.
- Chạy lại (re-drive) các execution Step Functions để xử lý lại các bundle họp bị lỗi.

### 8. Kết quả kỳ vọng
*Cải tiến kỹ thuật*: Các phiên họp có ghi âm được gỡ băng và biến thành công việc được giao một cách tự động, thay thế quy trình xử lý thủ công. Tầng thời gian thực luôn sẵn sàng trên 2 AZ, và pipeline serverless co giãn theo mức sử dụng.
*Giá trị dài hạn*: Kho lưu trữ bundle họp và công việc đã tách có thể tra cứu, một mẫu kiến trúc lai (serverless + VPC) có thể tái sử dụng, và nền tảng đã tích hợp thanh toán để phát triển cùng nhóm.
