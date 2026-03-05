# PROJECT OUTLINE: CLOUD-SENTINEL

**Multi-tier Security Monitoring and Response System based on Multi-Agent AI Architecture**

---

## 1. PROJECT OVERVIEW

* **Objective:** To build an automated system for detecting, analyzing, and responding to security threats on cloud infrastructure by coordinating specialized AI Agents through a Multi-Agent Orchestration model.
* **Problem Statement:** 
    * Reduce the operational burden on the SOC team.
    * Minimize false positives through multi-tier verification.
    * Shorten threat response time via automated response workflows.


## 2. SYSTEM ARCHITECTURE

The architecture is designed based on a Serverless model, divided into 5 centrally managed functional layers:

### 2.1. Source Logs + Event & Activation Layer

* **Data Sources:** * **AWS CloudTrail:** Records the complete API activity history of the account.
* **VPC Flow Logs:** Monitors IP traffic flowing through network interfaces within the VPC.


* **Activation Mechanism:** * **Native API Integration:** CloudTrail integrates directly with EventBridge to respond in real-time to unauthorized configuration changes.
* **S3 Event Notification:** Triggers analysis workflows when large log files are uploaded to the S3 Bucket.



### 2.2. Orchestrator Layer

Housed within the management framework of **AWS Step Functions**, ensuring process consistency and resilience.

* **Security Supervisor Agent:** The central "brain" that receives requests, plans execution, and coordinates 4 specialist agents:
* **Agent 1 (The Sifter Agent):** Retrieves raw logs from S3, performs screening, and writes "Refined Data" back to S3 to optimize Token costs for subsequent steps.
* **Agent 2 (The Profiler Agent):** Reads refined data, maps behaviors against the **MITRE ATT&CK** framework to identify threat types.
* **Agent 3 (The Advisor Agent):** References Best Practices (NIST, CIS) to propose appropriate Remediation scripts.
* **Agent 4 (The Validator Agent):** Performs final cross-verification to ensure information accuracy before issuing alerts.

### 2.3. Data & RAG + Governance & State Layer

* **Agentic RAG Layer:** Combines Amazon S3 (Data Storage) and OpenSearch (Vector Database) to provide in-depth knowledge and real-world context for Agents during the reasoning process.
* **Semantic Caching (DynamoDB):** * Stores the thinking traces and analysis results of the Agents.
* Implements a **Cache Lookup** mechanism: Agents check if similar incidents have occurred previously to reduce the load on the RAG and optimize operational costs.



### 2.4. Action Layer

* **Amazon SNS:** Issues immediate alerts via Email, SMS.
* **Admin Approval Dashboard:** Integrates **Amazon Cognito** to authenticate security professionals, allowing the approval or rejection of remediation actions directly from the Dashboard.
* **Lambda:** Executes commands to block IPs, revoke IAM access, or isolate compromised resources following human confirmation.

### 2.5. Comprehensive Monitoring Layer

* **Amazon CloudWatch**: Monitors the operational "health" of the entire AI system, including Lambda execution logs, Step Functions states, and API latency.
* **Observability Agent**: Tracks the performance of each Agent, Token consumption, and processing throughput to ensure the system is always ready and auditable.

## 3. KEY DATA FLOW

1. **Ingestion:** Logs are pushed to S3 -> EventBridge -> Triggers Step Functions orchestration.
2. **Sifting:** Agent 1 fetches raw logs from S3 -> Filters noise -> Writes refined metadata to S3.
3. **Reasoning (Cache-First):** 
    * Agent 2, 3, 4 retrieve refined data from S3.
    * Cross-reference with DynamoDB (Cache): If results are available -> Bypass the RAG step.
    * If Cache-miss -> Query Agentic RAG for knowledge -> Update the Cache.

4. **Verification & Reporting:** Agent 4 consolidates results -> Sends verified reports to **SNS**.
5. **Remediation:** Admin approves -> Lambda performs immediate incident containment.

## 4. IMPACT ASSESSMENT

* **Performance:** Processes and delivers response decisions in a short time.
* **Accuracy:** Minimizes false positives thanks to the multi-tier verification process and foundational knowledge from RAG.
* **Cost Efficiency:** Optimizes LLM costs through the **Sifting** and **Semantic Caching** mechanisms.
* **Scalability:** The **Serverless** architecture allows the system to automatically scale with log volume, supporting multi-region and multi-account monitoring.
