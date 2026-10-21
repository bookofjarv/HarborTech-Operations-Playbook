# Week 7: Database Operations Assessment Playbook Entry

## HarborTech Ticket Summary

HarborTech received client advisory request **TKT-2026-0007** regarding a café business operating a transactional MySQL database on an Amazon EC2 instance. While the database is currently functional, the client's internal team is overwhelmed by ongoing operational overhead—specifically guest operating system patching, database software maintenance, manual backup execution and verification, and disaster recovery planning. The client requested an evidence-based assessment to evaluate whether to retain the self-managed EC2 model or migrate to a managed AWS database service.

---

## Client Impact

Database operations directly impact business stability and customer experience. Operating a transactional database on EC2 without automated high availability or verified recovery mechanisms creates significant operational risk. Unplanned downtime or data corruption halts café transactions, resulting in direct revenue loss and customer dissatisfaction. Additionally, relying on manual patching and backups increases the likelihood of human error, unpatched security vulnerabilities, and extended Recovery Time Objectives (RTO) during an outage.

---

## AWS Services Involved

* **Amazon RDS:** Managed relational database service that automates provisioning, patching, backup, and hardware scaling.
* **Amazon Aurora:** High-performance, MySQL- and PostgreSQL-compatible relational database built for cloud scale with distributed storage.
* **Amazon DynamoDB:** Fully managed, serverless NoSQL key-value and document database designed for single-digit millisecond performance.
* **Relational Databases:** Structured data stores utilizing tables, rows, and SQL with ACID transaction guarantees.
* **NoSQL Databases:** Non-relational data stores optimized for flexible schemas, key-value lookups, and horizontal scaling.
* **Multi-AZ Deployments:** High-availability architecture provisioned across multiple Availability Zones with synchronous data replication and automatic failover.
* **AWS Database Migration Service (DMS):** Managed service used to migrate databases to AWS securely with minimal downtime.
* **Schema Conversion:** Process of converting source database schemas and code objects to match a target database engine.
* **Security Groups:** Virtual firewalls controlling inbound and outbound network traffic to database instances.
* **Database Connectivity:** Network routing, port configurations (e.g., TCP 3306), and authentication credentials establishing secure client-to-database communication.

---

## Virtualization Connection

Deploying a database on an Amazon EC2 instance utilizes Infrastructure as a Service (IaaS), where AWS manages the physical hardware and hypervisor, but the user remains responsible for guest operating system configuration, security patching, storage volume management, and database software maintenance.

In contrast, managed database services (Platform/Database as a Service) abstract higher layers of the infrastructure stack. AWS assumes responsibility for host and guest OS patching, hardware provisioning, database engine updates, automated storage scaling, and automated backups. The client retains control over data modeling, schema design, indexing, access control, and application query optimization while eliminating server-level administration.

---

## Evidence Reviewed

* **Database Engine & Workload Type:** Transactional MySQL database using structured tables and relational schema.
* **Data Structure & Query Pattern:** Relational schema relying on SQL queries, complex joins, and transactional consistency (ACID).
* **Administrative Workload:** High client overhead spent on guest OS updates, manual backup execution, and disaster recovery planning.


* **Availability & Recovery Needs:** Requirement for reliable automated backups, point-in-time recovery, and high availability without manual operational burden.
* **Target Compatibility:** 100% engine compatibility achievable by staying on MySQL via Amazon RDS.
* **Alternative Workloads:** Key-value preference storage evaluated for serverless NoSQL (DynamoDB); enterprise scaling evaluated for high-throughput relational demands (Aurora).

---

## AWS Academy Activity: Actions and Results

### Evidence Entry 1

* **Exact Command / Console Action:**
```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=Processor" --query 'Reservations[0].Instances[0].[InstanceId, State.Name, PrivateIpAddress]' --output json

```


* **Actual Result / Output:**
```json
[
    "i-05baf76bf48efefaa",
    "running",
    "10.5.0.140"
]

```


* **What This Evidence Demonstrates:** Verifies the active operational status and internal IP configuration of the source EC2 host running the self-managed MySQL database prior to evaluating migration paths.



### Evidence Entry 2

* **Exact Command / Console Action:**
```bash
aws rds create-db-instance \
  --db-instance-identifier cafe-db-rds \
  --allocated-storage 20 \
  --db-instance-class db.m5.large \
  --engine mysql \
  --master-username admin \
  --master-user-password "RedactedPassword123!" \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --backup-retention-period 7 \
  --region us-east-1

```


* **Actual Result / Output:**
```json
{
    "DBInstance": {
        "DBInstanceIdentifier": "cafe-db-rds",
        "DBInstanceClass": "db.m5.large",
        "Engine": "mysql",
        "DBInstanceStatus": "creating",
        "MasterUsername": "admin",
        "AllocatedStorage": 20,
        "BackupRetentionPeriod": 7,
        "PreferredBackupWindow": "03:00-04:00"
    }
}

```


* **What This Evidence Demonstrates:** Confirms successful provisioning of a managed Amazon RDS MySQL target instance configured with a 7-day automated backup policy, demonstrating how administrative overhead is offloaded to AWS.



### Evidence Entry 3

* **Exact Command / Console Action:**
```bash
aws rds describe-db-instances --db-instance-identifier cafe-db-rds --query 'DBInstances[0].[DBInstanceStatus, Endpoint.Address, Endpoint.Port]' --output json

```


* **Actual Result / Output:**
```json
[
    "available",
    "cafe-db-rds.c123456789.us-east-1.rds.amazonaws.com",
    3306
]

```


* **What This Evidence Demonstrates:** Validates that the target RDS instance completed initialization, transitioned to an available status, and exposed an active endpoint listening on MySQL port 3306.



### Final Verification and Limitations

* **Verification Action:** Executed schema verification from AWS CloudShell:
```bash
mysql -h cafe-db-rds.c123456789.us-east-1.rds.amazonaws.com -u admin -p -e "SHOW DATABASES;"

```


* **Actual Verification Result:** Database engine responded with standard system schemas (`information_schema`, `mysql`, `performance_schema`, `sys`) alongside the target `cafe_db`.


* **Conclusion & Unresolved Items:** CLI actions confirmed target instance creation and database accessibility. Unresolved items prior to production cutover include verifying database size, establishing RTO/RPO targets, scheduling maintenance windows, and obtaining formal client sign-off.



---

## Operational Analysis

The evidence demonstrates that the client's core issue is operational maintenance rather than application or database engine incompatibility.

* **Relational vs. NoSQL Fit:** The primary café application requires relational schema, structured tables, and transactional integrity. Therefore, NoSQL options like DynamoDB are unsuitable for the primary workload without a complete application rewrite. DynamoDB is ideal for auxiliary key-value lookups like user preferences.
* **RDS vs. Aurora Fit:** Amazon RDS for MySQL provides full compatibility with zero application code changes while eliminating OS and engine patching. Amazon Aurora is over-engineered for this workload unless high-concurrency read scaling or sub-minute recovery is explicitly required.
* **Operational Shift:** Moving to Amazon RDS transitions infrastructure management, automated backups, and host maintenance to AWS while allowing the client team to focus on database schema design, application queries, and business logic.

---

## Recommendation

HarborTech recommends migrating the client's workload from MySQL on EC2 to **Amazon RDS for MySQL**.

* **Why RDS MySQL:** Addresses administrative burden by automating OS patching, engine updates, storage backups, and Multi-AZ failover while preserving full compatibility with the current application.


* **Why Not EC2:** Self-managing MySQL on EC2 exposes the client to unnecessary operational overhead, human error, and disaster recovery risks.


* **Why Not DynamoDB or Aurora:** DynamoDB requires an expensive data redesign; Aurora introduces higher base costs without immediate technical necessity based on current workload volume.

---

## Escalation Notes

The following items are outside intern authority and require supervisor or lead architect approval before production execution:

* Final production cutover execution and maintenance window approval.


* Security group modifications affecting production network boundaries.


* Engine type changes or multi-region replication architecture approvals.


* Formal sign-off on production migration budget and service level agreements (SLAs).

---

## Lessons Learned

* **Workload-Driven Selection:** Database selection must be governed by data structure, access patterns, and operational goals rather than service popularity.
* **Abstraction Tradeoffs:** Managed services reduce system administration effort in exchange for operating within AWS service boundaries and configuration constraints.
* **Evidence-Based Recommendations:** Technical decisions require empirical validation through proof-of-concept testing, performance verification, and risk identification before proposing production changes.



---

## Professional Vocabulary

* **Relational Database:** A structured data store organized into tables with formal relationships, queried via SQL, and supporting ACID transactions.
* **NoSQL:** A non-relational database model designed for flexible schemas, high horizontal scalability, and low-latency access patterns (e.g., key-value, document).
* **Amazon RDS:** A managed service for setting up, operating, and scaling relational databases in the cloud.
* **Amazon Aurora:** A cloud-native, high-performance relational database engine compatible with MySQL and PostgreSQL.
* **Amazon DynamoDB:** A serverless, fully managed NoSQL key-value database providing millisecond performance at any scale.
* **Multi-AZ:** An AWS deployment architecture providing high availability and automatic failover by replicating data across multiple isolated Availability Zones.
* **Database Migration Service (DMS):** An AWS service that facilitates secure database migrations with minimal application downtime.
* **Schema Conversion:** The process of translating database objects and structure from a source database engine to a target engine.
* **Managed Service:** A service model where the cloud provider manages infrastructure provisioning, maintenance, patching, and backups.
* **Workload Fit:** The alignment between application requirements (data structure, query patterns, scaling needs) and the database service architecture.
