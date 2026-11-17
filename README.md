# Week 11: Monitoring and Change Evidence

## HarborTech Ticket Summary
This investigation addresses HarborTech Ticket **TKT-2026-0011** for the client **Riverside Goods**. The client reported sudden website performance degradation following an unannounced change window. Specifically, the web tier Auto Scaling group (ASG) capacity dropped unexpectedly, leaving insufficient resources to handle incoming web traffic and triggering high-latency alarms.

<img width="773" height="638" alt="image" src="https://github.com/user-attachments/assets/49c479f7-3b74-4dd6-9a85-5e8be158001e" />


## Client Impact
Unapproved infrastructure changes combined with delayed detection directly impact business continuity and customer trust. When capacity changes go undocumented or unmonitored:
* **Availability Drops:** Under-provisioned web tiers lead to request queuing, high error rates, and connection timeouts.
* **Support Drag:** Operations teams waste critical incident-response time hunting for root causes without a reliable change history.
* **Security Blind Spots:** Unverified role sessions modifying production resources without change tickets expose the environment to unauthorized alterations and compliance risks.

## AWS Services Involved
* **Amazon CloudWatch Metrics:** Used to track system-level and custom performance telemetry, such as target response times.
* **Amazon CloudWatch Logs:** Captured application-tier execution warnings, upstream response times, and healthy target counts.
* **Amazon CloudWatch Alarms:** Automated the threshold evaluation to transition the operational status from `OK` to `ALARM`.
* **AWS CloudTrail:** Recorded management-plane API activity and actor identity context.
* **AWS Config:** Tracked resource configuration changes and historical state transitions (before-and-after states).

## Virtualization Connection
In cloud-native, software-defined infrastructure, servers, networks, and scaling policies are managed via APIs rather than physical adjustments. Because physical hardware visibility is abstracted away, operations teams depend entirely on unified telemetry, immutable audit logs (`CloudTrail`), and configuration timelines (`AWS Config`) to reconstruct what changed, when it changed, and who initiated it.

## Evidence Reviewed
The operational evidence packet collected for TKT-2026-0011 comprises five sequential records:
* **CloudTrail Management Event:** Captured an `UpdateAutoScalingGroup` API call executed by a role session (`RG-WebOps-Role`), modifying desired and maximum capacity parameters.
* **AWS Config Resource History:** Recorded the precise configuration change showing the ASG desired capacity dropping from `3` to `1` and maximum capacity dropping from `4` to `1`.
* **CloudWatch Target Response Time Metric:** Indicated an abrupt performance spike, with latency jumping from `0.3` seconds to `2.4` seconds.
* **CloudWatch Application Logs (`/rg/web/app`):** Logged upstream response warnings averaging `2.38` seconds and confirmed only `1` healthy target remaining behind the load balancer.
* **CloudWatch Alarm (`rg-web-latency-high`):** Transitioned state from `OK` to `ALARM` after crossing the latency threshold (>1s) for two consecutive evaluation periods.

## Operational Analysis
* **Incident Window:** The incident degradation window opened at **14:05** with the configuration modification and culminated in an active alarm state at **14:09**.
* **Reconstructed Timeline:**
  1. **14:05:12** – CloudTrail records an `UpdateAutoScalingGroup` event via an assumed role session, setting `desiredCapacity: 1` and `maxSize: 1`.
  2. **14:06:03** – AWS Config confirms the resource configuration change, logging the drop from desired capacity `3` to `1`.
  3. **14:07:00** – CloudWatch records a sharp increase in target response time from `0.3 s` to `2.4 s`.
  4. **14:07:30** – CloudWatch Logs (`/rg/web/app`) report slow upstream responses (`2.38 s`) and note that only `1` healthy target is active.
  5. **14:09:00** – CloudWatch alarm `rg-web-latency-high` transitions from `OK` to `ALARM` after sustaining latency above the 1-second threshold.
* **Supported Findings & Assumptions:** The evidence strongly supports the conclusion that the unannounced capacity reduction directly starved the web tier of compute resources, resulting in latency and subsequent alarm activation. However, whether the role session action was intentional maintenance or an unauthorized configuration error remains an open assumption requiring administrative verification.

## Recommendation
1. **Immediate Capacity Restoration:** Through proper change management channels, safely restore the web tier ASG desired capacity back to `3` and maximum capacity to `4`.
2. **Post-Change Verification:** Monitor CloudWatch latency metrics and CloudWatch logs to verify that target response times return to the baseline (<0.5 s) and that healthy target counts reflect the restored capacity.
3. **Change Control Audit:** Review change advisory board (CAB) logs to determine why no approved ticket existed for the 14:05 change window.

## Escalation Notes
This incident involves an unapproved production configuration change executed via a privileged role session without an accompanying change ticket. While no malicious activity has been confirmed, the lack of documentation requires escalation to the IT Change Management and Cloud Security teams for identity auditing and session verification.

## Lessons Learned
Week 11 demonstrated that effective troubleshooting requires correlating multiple observability pillars. Metrics and alarms reveal *when* and *where* a failure occurs, but they cannot explain *why*. By correlating CloudWatch alarms with CloudTrail management events and AWS Config history, operations engineers can successfully close the gap between symptoms and root causes.

## Professional Vocabulary
* **Monitoring:** The continuous collection and tracking of operational metrics and health indicators across cloud environments.
* **Metric:** A numerical value representing a specific operational measurement (e.g., CPU utilization or response time) over time.
* **Log:** A timestamped record of discrete events, errors, or operational warnings generated by applications, operating systems, or AWS services.
* **Alarm:** An automated watcher that monitors a metric against predefined thresholds and triggers state changes or notifications.
* **CloudWatch:** The AWS monitoring and observability service aggregating metrics, logs, and alarms.
* **CloudTrail:** An AWS service that records account activity and API management events for auditing and compliance.
* **AWS Config:** A service that tracks configuration changes, evaluates resource compliance, and maintains historical snapshots of AWS resources.
* **Audit Trail:** An immutable chronological record providing documentary evidence of sequence of activities and administrative actions.
* **Configuration History:** The recorded chronological sequence of property changes applied to a cloud resource over its lifecycle.
* **Incident Timeline:** An ordered sequence of events detailing the exact progression of a system failure from initiation to detection and resolution.
* **Evidence Correlation:** The analytical process of aligning timestamps and parameters across disparate logs, metrics, and audit feeds to establish a root-cause narrative.
