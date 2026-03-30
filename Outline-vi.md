---
layout: default
title: FPT-CHALLENGERS-GROUP 2 
---

# PROJECT OUTLINE: CLOUD-SENTINEL
**Hệ thống Giám sát và Phản ứng Bảo mật tự động đa tầng**

---

## 1. TỔNG QUAN DỰ ÁN 

* **Mục tiêu:** Xây dựng một hệ thống phát hiện, phân tích và phản ứng hoàn toàn tự động đối với các mối đe dọa trên nền tảng đám mây bằng cách kết hợp sức mạnh suy luận của AI (thông qua LLM) và kiến trúc Serverless hiện đại.
* **Vấn đề giải quyết:** * Chuyển đổi quy trình vận hành bảo mật từ thủ công sang tự động nhằm giảm thiểu gánh nặng cho quản trị viên.
    * Loại bỏ các báo động giả thông qua quy trình đánh giá chéo giữa các AI Agent.
    * Cung cấp khả năng phân tích sự cố theo thời gian thực và tự động thực thi các hành động bảo vệ sau khi có sự kiểm duyệt của chuyên viên.

---

## 2. KIẾN TRÚC HỆ THỐNG

![Cloud-Sentinel-Architecture-Diagram](./Cloud-Sentinel-Architecture-Diagram-ver3.gif)

Hệ thống được thiết kế hoàn toàn trên nền tảng AWS và triển khai tại khu vực Asia Pacific (Singapore). Kiến trúc được chia thành 5 lớp chuyên biệt theo dòng chảy của sơ đồ:

### 2.1. Lớp Nguồn dữ liệu & Sự kiện (Source Logs + Events and Activation Layer)
* **VPC Flow Logs:** Đóng vai trò là nguồn dữ liệu hạ tầng, thu thập toàn bộ nhật ký lưu lượng mạng.
* **Amazon GuardDuty:** Phân tích trực tiếp dữ liệu từ EC2 VPC Flow Logs. Quá trình này bắt đầu bằng việc đẩy dữ liệu log vào hệ thống (Bước 1: Push logs).
* **Amazon EventBridge:** Đóng vai trò làm cầu nối kích hoạt (Bước 2: Trigger). Dịch vụ này xử lý dữ liệu với kích thước payload ước tính là 2 KB để khởi chạy quy trình phân tích.

### 2.2. Lớp Điều phối (Orchestrator Layer)
Lớp điều phối được vận hành bởi **AWS Step Functions**, quản lý chuỗi hành động của các Agent. Cấu hình hệ thống ước tính xử lý 100 workflow requests mỗi tháng. Mỗi workflow thực hiện trung bình 40 state transitions.
* **Supervisor Agent:** Tiếp nhận yêu cầu khởi tạo từ EventBridge (Bước 3: Activate) và đóng vai trò như bộ não điều phối các hành động tiếp theo.
* **Precedent Check (DynamoDB):** Supervisor Agent truy vấn dữ liệu từ DynamoDB (Bước 3.1) để kiểm tra xem dấu hiệu bất thường đã có tiền lệ giải quyết hay chưa, giúp tối ưu hóa thời gian và chi phí xử lý.
* **RAG Lambda:** Nếu cần thêm tri thức để phân tích, Supervisor Agent kích hoạt AWS Lambda (Bước 3.2A). Dịch vụ Lambda được phân bổ 512 MB bộ nhớ ephemeral và chạy trên kiến trúc x86.
* **Advisor Agent:** Sau khi có đủ thông tin ngữ cảnh, Supervisor Agent kích hoạt Advisor Agent (Bước 3.2B) để tổng hợp tình hình và đưa ra khuyến nghị. Sức mạnh phân tích của các Agent được hỗ trợ bởi Amazon Bedrock.

### 2.3. Lớp Xác thực & Lưu trữ (Verify Layer)
* **Pinecone:** Hoạt động như một Vector Database phục vụ cho truy vấn dữ liệu của RAG Lambda (Query & Retrieve).
* **Amazon Cognito:** Quản lý quy trình xác thực quản trị viên trước khi họ được phép đưa ra quyết định. Dịch vụ này được cấu hình để phục vụ 800 người dùng hoạt động hàng tháng (MAU).

### 2.4. Lớp Phản ứng (Action Layer)
* **Telegram:** Advisor Agent kích hoạt thông báo (Bước 4: Trigger Notify). Thông báo chứa phân tích chi tiết sẽ được gửi đến quản trị viên qua Telegram (Bước 5: Notify).
* **Quy trình Phê duyệt (Authenticated User):** Chuyên viên bảo mật đã được xác thực sẽ đưa ra quyết định:
    * **Approve/Cancel (Bước 6A):** Chấp thuận kịch bản phản ứng, trực tiếp kích hoạt khối **Lambda Trigger** để tiến hành cô lập hoặc sửa lỗi.
    * **Reject (Bước 6B):** Từ chối đề xuất, chu trình sẽ hoàn trả về để phân tích lại hoặc hủy bỏ.
* **Lưu vết (Bước 7: Save history logs):** Sau khi hành động được thực thi, Lambda Trigger sẽ lưu lại lịch sử sự cố vào hệ thống cơ sở dữ liệu để làm tiền lệ cho tương lai.

### 2.5. Lớp Giám sát (Monitoring Layer)
* **Amazon CloudWatch:** Giám sát trạng thái hoạt động tổng thể của toàn bộ kiến trúc. Thiết lập giám sát bao gồm 1 Dashboard, 10 Metrics đo lường tùy chỉnh và 5 Standard Resolution Alarm Metrics.

---

## 3. LUỒNG DỮ LIỆU CỐT LÕI

1. **Ingestion & Detection:** Lưu lượng từ **VPC Flow Logs** được gửi đến **GuardDuty**.
2. **Activation:** Khi có bất thường, **GuardDuty** gửi tín hiệu đến **Event Bridge**, từ đó kích hoạt luồng **Step Functions**.
3. **Orchestration & Reasoning:**
    * **Supervisor Agent** tra cứu **DynamoDB** để kiểm tra tiền lệ.
    * Nếu cần bổ sung kiến thức, truy vấn **RAG Lambda** kết hợp cơ sở dữ liệu vector **Pinecone**.
    * **Advisor Agent** đề xuất phương án xử lý dựa trên dữ liệu thu thập được.
4. **Alerting:** Cảnh báo được chuyển qua **Telegram** đến hệ thống của chuyên viên bảo mật.
5. **Remediation:** Chuyên viên đăng nhập qua **Cognito**, kiểm tra và nhấn nút phê duyệt (Approve) để **Lambda Trigger** thực thi vá lỗi tự động, đồng thời lưu lại log hệ thống.

---

## 4. DỰ TOÁN CHI PHÍ VẬN HÀNH

[📄 Xem chi tiết Bảng dự toán chi phí AWS (PDF)](./Estimate_Costs.pdf)

Đây là mức chi phí ước tính được xuất ngày 26/03/2026 cho khu vực triển khai Asia Pacific (Singapore).

* **Tổng chi phí ước tính hàng tháng:** 14.46 USD.
* **Tổng chi phí ước tính cho 12 tháng:** 173.52 USD.

**Chi tiết các dịch vụ có phí:**
* **Amazon GuardDuty:** 6.90 USD/tháng cho 6 GB dữ liệu phân tích EC2 VPC Flow Log.
* **Amazon Bedrock:** 3.78 USD/tháng (On Demand - Standard). Ước tính 600 input tokens và 300 output tokens mỗi yêu cầu.
* **Amazon CloudWatch:** 3.50 USD/tháng (Bao gồm 1 Dashboard, 10 Metrics, 5 Alarms).
* **Amazon DynamoDB:** 0.28 USD/tháng (1 GB dữ liệu lưu trữ, kích thước item trung bình 10 KB).

**Các dịch vụ tối ưu ngân sách (0.00 USD/tháng):**
* Amazon EventBridge, AWS Step Functions, AWS Lambda và Amazon Cognito đều ở mức 0.00 USD/tháng dựa trên cấu hình sử dụng giới hạn của hệ thống.