---
title: "Proposal"
date: 2026-05-20
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# AI Meeting Assistant Platform
## A Hybrid AWS Serverless + VPC Solution for Recorded Chat & Voice Meetings with Automated Task Assignment

### 1. Executive Summary
The AI Meeting Assistant Platform is a web application (Next.js Dashboard) that lets users hold recorded **Chat & Voice** meetings, then automatically turns each session into actionable work. After a meeting, the platform bundles the audio and chat content, sends it to a multimodal AI model for Speech-to-Text and task extraction, stores the resulting tasks, and notifies the assignees. It also supports online payments via VNPay for plan upgrades. The system is deployed in the AWS **ap-southeast-1 (Singapore)** region and combines a serverless/managed layer for business APIs and the AI pipeline with a VPC + EC2 (Auto Scaling, 2 Availability Zones) layer for the real-time Chat & Voice service. It is designed for high availability, layered security, storage cost optimization, and centralized monitoring, serving an initial base of roughly 100 monthly active users (MAU).

### 2. Problem Statement
### What's the Problem?
Meeting notes and follow-up tasks are usually captured manually. Someone has to listen back to recordings, transcribe them, decide who does what, and then chase people to actually get the work assigned. This is slow, inconsistent, and easy to forget. Off-the-shelf meeting tools are either expensive, hard to customize, or don't tie transcription, task extraction, and assignment together in one flow, and few of them fit a small team that also needs online payment.

### The Solution
The platform ingests all traffic through Amazon CloudFront (protected by AWS WAF) and routes API calls through Amazon API Gateway, which validates JWTs issued by Amazon Cognito before invoking business logic in AWS Lambda backed by Amazon DynamoDB. Real-time Chat & Voice runs on Amazon EC2 inside a VPC across two Availability Zones, fronted by an Application Load Balancer and managed by an Auto Scaling group; chat sessions are written to a dedicated DynamoDB table through VPC Interface Endpoints so traffic never leaves the AWS network. When a meeting ends, the audio and chat.json are packaged into an Amazon S3 "Meeting Bundle." An S3 event triggers Amazon EventBridge, which starts an AWS Step Functions workflow; a Lambda function sends the bundle to an external multimodal AI (e.g., Gemini / OpenAI / Claude) that returns the transcript and the extracted task list in a single call. A second Lambda saves those tasks to DynamoDB, and Amazon SNS notifies the assignees. Older bundles roll off to S3 Glacier on a lifecycle policy. Key features include recorded meetings, automatic transcription, automated task extraction and assignment, assignment notifications, and VNPay payments.

### Benefits and Return on Investment
The platform removes the manual "listen, transcribe, split, assign, and chase" cycle, turning every recorded meeting into a tracked task list automatically. It gives the team a single place to run meetings and manage the work that comes out of them, improves consistency and traceability, and creates a searchable archive of meeting bundles for later reference. The hybrid design keeps costs predictable: the AWS infrastructure runs at **$152.47/month** ($1,829.64 for 12 months), and the AI model usage adds roughly **$37.33/month** at the recommended medium-usage scenario, for a combined **~$189.80/month** (~$2,277.54/year). Payment support via VNPay opens a path to recover cost through paid plans. Break-even depends on how many hours of manual meeting follow-up the team eliminates each month.

### 3. Solution Architecture
The platform is organized into five service groups: Edge (global), Managed (outside the VPC), the VPC (real-time Chat & Voice), the AI Processing Pipeline (managed, outside the VPC), and Monitoring. Requests enter through CloudFront with WAF filtering; static assets are served from S3, dynamic APIs go through API Gateway to Lambda and DynamoDB, and real-time sessions are routed through the Internet Gateway to an ALB and EC2 instances managed by Auto Scaling across two AZs. Meeting bundles land in S3 and drive the EventBridge → Step Functions → Lambda → external AI pipeline, with results saved to DynamoDB and notifications sent via SNS. Everything is observed by CloudWatch. The architecture is detailed below:

![](Architecture.drawio.png)

### AWS Services Used
- **Amazon CloudFront**: CDN that distributes the Next.js dashboard and caches static content.
- **AWS WAF**: Filters malicious requests (SQLi, XSS, bots) at the edge.
- **Amazon S3**: Static web assets, the Meeting Bundle (audio + chat.json), and Glacier cold storage (three buckets/tiers).
- **Amazon API Gateway**: Single entry point for all API traffic.
- **Amazon Cognito**: User authentication and JWT issuance/validation (~100 MAU).
- **AWS Lambda**: Business API logic plus AI-pipeline functions (call model, save tasks) — three functions.
- **Amazon DynamoDB**: Business data (API DB), chat sessions (Save Chats), and extracted tasks (Task DB) — three tables.
- **Application Load Balancer**: Balances Chat & Voice traffic across two AZs.
- **Amazon EC2 + Auto Scaling**: Real-time Chat & Voice servers (Primary/Standby across two AZs), self-healing under load.
- **Amazon VPC (Internet Gateway, NAT Gateways, Interface Endpoints)**: Private networking, controlled egress, and private DynamoDB access.
- **Amazon EventBridge**: Triggers the AI pipeline on "object created" bundle events.
- **AWS Step Functions**: Orchestrates the AI processing workflow.
- **Amazon SNS**: Task-assignment notifications and monitoring alarms.
- **Amazon CloudWatch**: Centralized logs, metrics, and alarms.
- **VNPay (third party)**: Online payment gateway.

### Component Design
- **Edge**: CloudFront + WAF handle inbound HTTPS and serve static assets from S3.
- **Authentication**: Cognito issues JWTs; API Gateway validates them before invoking Lambda.
- **Business API**: Lambda reads/writes the API DynamoDB table.
- **Real-time Chat & Voice**: ALB routes to EC2 instances managed by an Auto Scaling group across AZ A and AZ B; chats are saved to a dedicated DynamoDB table through Interface Endpoints (one per AZ) for high availability.
- **AI Pipeline**: An S3 bundle event → EventBridge → Step Functions → Lambda calls the external multimodal AI, which returns transcript + tasks; a second Lambda writes tasks to DynamoDB and SNS notifies users.
- **Storage Lifecycle**: Meeting bundles transition to S3 Glacier on a 30/120-day policy.
- **Payment**: Users pay through VNPay; the callback is processed via API Gateway and Lambda.
- **Monitoring**: CloudWatch collects logs/metrics/alarms; SNS raises alerts.

### 4. Technical Implementation
**Implementation Phases**
This project is delivered in four phases:
- Build Theory and Draw Architecture: Research the hybrid serverless + VPC design and the multimodal AI pipeline, and produce the architecture diagrams (planning phase).
- Calculate Price and Check Practicality: Use the AWS Pricing Calculator and estimate AI (token) costs; validate feasibility against the target ~100 MAU (Month 1).
- Fix Architecture for Cost or Solution Fit: Tune the design — e.g., NAT/endpoint layout, S3 lifecycle, batching and caching for AI calls — to stay cost-effective (Month 2).
- Develop, Test, and Deploy: Build the Next.js app, the EC2 Chat & Voice service, and the AWS services via CDK/SDK, then test and release to production (Months 2–3).

**Technical Requirements**
- Frontend & API: Practical knowledge of Next.js (hosted via CloudFront/S3), API Gateway, Cognito (JWT), and Lambda + DynamoDB for business logic.
- Real-time layer: EC2 within a VPC across two AZs, an Application Load Balancer, an Auto Scaling group, NAT Gateways for controlled egress, and VPC Interface Endpoints for private DynamoDB access.
- AI pipeline: S3 event notifications, EventBridge, Step Functions, and Lambda to call a multimodal AI model (Speech-to-Text + task extraction in one call), plus SNS for notifications.
- Integration: AWS CDK/SDK to provision and wire services; VNPay integration for payments; S3 lifecycle policies for cost control.

### 5. Timeline & Milestones
**Project Timeline**
- Planning (Month 0): Research the design, draw v1/v2 architecture, and confirm the service list.
- Implementation (Months 1–3):
    - Month 1: Set up accounts/region (ap-southeast-1), study services, and confirm cost estimates.
    - Month 2: Build and adjust the architecture (VPC/EC2, serverless API, AI pipeline, payment).
    - Month 3: Implement, test end-to-end, and launch.
- Post-Launch: Ongoing operation, monitoring, and tuning.

### 6. Budget Estimation
You can view the full estimate on the [AWS Pricing Calculator](https://calculator.aws/) or download the [Budget Estimation File](../attachments/Cost.pdf). AI model costs are detailed in the accompanying Gemini 2.5 Flash estimate.

### Infrastructure Costs (AWS, ap-southeast-1)
- Amazon EC2 (t3.medium, 1 instance, 30 GB EBS): $41.82/month.
- Amazon VPC (NAT Gateways + public IPv4): $49.67/month.
- Elastic Load Balancing (1 ALB): $19.20/month.
- Amazon CloudWatch (logs, metrics, alarms, 1 dashboard): $19.05/month.
- AWS WAF (1 Web ACL, 5 rules + 1 managed group): $11.60/month.
- Amazon DynamoDB (API DB + Task DB + Chat DB): $4.76/month.
- Amazon CloudFront (~20 GB out, 1M requests): $3.66/month.
- Amazon API Gateway: $1.25/month.
- Amazon S3 (Frontend + Audio + Glacier): $1.28/month.
- Amazon SNS: $0.18/month.
- AWS Lambda (3 functions), Amazon Cognito (100 MAU), AWS Step Functions: $0.00/month.

Subtotal (AWS): **$152.47/month, $1,829.64/12 months.**

### AI Model Costs (Gemini 2.5 Flash, estimated)
- Low usage: ~$15.53/month.
- Medium usage (recommended baseline): ~$37.33/month.
- High usage: ~$96.53/month.

### Combined Total (recommended medium scenario)
- Monthly: ~$189.80 (AWS $152.47 + AI $37.33).
- 12 months: ~$2,277.54.

### 7. Risk Assessment
#### Risk Matrix
- AZ or EC2 failure (Chat & Voice interruption): High impact, low probability.
- AI cost overruns (long outputs / heavy audio): Medium impact, medium probability.
- Security incidents (malicious traffic, credential abuse): High impact, low probability.
- Third-party AI or VNPay outage: Medium impact, low probability.

#### Mitigation Strategies
- Availability: Multi-AZ VPC, ALB + Auto Scaling with Primary/Standby failover, and two NAT Gateways + two Interface Endpoints (one per AZ).
- AI cost: Cap max output tokens, use batch API and context caching, and consider Flash-Lite for simple tasks.
- Security: WAF at the edge, Cognito/JWT authorization, EC2 in private subnets, and VPC Endpoints so DynamoDB/S3 traffic stays off the internet.
- Dependencies: Retry/backoff and dead-letter handling in Step Functions; graceful handling of payment callbacks.

#### Contingency Plans
- Fall back to manual note-taking if the AI pipeline is unavailable, then reprocess bundles later.
- Use S3 lifecycle and CloudWatch budget alarms to control storage and cost.
- Re-drive Step Functions executions to reprocess failed meeting bundles.

### 8. Expected Outcomes
#### Technical Improvements
Recorded meetings are transcribed and turned into assigned tasks automatically, replacing manual follow-up. The real-time layer stays available across two AZs, and the serverless pipeline scales with usage.
#### Long-term Value
A searchable archive of meeting bundles and extracted tasks, a reusable hybrid architecture pattern (serverless + VPC), and a payment-enabled foundation that can grow with the team.
