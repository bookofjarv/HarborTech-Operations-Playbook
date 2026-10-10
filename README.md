# Week 6: Containers and Serverless Computing

## HarborTech Ticket Summary
Ticket TKT-2026-0006 addresses Bright Path Community Services running a daily reporting task and a small REST API on a single EC2 instance. The server spends most of its time idle, yet the client continues to pay for constant compute time and manage OS-level maintenance. The goal is to evaluate workload patterns and assign each to the most effective compute model (VM, Container, or Serverless).

## Client Impact
Using an mismatched compute model leads to unnecessary operational costs, idle capacity, and administrative burden. Over-provisioning full virtual machines for lightweight, periodic tasks wastes budget on idle compute and OS management. Conversely, attempting to force workloads with deep OS dependencies into restrictive serverless environments can cause application instability and loss of administrative control.

## AWS Services Involved
- **AWS Lambda:** Serverless compute for event-driven code execution without server management.
- **Amazon API Gateway:** Managed service for creating, publishing, and securing REST APIs.
- **Amazon EventBridge:** Event bus used to trigger daily scheduled executions for Lambda.
- **Amazon EC2:** Virtual machine compute for workloads requiring full OS control.
- **AWS IAM:** Access management for configuring least-privilege execution roles.
- **Amazon CloudWatch:** Centralized logging and monitoring for Lambda and API Gateway.

## Virtualization Connection
Traditional Virtual Machines (EC2) abstract physical hardware, requiring full operational management of the guest operating system, runtime, and patches. Containers (ECS/Docker) abstract the operating system level, allowing applications to share a kernel while packaging dependencies together. Serverless computing (Lambda) abstracts compute resources entirely, allowing code to run on demand while AWS handles host management, scaling, and provisioning.

## Evidence Reviewed
- **Reporting Task:** Runs once daily, completes quickly, maintains no local state, sits idle 99% of the time.
- **REST API:** Receives low-volume, intermittent requests throughout the day.
- **Legacy Workload Requirements:** Requires continuous runtime, custom OS-level packages, and host-level access.
- **Resource Constraints:** Verified execution time and memory requirements fit within Lambda limits (15-min max execution time).

## Operational Analysis
- **Daily Report:** Best suited for **Serverless (AWS Lambda)** triggered by EventBridge. Paying for a continuously running EC2 instance for a brief daily job is inefficient.
- **REST API:** Best suited for **Serverless (API Gateway + AWS Lambda)**. Automatic scaling handles bursty or intermittent web traffic without maintaining idle web servers.
- **Host-Dependent App:** Best suited for a **Virtual Machine (Amazon EC2)**. The requirement for custom OS packages and host administrative access excludes serverless execution models.

## Recommendation
I recommend a hybrid compute architecture tailored to each workload's behavior:
1. **Migrate the Daily Report** to a stateless AWS Lambda function scheduled via Amazon EventBridge.
2. **Migrate the REST API** to Amazon API Gateway backed by AWS Lambda functions.
3. **Retain EC2 Virtual Machines** only for workloads requiring host-level OS access or custom kernel-level dependencies.

**Responsibility Shift:** AWS assumes management of server hardware, OS patching, capacity scaling, and runtime provisioning. HarborTech maintains responsibility for application code, IAM execution roles, API security rules, and CloudWatch log monitoring.

## Escalation Notes
The following actions must be escalated to senior HarborTech engineering prior to execution:
- Production API DNS migration and route cutovers in Route 53.
- Creation or modification of IAM execution roles and policy boundaries.
- VPC endpoint configurations for Lambda functions needing secure private database access.

## Lessons Learned
- Workload characteristics (timing, state, dependencies) must drive compute selection—not service popularity.
- Serverless eliminates idle infrastructure costs for scheduled or intermittent tasks.
- Abstracting infrastructure shifts operational maintenance to AWS, allowing teams to focus on code and security policies rather than host patching.

## Professional Vocabulary
- **Container:** A lightweight, standalone package containing application code and dependencies that shares the host OS kernel.
- **Serverless Computing:** A cloud execution model where the cloud provider dynamically manages machine resource allocation.
- **AWS Lambda:** An event-driven, serverless compute service that runs code in response to events.
- **API Gateway:** A managed service that enables developers to create, publish, maintain, monitor, and secure APIs.
- **Execution Role:** An IAM role assumed by an AWS service (like Lambda) granting permissions to access other AWS resources.
- **Event Trigger:** A state change or scheduled action that automatically invokes a serverless function.
- **Step Functions:** A serverless orchestration service used to sequence multiple AWS Lambda functions into workflows.
- **VPC-Connected Lambda:** A Lambda function configured to access private resources within an Amazon VPC.
- **Runtime:** The software environment (e.g., Python 3.12, Node.js) in which a Lambda function executes.
- **Abstraction:** The practice of hiding underlying technical complexity to simplify management and operation.
