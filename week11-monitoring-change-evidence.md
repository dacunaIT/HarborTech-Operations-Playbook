
# Week 11: Monitoring and Change Evidence

## HarborTech Ticket Summary

**Ticket:** TKT-2026-0011  
**Client:** Riverside Goods  
**Investigation:** What Changed? — Reconstruct the Incident  
**Role:** Junior Cloud Operations Intern

Riverside Goods reported degraded website performance following an unknown configuration change. The web tier Auto Scaling group no longer matched its approved capacity settings, and no approved change request was on file for the incident window.

My responsibility was to review the provided AWS monitoring, audit, and configuration evidence, reconstruct the incident timeline, identify the strongest supported explanation, and recommend appropriate next steps without making unauthorized production changes.

## Client Impact

The client experienced slower website response times, increasing from 0.3 seconds to 2.4 seconds.

The reduced number of healthy targets could limit the application's ability to handle incoming traffic, potentially affecting customer experience and service availability.

Unapproved configuration changes also create operational and security concerns because they make it difficult to determine why infrastructure behavior changed.

## AWS Services Involved

- **Amazon CloudWatch:** Monitors application and infrastructure performance.
- **CloudWatch Metrics:** Records measurable performance data, including response times.
- **CloudWatch Logs:** Provides application-level evidence of performance issues.
- **CloudWatch Alarms:** Notifies operations teams when monitored thresholds are exceeded.
- **AWS CloudTrail:** Records AWS API activity and identity context.
- **AWS Config:** Tracks resource configurations and changes over time.
- **Amazon EC2 Auto Scaling:** Adjusts the number of EC2 instances available to support application workloads.

Amazon Athena can also be used to query stored operational evidence, but no Athena query results were included in this investigation.

## Virtualization Connection

AWS uses virtualization to provide computing resources without requiring clients to manage physical servers directly.

Amazon EC2 instances operate as virtual servers, while Auto Scaling manages how many instances are available to support an application.

Monitoring and auditing tools are important because software-defined infrastructure can be modified through APIs, management consoles, and automated processes.

In this investigation, the reduction in Auto Scaling capacity demonstrates how a configuration change can affect the resources available to a virtualized application.

## Evidence Reviewed

The investigation used five records provided in the HarborTech evidence packet.

### 1. AWS CloudTrail

**Time:** 14:05:12  
**Event:** UpdateAutoScalingGroup  
**Resource:** Riverside Goods web-tier Auto Scaling group

CloudTrail recorded an API request from an assumed role session to set:

- Desired capacity: 1
- Maximum capacity: 1

The record confirms that the API request occurred but does not identify a specific employee or prove that the request alone caused the slowdown.

### 2. AWS Config

**Time:** 14:06:03

AWS Config recorded the following configuration changes:

| Setting | Before | After |
|---|---|---|
| Desired Capacity | 3 | 1 |
| Maximum Size | 4 | 1 |

This confirms that the Auto Scaling group configuration changed shortly after the CloudTrail event.

### 3. CloudWatch Metrics

**Time:** 14:07:00

Target response time increased from 0.3 seconds to 2.4 seconds.

This establishes that application performance degraded shortly after the configuration change.

### 4. CloudWatch Logs

**Time:** 14:07:30

The application log reported:

- Warning: Upstream response
- Response time: 2.38 seconds
- Healthy targets: 1

The log provides additional evidence of degraded application performance and reduced healthy target capacity.

### 5. CloudWatch Alarm

**Time:** 14:09:00  
**Alarm:** Web-tier latency alarm  
**State Change:** OK to ALARM

The alarm was configured to trigger when latency exceeded 1 second for two datapoints.

The ALARM state confirmed that the monitored latency condition had been breached.

## Operational Analysis

### Reconstructed Incident Timeline

| Time | Event | Operational Significance |
|---|---|---|
| 14:05:12 | CloudTrail API request | Capacity reduction requested |
| 14:06:03 | AWS Config change | Capacity reduction confirmed |
| 14:07:00 | CloudWatch metric increase | Website performance degraded |
| 14:07:30 | Application log warning | Slow upstream response and one healthy target |
| 14:09:00 | CloudWatch alarm | Latency threshold breached |

### Supported Findings

The strongest supported conclusion is that the Auto Scaling capacity reduction likely contributed to the website slowdown.

CloudTrail recorded the capacity change request, AWS Config confirmed the configuration change, and CloudWatch recorded degraded performance shortly afterward.

These records establish a clear sequence of events and a plausible relationship between the reduced capacity and increased latency.

### Assumptions and Unresolved Questions

- The evidence does not establish whether the capacity change was the only cause of the slowdown.
- The assumed role session does not identify the individual or automated process responsible.
- The investigation does not establish whether the change was intentional.
- No approved change request was on file for the incident window.
- Additional evidence would be needed to confirm the complete root cause.

No production changes were made during this investigation.

## Recommendation

I recommend that the cloud operations team review the Auto Scaling configuration change and determine whether restoring the previously approved capacity is appropriate.

Before making changes, an authorized engineer should verify the current resource configuration, application demand, and relevant change history.

CloudWatch metrics, logs, and alarms should continue to be monitored to determine whether application performance improves following any approved corrective action.

The investigation should remain evidence-based, and no individual should be assigned responsibility without additional verification.

## Escalation Notes

This incident should be escalated to the appropriate cloud operations engineer or manager because it involves a production configuration change without an approved change request on file.

Further investigation may be necessary to determine the source and authorization of the assumed role activity.

As a junior cloud operations intern, I am responsible for documenting the evidence and recommending further investigation.

I would not independently modify production resources, disable accounts, or perform a rollback without the required authorization.

## Lessons Learned

This investigation demonstrated the importance of correlating multiple AWS evidence sources instead of relying on one event.

CloudWatch helped establish when application performance degraded, CloudTrail showed the API request, and AWS Config confirmed that the resource configuration changed.

I also learned that an audit event does not automatically prove who performed an action or whether it caused an incident.

Reconstructing events in chronological order provides a stronger foundation for troubleshooting and professional incident reporting.

## Professional Vocabulary

- **Monitoring:** Observing infrastructure and application behavior to identify performance or availability problems.
- **Metric:** A numerical measurement used to track system performance.
- **Log:** A record of application or system activity.
- **Alarm:** A monitoring mechanism that changes state when defined conditions are met.
- **CloudWatch:** AWS service used to collect and monitor operational information.
- **CloudTrail:** AWS service that records API activity and associated identity information.
- **AWS Config:** Service that tracks AWS resource configurations and configuration history.
- **Audit Trail:** A chronological record of activities used to investigate system changes.
- **Configuration History:** A record of how a resource's settings have changed over time.
- **Incident Timeline:** An ordered sequence of events used to reconstruct an operational incident.
- **Evidence Correlation:** Comparing multiple records to determine how events may be related.
