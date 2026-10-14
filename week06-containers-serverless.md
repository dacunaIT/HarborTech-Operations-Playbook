# Week 6: Containers and Serverless Computing

## HarborTech Ticket Summary

**Ticket ID:** TKT-2026-0006  
**Client:** Bright Path Community Services  
**Issue:** Evaluation of the current EC2-based compute model  
**Status:** Analysis and recommendation completed; implementation pending review

Bright Path Community Services currently operates two workloads on a shared Amazon EC2 instance: a daily reporting task and a small REST API. Although both applications are functioning, the EC2 instance remains lightly utilized for extended periods.

The client continues to maintain the guest operating system and application environment even when little computing activity is occurring.

HarborTech was tasked with evaluating whether virtual machines, containers, or serverless computing would better support these workloads. The investigation focuses on workload behavior, operational responsibilities, resource efficiency, and application compatibility.

## Client Impact

Continuing to operate an EC2 instance for workloads that execute only occasionally results in inefficient resource utilization and unnecessary operational expenses to add up.

The client must maintain operating system updates, security patches, and server monitoring regardless of how frequently the applications execute.

A better-matched compute model could provide several benefits:

- **Cost Efficiency:** Reduce unnecessary computing expenses associated with maintaining idle server capacity.
- **Operational Efficiency:** Reduce time spent maintaining guest operating systems and server infrastructure.
- **Scalability:** Allow application execution to respond to changing workload demand.
- **Reliability:** Use managed AWS services to reduce certain infrastructure maintenance responsibilities.
- **Security:** Reduce direct operating system management while maintaining appropriate application and IAM security controls.

However, migrating without evaluating application dependencies could introduce compatibility problems, additional costs, or service interruptions.

## AWS Services Involved

The following AWS services and concepts were considered during this investigation.

**Amazon EC2:** Provides virtual machines that allow customers to configure and manage guest operating systems, application software, and instance-level resources. EC2 offers significant flexibility but requires additional maintenance.

**Containers:** Package applications and their dependencies into isolated environments that share a host operating system kernel. Containers can improve application portability and deployment consistency.

**Amazon ECS and AWS Fargate:** Amazon ECS orchestrates containerized applications, while AWS Fargate provides managed compute capacity for supported container workloads without requiring customers to manage EC2 container hosts.

**AWS Lambda:** Executes application code in response to events without requiring customers to manage servers directly. Lambda is appropriate for supported short-duration, event-driven workloads.

**Amazon API Gateway:** Receives and manages API requests and can integrate with Lambda to process incoming HTTP requests.

**Amazon EventBridge Scheduler:** Can trigger a Lambda function according to a defined schedule, making it appropriate for a daily reporting task.

**AWS Step Functions:** Coordinates multiple application steps or AWS services into workflows. It could be considered if the reporting task develops additional processing or error-handling requirements.

**IAM Execution Roles:** Provide Lambda functions with permissions to access authorized AWS resources while supporting least-privilege security practices.

**Amazon CloudWatch:** Provides metrics and logs that can be used to monitor workload performance, errors, and execution behavior.

**VPC-Connected Lambda:** Allows Lambda functions to access supported resources within a VPC when private network connectivity is required.

## Virtualization Connection

Traditional virtualization allows multiple virtual machines to operate on physical hardware through a hypervisor.

An Amazon EC2 instance is a virtual machine that provides a guest operating system and configurable computing resources. Although AWS manages the underlying physical infrastructure, the customer remains responsible for administering the guest operating system.

Containers provide a different abstraction layer. Rather than requiring a complete guest operating system for each application, containers share the host operating system kernel while isolating application processes and dependencies.

This makes containers useful when applications need consistent runtime environments without requiring separate virtual machines.

Serverless computing provides another level of abstraction. With AWS Lambda, AWS manages the underlying compute infrastructure and supported runtime environment while the customer focuses primarily on application code, configuration, permissions, and monitoring.

The major difference between these models is the level of infrastructure control and operational responsibility retained by the customer.

## Evidence Reviewed

**Evidence Source:** HarborTech Ticket TKT-2026-0006 and the provided Week 6 workload scenarios.

The following evidence was supplied through the assignment. No production AWS measurements or configuration changes were independently verified.

### Daily Reporting Task

- The task executes once each morning.
- Execution completes quickly.
- No local state is maintained between executions.
- The EC2 instance remains idle outside the reporting period.

**Interpretation:**

The reporting task demonstrates characteristics associated with scheduled, event-driven computing.

Because execution occurs only briefly each day, maintaining a continuously running EC2 instance solely for this task may be unnecessary.

AWS Lambda triggered by EventBridge Scheduler is a strong recommendation for adaption, provided the application meets Lambda execution and dependency requirements.

### Small REST API

- The API receives intermittent requests throughout the day.
- Request handling does not necessarily require continuously active compute capacity.
- The API currently shares the EC2 instance with the reporting task.

**Interpretation:**

An API Gateway and Lambda integration could allow the application to respond to incoming requests without requiring a continuously running application server.

However, request volume, response time, authentication, dependencies, and backend integration requirements must be evaluated before adaption.

### Continuous Application Scenario

The assignment also presented a continuously running application that requires a custom operating system package and host-level troubleshooting access.

**Interpretation:**

These requirements weaken the suitability of AWS Lambda because Lambda does not provide traditional administrative access to the underlying host operating system.

An EC2 virtual machine may be more appropriate. Containers remain an alternative if the dependencies can be packaged and host-level access is not essential.

### Evidence Still Required

Before making a production architecture decision, HarborTech would need to collect:

- Actual workload execution durations and frequency.
- EC2 CPU, memory, and network utilization.
- API request frequency.
- Required IAM permissions.
- VPC connectivity.
- Application logs and monitoring metrics.
- Current EC2 costs.


## Operational Analysis

The current EC2 environment provides both workloads with a complete virtual machine, even though their execution patterns do not require continuous computing activity.

For the daily reporting task, a scheduled Lambda function could provide computing resources only when the task executes. EventBridge Scheduler could invoke the function each morning.

For the REST API, API Gateway could receive incoming HTTP requests and route them to Lambda. Lambda would execute the relevant application logic and return a response.

This approach could reduce the need for guest operating system maintenance and improve resource efficiency.

However, serverless computing introduces different operational considerations, including execution time limits, cold-start latency, concurrency behavior, service quotas, logging, and integration requirements.

A container-based implementation could provide more control over application dependencies and support continuously running processes. AWS Fargate could also reduce the need to manage container-host infrastructure.

A traditional EC2 instance remains appropriate when applications require direct guest operating system management, specialized configurations, or host-level troubleshooting.

Selecting a compute model should depend on actual workload behavior and operational requirements rather than choosing a service simply because it is newer.

## Recommendation

Based on the available evidence, I recommend evaluating different compute models for the client's workloads rather than automatically retaining the existing shared EC2 environment.

**Daily Reporting Task — AWS Lambda with EventBridge Scheduler**

The scheduled reporting task is a strong candidate for serverless execution because it runs briefly, follows a predictable schedule, and does not maintain local state between runs.

Before migration, HarborTech should validate execution duration, runtime compatibility, external dependencies, IAM permissions, and output storage requirements.

**REST API — Amazon API Gateway with AWS Lambda**

The intermittent request pattern makes API Gateway and Lambda a reasonable serverless alternative.

This architecture could reduce idle compute usage while allowing AWS to manage the underlying execution infrastructure.

Before implementation, HarborTech should test request handling, authentication, response latency, error handling, and connectivity to any required backend services.

**Continuously Running Applications — EC2 or Containers**

Applications requiring continuous execution, custom operating system configurations, and host-level troubleshooting should remain candidates for EC2.

Containers should be considered if the application can operate within a containerized environment without requiring direct access to the host operating system.

**Operational Responsibility Tradeoff**

Under Lambda, AWS assumes responsibility for managing the underlying compute infrastructure.

HarborTech and the client would continue to manage application code, security configurations, IAM permissions, data access, deployment processes, and monitoring.

Any migration should occur only after technical validation, cost evaluation, and approval.

## Escalation Notes

No production implementation or infrastructure modification was performed during this investigation.

The following actions would require review or approval from the appropriate HarborTech personnel:

- Migrating the existing EC2 workloads into AWS Lambda.
- Creating production API Gateway endpoints.
- Modifying IAM roles and permissions.
- Configuring production networking or VPC connectivity.
- Updating application dependencies or deployment procedures.
- Changing monitoring, logging, or security configurations.
- Decommissioning the existing EC2 instance.

Additional coordination with the application owner would be necessary to confirm functional requirements and acceptable downtime.

Any migration plan should include testing, a rollback procedure, and verification that the replacement architecture meets application requirements.

The recommendation remains an architectural proposal until appropriate authorization is received.

## Lessons Learned

This investigation demonstrated that a functioning server does not necessarily represent the most efficient computing solution.

Virtual machines, containers, and serverless functions provide different levels of flexibility, abstraction, and management responsibility.

AWS Lambda is particularly useful for compatible workloads that respond to events or execute for short periods. However, it is not appropriate for every application, especially those requiring continuous execution or direct host-level control.

Containers provide application isolation and portability, but they do not eliminate all infrastructure or application management responsibilities.

I also learned that serverless computing transfers certain infrastructure responsibilities to AWS while leaving customers responsible for application security, permissions, configuration, and monitoring.

The most important lesson from Week 6 is that compute decisions should be based on workload evidence, technical compatibility, cost, and operational needs rather than service popularity.

## Professional Vocabulary

**Container:** An isolated environment that packages application code and dependencies while sharing a host operating system kernel.

**Serverless Computing:** A cloud computing model where the provider manages the underlying infrastructure required to execute workloads.

**AWS Lambda:** A serverless AWS compute service that executes code in response to events.

**API Gateway:** An AWS service that creates, publishes, secures, and manages APIs.

**Execution Role:** An IAM role that grants an AWS Lambda function permission to access authorized AWS services and resources.

**Event Trigger:** An event or schedule that initiates application execution.

**Step Functions:** An AWS service used to coordinate application tasks and workflows.

**VPC-Connected Lambda:** A Lambda configuration that provides network access to resources within a VPC.

**Runtime:** The software environment used to execute application code.

**Abstraction:** The process of hiding lower-level infrastructure complexity behind a simplified computing interface.
