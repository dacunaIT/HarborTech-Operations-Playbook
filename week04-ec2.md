# Week 4: EC2 Web Server Troubleshooting and Lifecycle Validation

## HarborTech Ticket Summary

Ticket `TKT-2026-0004` involved an EC2 web server that was running but could not initially be reached over HTTP from its public IPv4 address. The troubleshooting goal was to collect evidence, identify the failing layer, apply the smallest supported correction, verify the result, test stop/start behavior, and clean up the temporary lab resources.

## Client Impact

The EC2 instance was available as an AWS resource, but the client could not reach the Apache test page over HTTP. This prevented external access to the web service even though the instance itself was running.

## Environment and Resource Names

- Region: `us-east-1`
- VPC ID: `vpc-04734b8ce352b7a2d`
- Subnet ID: `subnet-0ceaf221b3147c640`
- Security group name: `week4-practice-diego`
- Security group ID: `sg-0a117d6171696436a`
- Instance ID: `i-00fab04391e0a6d6c`
- Initial public IPv4 address: `100.26.5.58`
- Public IPv4 address after stop/start: `44.204.110.55`
- AMI ID: `ami-0fef201115eefe936`
- Test page content: `IT 4560 Week 4 EC2 Practice`

Sensitive identity details such as account ID, user ID, ARN, access keys, secret keys, and session tokens are recorded as `[REDACTED]` when applicable.

## AWS Documentation Evidence

### Source 1 - Security Groups

**Document title:** *Amazon Elastic Compute Cloud User Guide*  
**PDF page:** 3296  
**Exact quote:** “Inbound rules control the incoming traffic to your instance, and outbound rules control the outgoing traffic from your instance.”  
**URL:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf

This source explains that security group inbound rules determine what traffic can reach an EC2 instance. In this lab, Apache was running, but HTTP access failed because TCP port 80 was not allowed inbound. Adding the port 80 rule corrected the reachability issue without rebuilding the instance.

### Source 2 - User Data

**Document title:** *Amazon Elastic Compute Cloud User Guide*  
**PDF page:** 1735  
**Exact quote:** “When you launch an Amazon EC2 instance, you can pass user data to the instance that is used to perform automated configuration tasks, or to run scripts after the instance starts.”  
**URL:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf

In this lab, user data was used to automate the installation and startup of Apache and create the test web page when the instance launched. However, having user data configured does not by itself prove that Apache is running or that the website is reachable, so those conditions were verified separately.

### Source 3 - Stop/Start Lifecycle

**Document title:** *Amazon Elastic Compute Cloud User Guide*  
**PDF pages:** 1551-1552  
**Exact quote:** “The instance gets a new public IPv4 address, unless it has a secondary network interface or a secondary private IPv4 address that is associated with an Elastic IP address.”  
**URL:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf

During the stop/start test, the instance kept its identity and EBS-backed data, but the automatically assigned public IPv4 address changed. This required checking the current public IP again before retesting the Apache page.

## CloudShell Command Record

```bash
$ aws sts get-caller-identity
{
    "UserId": "[REDACTED]",
    "Account": "[REDACTED]",
    "Arn": "[REDACTED]"
}
```

```bash
$ aws configure get region
[no output]
```

```bash
$ aws configure set region us-east-1
```

```bash
$ aws configure get region
us-east-1
```

```bash
$ aws ec2 describe-subnets
...
"SubnetId": "subnet-0ceaf221b3147c640",
"VpcId": "vpc-04734b8ce352b7a2d",
"AvailabilityZone": "us-east-1e"
```

```bash
$ aws ec2 create-security-group --group-name week4-practice-diego --description "IT 4560 Week 4 EC2 Practice" --vpc-id vpc-04734b8ce352b7a2d
{
    "GroupId": "sg-0a117d6171696436a"
}
```

```bash
$ cat > user-data.sh <<'EOF'
#!/bin/bash
yum install -y httpd
systemctl enable --now httpd
echo "IT 4560 Week 4 EC2 Practice" > /var/www/html/index.html
EOF
```

```bash
$ cat user-data.sh
#!/bin/bash
yum install -y httpd
systemctl enable --now httpd
echo "IT 4560 Week 4 EC2 Practice" > /var/www/html/index.html
```

```bash
$ aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query "Parameter.Value" --output text
ami-0fef201115eefe936

$ aws ec2 run-instances --image-id ami-0fef201115eefe936 --instance-type t2.micro --subnet-id subnet-0ceaf221b3147c640 --security-group-ids sg-0a117d6171696436a --user-data file://user-data.sh --iam-instance-profile Name=LabInstanceProfile --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=week4-practice-diego}]'

An error occurred (InsufficientInstanceCapacity): We currently do not have sufficient t2.micro capacity in the requested Availability Zone.
```

[Instance was then launched using UI and was successful.]

Instance ID: i-00fab04391e0a6d6c
Initial Public IPv4: 100.26.5.58

```bash
$ aws ec2 describe-security-groups --group-ids sg-0a117d6171696436a --query "SecurityGroups[0].IpPermissions"

[
    {
        "IpProtocol": "tcp",
        "FromPort": 443,
        "ToPort": 443,
        "IpRanges": [
            {
                "CidrIp": "0.0.0.0/0"
            }
        ]
    }
]
```

Port 80 was not present in the inbound rules.

```bash
$ curl http://100.26.5.58
curl: (7) Failed to connect to 100.26.5.58 port 80: Could not connect to server
```

## Baseline Evidence
Evidence A - Instance ID and AMI ID
The instance ID proves which EC2 instance was being tested, and the AMI ID identifies the image used to create it. This does not prove that the instance is healthy, that Apache is running, or that the web page is reachable.

Evidence B - Initial Public IPv4 Address
The public IPv4 address proves which public address was assigned to the instance at that time. It does not prove that port 80 is open, that the security group allows HTTP, or that the web server is responding.

Evidence C - Status Checks
Passing EC2 status checks proves that AWS considers the instance and underlying infrastructure operational. It does not prove that Apache is running correctly or that the application is reachable over HTTP.

Evidence D - Security Group Before Fix
The security group evidence proves that inbound TCP port 80 was not allowed before remediation. This supports the conclusion that HTTP traffic could be blocked at the network-access layer, but by itself it does not prove that Apache was functioning inside the instance.

Evidence E - Failed HTTP Test
The failed curl http://PUBLIC_IP test proves that the web page was not reachable over HTTP from the test location at that time. It does not by itself identify the root cause because the failure could have been caused by the security group, Apache, the instance network path, or another configuration issue

## Root-Cause Analysis
The evidence showed that the EC2 instance was running and that the external HTTP request failed while the security group did not allow inbound TCP port 80. The strongest supported root cause was the missing HTTP ingress rule. The failed HTTP request alone was not enough to prove the cause, so guest-side verification was also performed to rule out an Apache service failure.

## Corrective Action
The smallest supported correction was to add one inbound TCP port 80 rule to the existing security group instead of rebuilding the instance or changing unrelated resources.
```bash
aws ec2 authorize-security-group-ingress --group-id sg-0a117d6171696436a --protocol tcp --port 80 --cidr 0.0.0.0/0
```
After the change, the security group was checked again to confirm that TCP port 80 was present.

## Verification Evidence
```bash
$ curl http://100.26.5.58
IT 4560 Week 4 EC2 Practice

The successful HTTP response confirmed that the web server became externally reachable after the security-group correction.

## IMDSv2 and Guest Evidence
The instance was accessed with Session Manager so the web service could be verified from inside the guest operating system.
```bash
$ sudo systemctl status httpd
httpd.service - The Apache HTTP Server
Active: active (running)

$ curl http://localhost
IT 4560 Week 4 EC2 Practice
```
This confirmed that Apache was actively running and serving the expected page locally.

An IMDSv2 token was then requested and used to retrieve the instance ID:
```bash
$ TOKEN=$(curl -X PUT -s "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
i-00fab04391e0a6d6c
```
This confirmed from inside the guest that the running system was the same EC2 instance being investigated. CloudShell control-plane commands can describe AWS resources, but they do not by themselves prove that an application inside the guest is running and serving content.

## Stop/Start Lifecycle Test
```bash
$ aws ec2 stop-instances --instance-ids i-00fab04391e0a6d6c
[Instance entered stopping state]

$ aws ec2 wait instance-stopped --instance-ids i-00fab04391e0a6d6c

$ aws ec2 start-instances --instance-ids i-00fab04391e0a6d6c
[Instance entered pending/running state]

$ aws ec2 wait instance-running --instance-ids i-00fab04391e0a6d6c

$ aws ec2 describe-instances --instance-ids i-00fab04391e0a6d6c --query "Reservations[0].Instances[0].[InstanceId,PublicIpAddress,PrivateIpAddress]" --output table
```
Before the stop/start cycle, the public IPv4 address was 100.26.5.58. After the instance restarted, the public IPv4 address changed to 44.204.110.55, while the instance ID remained i-00fab04391e0a6d6c.
```bash

$ curl http://44.204.110.55
IT 4560 Week 4 EC2 Practice
```
The Apache installation and page content persisted through the lifecycle event. This demonstrated that the instance and EBS-backed data persisted while the automatically assigned public IPv4 address was replaced.


## Cleanup Evidence
```bash
$ aws ec2 terminate-instances --instance-ids i-00fab04391e0a6d6c
[Instance entered shutting-down state]

$ aws ec2 wait instance-terminated --instance-ids i-00fab04391e0a6d6c

$ aws ec2 delete-security-group --group-id sg-0a117d6171696436a
```
After testing was complete, the temporary EC2 instance was terminated and the temporary security group was deleted. This removed the disposable lab resources and avoided leaving unnecessary resources or access rules active.

## Escalation and Change-Control Notes
In a production environment, I would not add or broaden an inbound security-group rule, such as allowing TCP port 80 or 22 from 0.0.0.0/0, without authorization. A broad ingress rule can expose a workload to traffic from any IPv4 address, increase the attack surface, and potentially violate organizational security requirements.

Before making a production change, I would obtain approval from the system owner, security team, or authorized change-management personnel. I would also record the original security-group configuration so the new rule could be removed if necessary, and I would keep before-and-after connectivity evidence to verify either the change or the rollback.

Rebuilding the EC2 instance was not justified because the instance was healthy and Apache worked correctly inside the guest. The failure was isolated to the network-access layer, so adding the required TCP port 80 rule was the smallest supported corrective action.

## Lessons Learned
This lab reinforced the importance of troubleshooting by layers instead of immediately rebuilding a resource. A healthy EC2 instance does not automatically mean an application is reachable, and a failed external request does not automatically prove that the application itself is broken. Control-plane evidence, security-group configuration, guest-side service status, local application testing, and external connectivity testing each answer different questions.

The stop/start test also demonstrated that an EC2 instance can preserve its identity and EBS-backed workload while an automatically assigned public IPv4 address changes.

## Professional Vocabulary
Security group: A stateful virtual firewall that controls permitted inbound and outbound traffic for AWS resources such as EC2 instances.

Ingress rule: A rule that defines which incoming traffic is allowed to reach a resource.

User data: Data or scripts supplied to an EC2 instance at launch for automated configuration or startup tasks.

IMDSv2: Instance Metadata Service Version 2, which uses a session token to securely retrieve metadata from inside an EC2 instance.

Instance ID: The unique identifier assigned to an EC2 instance.

AMI: Amazon Machine Image, the template used to launch an EC2 instance.

Public IPv4 address: An internet-routable IPv4 address assigned to an EC2 instance.

EBS-backed storage: Persistent block storage that can retain data across an EC2 stop/start cycle.

Status checks: AWS health checks that evaluate the instance and underlying infrastructure.

Root cause: The underlying condition supported by evidence that explains the observed problem.

Corrective action: The change made to address the identified cause of a problem.

Verification: Evidence collected after a change to confirm that the expected result occurred.

Change control: A formal process for reviewing, approving, documenting, and managing changes to production systems.

Rollback: Reversing a change to restore a previously known working configuration
