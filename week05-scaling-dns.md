# Week 5: Scaling, Load Balancing, and DNS

## HarborTech Ticket Summary

Ticket TKT-2026-0005 focuses on an availability and capacity risk for Riverside Goods. The current design depends too heavily on a single server, while a predictable seasonal promotion is expected to significantly increase traffic. Previous promotion traffic already pushed the existing system to 92% CPU utilization, so the upcoming increase creates both a capacity problem and a single-point-of-failure risk.

The proposed design includes Auto Scaling, an Application Load Balancer, healthy targets, and a secondary recovery endpoint. The goal of the investigation was to determine which parts of that design were supported by ticket evidence and which parts could actually be verified in my assigned AWS Learner Lab.

## Client Impact

If traffic increases beyond what the current server can handle, users could experience slow response times, failed requests, or complete service interruption. Because the current environment also depends on one main endpoint, the failure of that endpoint could make the application unavailable even if additional compute resources existed elsewhere.

For Riverside Goods, this could affect customers during an important promotion, reduce sales, create a poor customer experience, and increase operational pressure on support staff. Improving both capacity and availability is therefore important before the expected increase in traffic.

## Provided Ticket Evidence

The following information was provided by HarborTech as ticket evidence and was not personally collected from my AWS environment:

- Previous promotion traffic reached 92% CPU utilization.
- The upcoming promotion is expected to double request volume.
- The proposed Auto Scaling configuration uses a minimum capacity of 2, desired capacity of 2, and maximum capacity of 6.
- The proposed Application Load Balancer reports both test targets as healthy.
- A secondary recovery endpoint is available.
- Route 53 failover has not yet been confirmed.

This evidence supports the need for additional capacity, traffic distribution, health monitoring, and DNS failover verification.

## AWS Commands Used

I executed the following AWS CLI commands in AWS CloudShell.

## Auto Scaling Investigation

```bash
aws autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize}' \
  --output table
```

## Load Balancer / Target Groups
```bash
aws elbv2 describe-target-groups \
  --query 'TargetGroups[].{Name:TargetGroupName,TargetGroupArn:TargetGroupArn,Protocol:Protocol,Port:Port}' \
  --output table
```
Because no target group was returned, I stopped the target health investigation as instructed and did not run the target health command.

## Route 53 Investigation
```bash
aws route53 list-hosted-zones \
  --query 'HostedZones[].{Name:Name,Id:Id,Private:Config.PrivateZone}' \
  --output table
```
## AWS Evidence Collected
My assigned AWS Learner Lab returned no Auto Scaling groups. No Auto Scaling group name, minimum capacity, desired capacity, or maximum capacity values were displayed.
The target group command also returned no target groups. Because no target group was available, there was no target group ARN to use for a target health investigation.
The Route 53 command returned no hosted zones. No hosted zone name, hosted zone ID, or private zone value was displayed.
These results mean that I could not personally verify the proposed Auto Scaling configuration, load balancer target configuration, target health, or Route 53 failover within my assigned AWS environment.

## Virtualization Connection
Virtualization allows compute resources such as EC2 instances to be created, replaced, and scaled without depending on one physical server. An Auto Scaling group can increase or decrease the number of virtual EC2 instances based on workload requirements.
A load balancer can then distribute incoming traffic across multiple virtual instances instead of sending all requests to one server. This improves scalability and availability because workloads can be spread across multiple systems, and unhealthy or failed instances can be replaced without requiring the application to depend on one physical machine.


## Operational Analysis
The provided ticket evidence shows that the current environment has both a capacity risk and an availability risk. A previous promotion already reached 92% CPU utilization, and the upcoming promotion is expected to double request volume. The proposed Auto Scaling configuration of minimum 2, desired 2, and maximum 6 could help by maintaining at least two instances and allowing capacity to expand during higher demand.
However, my AWS evidence did not return an Auto Scaling group, so I could not verify that this configuration is currently deployed.
Additional compute capacity alone does not distribute client traffic. An Application Load Balancer is needed to send incoming requests across registered targets. The ticket states that two test targets are healthy, but my AWS environment returned no target groups, so I could not independently verify target registration or health behavior.
Healthy targets also do not prove that the application can survive a DNS or endpoint failure. The ticket states that a secondary recovery endpoint exists, but my AWS environment returned no Route 53 hosted zones. Because of this, I could not verify DNS records, Route 53 health checks, routing policies, or failover behavior.
The major remaining unknowns are whether the proposed Auto Scaling group is actually deployed, whether a load balancer is actively distributing traffic, whether health checks remove unhealthy targets correctly, and whether Route 53 failover is fully configured and tested.


##cRecommendation
HarborTech should verify and implement the proposed Auto Scaling, load balancing, health-check, and DNS failover controls before the seasonal promotion.
For capacity, the proposed minimum 2, desired 2, and maximum 6 Auto Scaling configuration should be validated and tested to confirm that the system can increase capacity during higher traffic while avoiding unnecessary cost during normal demand.
For traffic distribution, an Application Load Balancer should be verified to ensure that requests are distributed across multiple healthy targets rather than relying on one endpoint.
For health behavior, HarborTech should confirm that health checks correctly identify unhealthy targets and prevent the load balancer from sending traffic to failed instances.
For DNS availability, Route 53 failover should not be considered ready until the primary and secondary records, health checks, routing policy, and actual failover behavior have been verified.
After implementation, HarborTech should monitor CPU utilization, instance count, Auto Scaling activity, target health, load balancer request distribution, application errors, Route 53 health checks, and failover results. This approach improves availability while still controlling cost by allowing capacity to increase only when required.




## Lessons Learned
Week 5 showed me the importance of separating proposed architecture from verified evidence. A design can sound highly available, but that does not prove that the configuration is actually deployed or working.
I also learned that Auto Scaling and load balancing solve different problems. Auto Scaling changes the amount of compute capacity available, while a load balancer distributes traffic across that capacity. Health checks help determine which targets should receive traffic, but they do not automatically provide DNS failover.
Route 53 adds another layer of availability by controlling where users are directed when an endpoint becomes unhealthy. Reliable cloud operations require evidence from each layer rather than assuming that one service solves the entire availability problem.

## Professional Vocabulary
Elasticity: The ability of a cloud environment to automatically increase or decrease resources as demand changes.
Scalability: The ability of a system to handle increased workload by adding or improving resources.
Load Balancer: A service that distributes incoming network or application traffic across multiple available targets.
Target Group: A collection of resources, such as EC2 instances, that a load balancer can send traffic to.
Health Check: A test used to determine whether a target or endpoint is functioning correctly and should continue receiving traffic.
Auto Scaling Group: A group of EC2 instances managed together so AWS can maintain, increase, or decrease the number of running instances.
Launch Template: A reusable configuration that defines settings used when new EC2 instances are launched, such as the AMI, instance type, networking, and security settings.
Desired Capacity: The number of instances that an Auto Scaling group attempts to maintain under the current configuration.
Minimum Capacity: The lowest number of instances an Auto Scaling group is allowed to maintain.
Maximum Capacity: The highest number of instances an Auto Scaling group is allowed to create.
Route 53: AWS's DNS service used to direct users to application endpoints and support routing methods such as failover.
Failover: The process of redirecting traffic from a failed or unhealthy primary resource to a working secondary resource.
Single Point of Failure: A component whose failure can make the entire application or service unavailable.






