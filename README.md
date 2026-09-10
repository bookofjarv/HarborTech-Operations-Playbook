# HarborTech Operations Playbook: Week 1 Onboarding

## HarborTech Ticket Summary
* **Task:** Week 1 Cloud Operations Intern Onboarding & Environment Readiness Check
* **Status:** Complete
* **Environment:** AWS Academy Learner Lab / Blackboard / GitHub

## Client Impact
* Operational readiness verification prevents access delays and account suspensions prior to taking live client tickets.
* Understanding Learner Lab boundaries ensures non-disruptive operations and budget compliance.

* <img width="1434" height="675" alt="image" src="https://github.com/user-attachments/assets/6bb5d8c1-4c59-42f4-a7a1-70ff205f9506" />


## AWS Services Involved
* **AWS IAM:** Verified pre-created `LabRole` and `LabInstanceProfile`.
* **Amazon S3 & EC2:** Reviewed official documentation for launching virtual machines and managing cloud storage.
* **AWS Budgets:** Identified the 8-to-12-hour reporting lag to avoid account disablement.

## Virtualization Connection
* Reviewed procedures for deploying and managing cloud-hosted virtual machines (Linux instances) within an AWS sandbox infrastructure.

## Evidence Reviewed
* **Access Confirmation:** Successfully logged into AWS Academy, Learner Lab, and GitHub.
* **Permitted Regions:** Confirmed operating boundary limited to `us-east-1` and `us-west-2`.
* **IAM Boundaries:** Confirmed inability to create IAM users or groups.
* **Session & Reset Rules:** Timer reaching zero ends active session but preserves resources; Reset permanently deletes resources without restoring budget.

* <img width="1520" height="827" alt="image" src="https://github.com/user-attachments/assets/aae104a9-7f71-4b22-b8ff-a90e2b8fdaff" />


## Operational Analysis
* The Learner Lab provides a restricted sandbox environment requiring careful resource monitoring due to delayed budget reporting.
* Operational skills rely on referencing authoritative AWS documentation rather than relying on memorization.

## Recommendation
* Always verify the active AWS Region (`us-east-1` or `us-west-2`) before deploying services.
* Avoid using the Reset function unless instructed, as it does not refresh the budget quota.

## Escalation Notes
* **Unresolved Issues:** None.
* *(If you had an issue, you would record: exact error message, timestamp, platform, and steps attempted.)*

## Lessons Learned
1. Expected restrictions (like IAM creation blocks) are intentional sandbox boundaries, not platform errors.
2. Official documentation serves as the durable reference when service interfaces or features update over time.

## Professional Vocabulary
* **Learner Lab Sandbox:** A restricted AWS account environment designated for educational lab exercises with strict budget and IAM boundaries.
* **IAM Role (`LabRole`):** A pre-configured identity with specific permissions assigned to manage authorized AWS services within the lab.
