# Week 3: Systems Manager and S3

## HarborTech Ticket Summary
In TKT-2026-0003, I investigated two operational challenges reported by Bright Path Community Services:
1. **Unautomated Server Maintenance:** Dana has been manually performing weekly administrative tasks and updates across five individual EC2 instances, leading to operational overhead and consistency risks.
2. **Unnecessary Infrastructure Overhead:** Bright Path needed a way to host public community resources (program schedules, contact details, downloadable forms). Hosting static files on a dedicated compute server introduces unnecessary expenses.

## Client Impact
Relying on manual, repetitive administration across multiple servers negatively impacts efficiency, and operational consistency. Manual updates often lead to human error as well as missed security patches. Additionally, running a dedicated virtual machine for simple static content wastes engineering effort on OS patching and maintenance, diverting time and budget away from Bright Path's core mission.

## AWS Services Involved
- **AWS Systems Manager (SSM):** A centralized operational management service for AWS resources and virtual machines.
- **Run Command:** Enables remote, automated script execution across instance fleets at scale without direct terminal access or open SSH ports.
- **Session Manager:** Provides secure, browser-based interactive shell access without requiring bastion hosts or open inbound network ports.
- **Inventory:** Automatically gathers software, operating system, and patch metadata across managed instances.
- **Parameter Store:** Centralizes secure storage for configuration data, strings, and secrets.
- **Amazon S3:** Scalable, highly available object storage used to store static content.
- **Static Website Hosting:** An S3 feature that serves static web files directly to visitors over HTTP without needing web application servers.

## Virtualization Connection
AWS Systems Manager serves as a centralized management plane for virtual machines. Instead of managing each VM individually through SSH/RDP or direct network connections, Systems Manager uses an agent to allow central oversight, batch maintenance, and security policy enforcement across virtual compute assets.

For workloads that do not require server-side code execution or active database processing, Amazon S3 replaces traditional virtual servers altogether. Using S3 for static content offloads compute, OS updates, and web server scaling directly to AWS infrastructure.

## Evidence Reviewed
During my investigation, I analyzed several key technical and environment factors:
- **Instance Count & Tasks:** Five EC2 instances requiring repetitive weekly software and patch updates.
- **Interactive vs. Non-Interactive Needs:** Automated patch runs require non-interactive execution (Run Command), while occasional troubleshooting requires secure, audited interactive sessions (Session Manager).
- **Configuration Management:** Dana’s application hardcodes database connection strings across multiple configuration files on two instances, indicating a need for centralized parameters.
- **Static Web Requirements:** Bright Path's public landing page consists of static assets (`index.html`, CSS, static files) that require no backend computing.
- **AWS Environment Evidence:** I deployed and verified the S3 static website infrastructure using the AWS CLI and S3 Management Console.

### Environment Deployment Evidence

#### 1. CLI Caller Identity & Execution
I verified my IAM identity and confirmed execution status in CloudShell using the AWS CLI.

<img width="883" height="120" alt="get caller id command" src="https://github.com/user-attachments/assets/c3cb9faa-f039-4c5d-971b-522eb1404280" />


#### 2. S3 Bucket Object Upload
I confirmed that `index.html` was successfully uploaded and stored in the root directory of bucket `brightpath-public-site-891432261928`.

<img width="869" height="162" alt="get bucket policy" src="https://github.com/user-attachments/assets/17ecda54-2fc6-4e85-9a44-f783beae184e" />


#### 3. Static Website Hosting Configuration
I verified that static website hosting was enabled on bucket `brightpath-public-site-891432261928` in region `us-east-1` and noted the live website endpoint URL.

<img width="878" height="110" alt="website endpoint" src="https://github.com/user-attachments/assets/cc0eb842-d1b1-4f9b-bf81-72101e98cf7f" />

<img width="925" height="643" alt="static website hosting 2" src="https://github.com/user-attachments/assets/bb8eb968-3b71-4165-8e1f-261d6595d8de" />



#### 4. Bucket Policy Verification
I checked the applied bucket policy using the CLI (`aws s3api get-bucket-policy`) to verify public read access permissions (`s3:GetObject`).

<img width="869" height="162" alt="get bucket policy" src="https://github.com/user-attachments/assets/069b7e6f-8e4c-484a-8f9a-f65b57f6a53d" />


#### 5. Live Website Access Test
I tested endpoint reachability by navigating to the S3 website URL in a web browser to verify that the public landing page renders properly.

<img width="820" height="636" alt="brightpath website in browser" src="https://github.com/user-attachments/assets/7fa25344-544c-4fb6-abf1-5af539fa6c9f" />


## Operational Analysis
Centralized automation is far better suited for multi-instance maintenance than manual administrative logins. Tasks like operating system updates and patch enforcement should be automated via Systems Manager Maintenance Windows and Run Command to guarantee identical configuration across all five instances.

For hardcoded parameters, moving database connection strings to Parameter Store removes sensitivity from application code. However, storing values in Parameter Store requires updating application code to pull variables dynamically via the AWS SDK during runtime.

For web delivery, Amazon S3 is the optimal choice for Bright Path’s current static landing page. Hosting static files on S3 removes the need for web server compute, eliminates OS patching, and reduces hosting costs. If Bright Path eventually adds dynamic user authentication or server-side workflows, the workload can transition to AWS Lambda and API Gateway or an EC2 application tier behind CloudFront.

## Recommendation
I recommend the following AWS configuration for Bright Path:
1. **Automated Instance Maintenance:** Implement **AWS Systems Manager Maintenance Windows** combined with **Run Command** to execute automated maintenance and patching across all five instances on a predictable schedule.
2. **Software Inventory Tracking:** Enable **Systems Manager Inventory** to maintain real-time visibility into OS patches and installed packages.
3. **Centralized Configuration:** Store connection parameters in **Parameter Store** (`/brightpath/production/database/endpoint`) and refactor application startup scripts to fetch this value dynamically.
4. **Static Resource Hosting:** Host the public landing page on **Amazon S3 Static Website Hosting** (`brightpath-public-site-891432261928`).

## Escalation Notes
- **IAM Instance Profiles:** To allow Systems Manager to manage the five EC2 instances, an IAM instance profile containing the `AmazonSSMManagedInstanceCore` policy must be attached to each instance. If permissions are missing, an IAM administrator escalation is required.
- **Sandbox Control Policies:** In sandbox environments, Service Control Policies (SCPs) may block direct bucket policy modifications or public access changes. An `AccessDenied` response on public policy creation indicates enforced sandbox guardrails and requires administrative escalation if full public access is required in production.

## Lessons Learned
This investigation highlighted the power of separating management capabilities from network access. Using Systems Manager eliminates the security risk of open SSH/RDP ports while enforcing operational standardization across compute fleets. Additionally, selecting the right service model—like swapping dedicated virtual servers for S3 static hosting—dramatically cuts operational overhead, maintenance burdens, and cloud expenditure.

## Professional Vocabulary
- **Systems Manager:** An AWS management service that provides central operational visibility and control across cloud and on-premises resources.
- **Managed Node:** An EC2 instance or virtual machine configured with the SSM Agent, allowing it to be managed centrally by AWS Systems Manager.
- **Run Command:** A Systems Manager feature that enables safe, remote script execution across managed nodes without requiring direct terminal logins.
- **Session Manager:** A capability providing fully audited, browser-based interactive shell access to managed instances without open inbound network ports.
- **Inventory:** A Systems Manager feature that automatically collects system, application, and patch metadata across instance fleets.
- **Parameter Store:** A secure, centralized storage service for configuration data, database connection strings, and secrets.
- **Automation:** A Systems Manager feature used to simplify and automate complex maintenance playbooks and administrative routines.
- **Static Website Hosting:** An Amazon S3 feature that allows a bucket to directly serve static web content (HTML, CSS, images) via an HTTP URL.
- **Object Storage:** A storage architecture that stores data as individual objects with metadata and unique identifiers, optimized for unstructured files and static web content.
- **Management Plane:** The architectural layer and API interface used to manage, configure, and monitor systems independently of user data traffic paths.
