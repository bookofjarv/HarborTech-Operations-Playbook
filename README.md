```markdown
# Week 2: IAM and AWS CLI Investigation

## HarborTech Ticket Summary
My team and I took on ticket TKT-2026-0002 for Riverside Goods to resolve an issue with inventory coordinator Marcus Webb. Marcus can sign into the AWS Management Console without any issues, but whenever he tries to work with inventory files in Amazon S3, he hits an `AccessDenied` wall. We investigated the root cause of the authorization failure, evaluated a risky quick-fix requested by the client, ran CLI diagnostic commands in CloudShell, and established a clear remediation strategy following least-privilege standards.

## Client Impact
Because Marcus is locked out of S3, he cannot view, download, or upload inventory report files in the `riverside-inventory` bucket. This blocks daily inventory syncs, stalls supply chain updates, and creates a operational bottleneck for the warehouse team until access is restored.

## AWS Services Involved
* **AWS Identity and Access Management (IAM):** Used to check Marcus's identity, group membership, attached policies, and permissions.
* **IAM Users, Groups, Roles, and Policies:** The structural components we analyzed to determine how permissions are evaluated.
* **Amazon Simple Storage Service (S3):** The target storage service holding the `riverside-inventory` bucket.
* **AWS CloudShell:** The browser-based CLI environment where we verified caller identity and tested command execution.
* **AWS Command Line Interface (AWS CLI):** Used to run diagnostic queries like `sts get-caller-identity` and `iam get-role`.
* **AWS Regions:** The regional context (`us-east-1`) executing our control plane commands.

## Virtualization Connection
Cloud infrastructure decouples physical hardware into software-defined compute, storage, and networking layers. Because everything in a virtualized cloud environment is controlled through APIs, IAM serves as the central control plane boundary. Without fine-grained permission rules, an identity could accidentally or intentionally modify virtual storage volumes, reconfigure virtual networks, or expose cloud data assets across the entire tenant.

## Evidence Reviewed
* **Evidence A (Authentication):** Marcus successfully logs into the Riverside Goods AWS console using his IAM credentials.
* **Evidence B (Business Need):** Marcus needs to list the `riverside-inventory` bucket, read report objects, and upload new inventory files. He does not need access to any other S3 buckets or EC2 resources.
* **Evidence C (Permission Record):** Marcus's IAM user exists, but has zero attached managed/inline policies and no job-function group membership.
* **Evidence D (Request Result):** Any attempt by Marcus to interact with the S3 bucket returns an explicit `AccessDenied` error.
* **Client Proposal:** The account owner requested attaching the AWS-managed policy `AmazonS3FullAccess` directly to Marcus so the ticket could be closed immediately.
* **CLI Investigation Evidence:** Output from `aws sts get-caller-identity`, `aws iam get-role`, `aws iam list-role-policies`, and test commands executed in CloudShell.

## Operational Analysis
The evidence clearly shows an authorization failure, not an authentication problem. Marcus's credentials are fully valid (Authentication), but AWS denies his API requests because there is no identity-based policy granting him permission to perform actions on S3 (Authorization). 

The account owner's proposal to slap `AmazonS3FullAccess` onto Marcus's account is a dangerous quick-fix. `AmazonS3FullAccess` grants full administrative control—including deleting buckets, changing bucket policies, and modifying data—across every single S3 bucket in the entire AWS account. This massively violates the Principle of Least Privilege and opens up huge operational risks for Riverside Goods.

## Recommendation
We recommend rejecting the client's proposal to grant `AmazonS3FullAccess`. Instead, we should create a custom, least-privilege IAM policy strictly scoped to the `riverside-inventory` resource (`arn:aws:s3:::riverside-inventory` and `arn:aws:s3:::riverside-inventory/*`).

This policy should grant only the specific S3 API actions Marcus needs for his job:
* `s3:ListBucket` (Bucket level: allows listing objects in the inventory bucket)
* `s3:GetObject` (Object level: allows reading inventory reports)
* `s3:PutObject` (Object level: allows uploading updated inventory files)

To keep access management clean and scalable, this policy should be attached to an `Inventory-Coordinators` IAM group rather than attached directly to Marcus's user identity.

## Escalation Notes
**Finding:** Marcus is authenticated properly, but lacks an authorization policy to perform S3 actions.  
**Required Access Direction:** Deploy a custom least-privilege policy restricting S3 actions (`s3:ListBucket`, `s3:GetObject`, `s3:PutObject`) strictly to `riverside-inventory`.  
**Implementation Decision:** Determine whether to attach this policy via an `Inventory-Coordinators` IAM group or directly to the user based on HarborTech's client account standards.  
**Escalation Reason:** The actual policy creation and attachment in the client's production account must be reviewed and applied by an authorized HarborTech senior administrator. Tier-1 support and interns should not apply unvetted production permission changes directly, preserving change management audit trails and preventing accidental privilege escalation.

## Learner Lab CLI Evidence
To practice running IAM investigations, my team and I executed commands in CloudShell within our sandbox environment. Here is what we captured:

```bash
$ aws sts get-caller-identity
{
    "UserId": "AROA47DMA3EUH5P2UFBEB:user4742512=Jarvis_D._Anderson",
    "Account": "891432261928",
    "Arn": "arn:aws:sts::891432261928:assumed-role/voclabs/user4742512=Jarvis_D._Anderson"
}

```

*Analysis:* This command verifies my active session context. It proves CloudShell is executing under my federated student account (`Jarvis_D._Anderson`) and gives us our target Account ID (`891432261928`).

```bash
$ aws iam get-role --role-name LabRole
{
    "Role": {
        "Path": "/",
        "RoleName": "LabRole",
        "RoleId": "AROA47DMA3EUKATZTLYHW",
        "Arn": "arn:aws:iam::891432261928:role/LabRole",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": [
                            "cognito-idp.amazonaws.com",
                            "deepracer.amazonaws.com",
                            "s3.amazonaws.com",
                            "ec2.application-autoscaling.amazonaws.com"
                        ]
                    }
                }
            ]
        }
    }
}

```

*Analysis:* This pulled the JSON definition for `LabRole`. The `AssumeRolePolicyDocument` defines the trust relationship—specifying which AWS services (like S3 or EC2) are allowed to assume `LabRole`. It does not grant any resource permissions by itself.

```bash
$ aws iam create-user --user-name test-user
aws: [ERROR]: An error occurred (AccessDenied) when calling the CreateUser operation: User: arn:aws:sts::891432261928:assumed-role/voclabs/user4742512=Jarvis_D._Anderson is not authorized to perform: iam:CreateUser on resource: arn:aws:iam::891432261928:user/test-user because no identity-based policy allows the iam:CreateUser action

```

*Analysis:* Attempting to create an IAM user triggered an explicit `AccessDenied` error. This proves that top-level Service Control Policies (SCPs) and permissions boundaries actively restrict administrative identity creation in our sandbox.

*CLI vs. Console Behavior:* When we tried attaching `AmazonS3ReadOnlyAccess` to `LabRole` in the AWS Management Console (`IAM > Roles > LabRole > Add Permissions`), the UI blocked us with a red banner stating `Failed to attach policy to role... is not authorized to perform: iam:AttachRolePolicy`. This confirms that both the CLI and Console evaluate permissions using the exact same IAM engine.

*Trust Relationship vs. Permission Sources:* A trust policy controls *who* can assume a role (the trust relationship). Permission policies control *what* that role can actually do once assumed. `LabRole` should never be assigned to Marcus because it is a generic service execution role meant for automated AWS tasks, not a human identity. Assigning `LabRole` to Marcus would violate least privilege and destroy auditability.

## Lessons Learned

* **Authentication vs. Authorization:** Valid credentials only get you through the front door; identity-based policies dictate what tools you can touch once inside.
* **Resisting Quick Fixes:** Clients often ask for broad admin policies (`AmazonS3FullAccess`) to close tickets fast. Part of our job is pushing back and applying least privilege to protect their environment.
* **CLI Diagnostics:** The CLI gives precise error messages (`iam:CreateUser` denied on resource X), making it much faster to debug permission gaps than vague UI banners.
* **Role Governance:** Never reuse background service roles (`LabRole`) for human users just to bypass permission errors.

## Professional Vocabulary

* **Authentication:** Verifying the identity of a user or system based on credentials (e.g., logging in with a username and password).
* **Authorization:** Evaluating permissions to determine if an authenticated identity is allowed to perform a specific action on a resource.
* **IAM (Identity and Access Management):** The AWS service used to manage identities, access credentials, and permission policies.
* **Policy:** A JSON document that formally defines permissions (Allows or Denies) for AWS actions, resources, and conditions.
* **Least Privilege:** The security best practice of granting an identity only the exact permissions needed to perform its job, and nothing more.
* **AccessDenied:** The standard error returned by AWS when an identity attempts an API action that is not explicitly allowed by a policy.
* **AWS CLI:** A command-line utility used to issue API calls directly to AWS services.
* **CloudShell:** A browser-based terminal environment pre-authenticated with the user's console session credentials.
* **Caller Identity:** Metadata returned by STS showing the active account ID, ARN, and user session executing commands.
* **Resource Scope:** Defining exact ARNs in a policy statement to limit access to specific resources (e.g., one bucket) rather than using wildcards (`*`).

```

```
