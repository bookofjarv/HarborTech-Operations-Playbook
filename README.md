# HarborTech Operations Playbook: Week 2 IAM & CLI Investigation

## HarborTech Ticket Summary

* **Ticket ID:** TKT-2026-0002
* **Client:** Riverside Goods
* **Assigned Personnel:** Jarvis D. Anderson (Cloud Operations Intern) & Team
* **Task:** Investigate S3 authorization failure for Inventory Coordinator Marcus Webb
* **Status:** Complete / Escalated for Production Implementation
* **Environment:** AWS Management Console / AWS CloudShell / AWS CLI

## Client Impact

* Marcus Webb can sign into the AWS Management Console but is completely blocked from accessing S3 bucket resources.
* The authorization failure prevents Marcus from viewing, reading, or uploading inventory report objects in the `riverside-inventory` bucket.
* This halts daily inventory synchronization, stalls supply chain tracking updates, and disrupts downstream warehouse operations until access is restored.

<img width="1821" height="762" alt="lab role console permissions denied" src="https://github.com/user-attachments/assets/4fc65c60-747d-4cb6-add3-b3e785eaf04b" />


## AWS Services Involved

* **AWS Identity and Access Management (IAM):** Used to evaluate identity verification, role assumptions, group structures, and policy evaluation logic.
* **IAM Identities & Policies:** Analyzed user objects, job-function groups, trust policies, and identity-based permissions documents.
* **Amazon Simple Storage Service (S3):** The target cloud storage service hosting the `riverside-inventory` bucket.
* **AWS CloudShell:** The pre-authenticated, browser-based CLI environment used to inspect session contexts and run API operations.
* **AWS Command Line Interface (AWS CLI):** Command-line tool used to run STS identity checks, query IAM role configurations, and document error responses.
* **AWS Regions:** Regional control plane boundary (`us-east-1`) executing our CLI API calls.


## Evidence Reviewed

* **Evidence A (Authentication):** Marcus successfully authenticates to the Riverside Goods console, proving valid credentials.
* **Evidence B (Business Requirement):** Marcus requires permission to list `riverside-inventory`, read report objects, and upload new inventory files. He does not administer EC2 or other S3 buckets.
* **Evidence C (Permission Record):** Onboarding records confirm Marcus's IAM user exists, but has no job-function group membership and zero directly attached policies.
* **Evidence D (Request Result):** Querying the inventory bucket returns an explicit `AccessDenied` exception.
* **Client Proposal:** The account owner requested attaching `AmazonS3FullAccess` directly to Marcus's user account to close the ticket quickly.
* **CLI Terminal Evidence:** Ran diagnostic queries (`aws sts get-caller-identity`, `aws iam get-role`, `aws iam list-role-policies`) in CloudShell to test IAM evaluation behavior.

<img width="1847" height="828" alt="lab role 1" src="https://github.com/user-attachments/assets/67a9c10b-1a5a-4b77-88b0-76feecda2900" />



## Operational Analysis

* **Authentication vs. Authorization:** The evidence demonstrates an authorization gap, not an authentication issue. Successful sign-in proves Marcus's identity is authenticated, while the `AccessDenied` result proves he lacks identity-based policies authorizing S3 actions.
* **Evaluating the Client Proposal:** The client's request to grant `AmazonS3FullAccess` must be rejected. `AmazonS3FullAccess` grants full administrative rights across every S3 bucket in the account (including deleting buckets and altering security policies). This grossly violates the Principle of Least Privilege and introduces severe operational risk.
* **CLI vs. Console Consistency:** Testing policy attachment in the console resulted in a red `iam:AttachRolePolicy` denial banner, matching the explicit `iam:CreateUser` `AccessDenied` CLI output. Both interfaces enforce the exact same backend IAM evaluation engine.

<img width="1821" height="762" alt="lab role console permissions denied" src="https://github.com/user-attachments/assets/215eeefa-8e4f-43f3-afb0-7cc3f8f56cda" />


## Recommendation

Reject the `AmazonS3FullAccess` request. Instead, create a custom, least-privilege customer-managed policy restricted strictly to the `riverside-inventory` resource (`arn:aws:s3:::riverside-inventory` and `arn:aws:s3:::riverside-inventory/*`).

The policy must grant only the three conceptual S3 actions Marcus needs:

* `s3:ListBucket` (Bucket level: to view bucket contents)
* `s3:GetObject` (Object level: to read inventory reports)
* `s3:PutObject` (Object level: to upload updated inventory files)

Attach this policy to an possible group named `Inventory-Coordinators` IAM group rather than directly to Marcus's user account.
## Escalation Notes

* **Supported Finding:** Marcus is properly authenticated, but lacks authorization policies required for inventory tasks.
* **Required Access Direction:** Implement a scoped least-privilege policy allowing `s3:ListBucket`, `s3:GetObject`, and `s3:PutObject` strictly on `riverside-inventory`.
* **Implementation Decision:** Determine whether to deploy via an `Inventory-Coordinators` group or direct attachment based on HarborTech tenant standards.
* **Escalation Safeguard:** The production IAM policy change must be reviewed and deployed by an authorized senior HarborTech administrator rather than an intern. This maintains strict change management controls, prevents unvetted privilege escalation, and preserves audit trails.

## Learner Lab CLI Evidence

During sandbox investigation in CloudShell, my team and I executed and documented the following terminal commands:

```bash
$ aws sts get-caller-identity
{
    "UserId": "AROA47DMA3EUH5P2UFBEB:user4742512=Jarvis_D._Anderson",
    "Account": "891432261928",
    "Arn": "arn:aws:sts::891432261928:assumed-role/voclabs/user4742512=Jarvis_D._Anderson"
}

```

* **Meaning:** Confirms my active session context (`Jarvis_D._Anderson`) under account ID `891432261928`. It proves CloudShell executes under my federated student identity rather than directly as `LabRole`.

```bash
$ aws iam get-role --role-name LabRole
{
    "Role": {
        "Path": "/",
        "RoleName": "LabRole",
        "RoleId": "AROA47DMA3EUKATZTLYHW",
        "Arn": "arn:aws:iam::891432261928:role/LabRole",
        "AssumeRolePolicyDocument": { ... }
    }
}

```

* **Meaning:** Retrieves the raw JSON metadata for `LabRole`. The `AssumeRolePolicyDocument` defines the trust relationship, specifying which AWS service principals (e.g., `ec2.amazonaws.com`, `s3.amazonaws.com`) are trusted to assume the role.

```bash
$ aws iam create-user --user-name test-user
aws: [ERROR]: An error occurred (AccessDenied) when calling the CreateUser operation: User: arn:aws:sts::891432261928:assumed-role/voclabs/user4742512=Jarvis_D._Anderson is not authorized to perform: iam:CreateUser on resource: arn:aws:iam::891432261928:user/test-user because no identity-based policy allows the iam:CreateUser action

```

* **Meaning:** Demonstrates that administrative identity creation is explicitly blocked by sandbox control policies.
* **Trust Relationship vs. Permission Sources:** A trust policy defines *who* can wear a role (trust relationship). Permission policies define *what* actions that role can perform once assumed (permission sources).
* **Why `LabRole` Is Not the Solution for Marcus:** `LabRole` is a generic service execution role designed for automated AWS service workloads. Recommending `LabRole` for Marcus violates least privilege by exposing broad service permissions beyond his job duties, destroys auditability by masking individual user identity, and fails to scope access to the `riverside-inventory` bucket.

## Lessons Learned

1. **Authentication ≠ Authorization:** Logging into the AWS console only verifies who you are; permission policies control what resources you can interact with.
2. **Resisting Fast Workarounds:** Over-permissioning identities with managed admin policies (`AmazonS3FullAccess`) to close tickets quickly introduces severe operational and security risks.
3. **Diagnostic CLI Precision:** CLI outputs provide exact API exception names (`iam:CreateUser`), making root-cause analysis much faster than troubleshooting generic console UI banners.
4. **Role Integrity:** Service roles (`LabRole`) should never be assigned to human users as a shortcut to bypass permission boundaries.

## Professional Vocabulary

* **Authentication:** Verifying the identity of a user or principal using valid credentials (e.g., username/password).
* **Authorization:** Evaluating permissions to determine if an authenticated identity is allowed to perform a requested action on a resource.
* **IAM (Identity and Access Management):** The AWS web service used to manage identities, credentials, and access permissions.
* **Policy:** A JSON document formally defining permissions (Allow or Deny) for actions, resources, and conditions.
* **Least Privilege:** The security standard of granting an identity only the minimum permissions necessary to perform its job.
* **AccessDenied:** The standard AWS error returned when an identity attempts an action not explicitly granted by an identity-based or resource-based policy.
* **AWS CLI:** A unified command-line tool used to make direct API calls to AWS services.
* **CloudShell:** A browser-based terminal environment pre-authenticated with the active console user's credentials.
* **Caller Identity:** Metadata returned by STS revealing the active AWS account, user ARN, and session identity.
* **Resource Scope:** Restricting policy actions to specific AWS resource ARNs (e.g., `arn:aws:s3:::riverside-inventory/*`) rather than using wildcard wildcards (`*`).
