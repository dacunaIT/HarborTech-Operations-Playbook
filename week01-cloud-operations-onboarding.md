# Week 1: Cloud Operations Onboarding

## HarborTech Ticket Summary
ONB-2026-0001 required me (as an intern) to verify I could access the training tools required for tech operation within HarborTech 
and that I understood the boundaries and restrictions of the learning environment. I was required to document what operations were verified
successful, what operations were restricted, and if an issue requires escalation. A client complains that they cannot create an IAM user
, this did not require escalation, given this is an expected behavior of the training tool.

## Client Impact
Environment readiness matters because it would be irresponsible of HarborTech to issue tickets to an intern who cannot access and verify
a successful work environment. It is also irresponsible and not acceptable to issue tickets to an intern who does not fully understand
the expected behaviors and restrictions of the work environment. This is not acceptable because the intern may not identify whether an issue
should be escalated or not. They may also not be able to identify the issue properly or accurately.

## AWS Services Involved
AWS Academy - The training platform which was used to access AWS learner lab
Learner Lab - The controlled AWS sandbox used for training
IAM -The AWS service used to create and control identities, roles, and permissions
Region - The AWS location in which services and resources operate, was confirmed to be us-east-1
AWS Account- Account is given boundaries within the AWS Learner Lab
## Virtualization Connection
Virtualization allows cloud resources to be created and managed through software rather than requiring access to the physical hardware 
of a server. Cloud resources are still controlled by the AWS account, Region, IAM permissions, cost limits, service availability,
and Learner Lab restrictions. This means that even though the physical hardware is being hosted elsewhere, the cloud operations team must still understand 
where resources are running, and what actions are restricted

## Evidence Reviewed
Document the evidence you reviewed, such as:
During the onboarding activity, I verified the following evidence
AWS Academy Access was successful
AWS Learner Lab was successful
The permitted region was set to us-east-1
IAM permissions are restricted within the learner lab
Budget Display will not always update immediately 
Resource status should be verified directly

## Operational Analysis
The evidence shows that I can access the required HarborTech training environment and understand the
major operating procedures,restrictions, and behavior of the Learner Lab.
A verified finding is that AWS Learner Lab access is functioning correctly. Another verified finding is that the environment limits certain IAM actions, 
such as creating IAM users. This restriction is expected Learner Lab behavior and does not indicate an AWS platform failure.
An assumption would be concluding that the AWS learner lab is broken or working incorrectly because we cannot create an IAM user.

## Recommendation
My environment is ready to operate in week 2, a next step is too continue using the professional process, confirm the environment is successfully
operational, document findings, recommend an action, and escalate the issue if need be.

## Escalation Notes
No escalation was required for this lab.

## Lessons Learned
A cloud operations professional must understand the environment they are working in, verifying evidence, understanding restrictions
and expected behaviors, and document findings.

## Professional Vocabulary
- Virtualization - Accessing hardware such as servers via software to manage and configure resources as if they were on prem.
- Evidence - Information that can be observed, collected, and verified to support a conclusion or finding.
- Finding - A conclusion that is supported by verified evidence.
- Assumption - A claim that has not yet been verified by evidence.
- Escalation - Sending an unresolved issue to a higher authority, after the issue is verified valid.
- Sandbox - An environment designed for training, usually with permissions restricted.
- Region - The geographical area where your AWS resources are operating.
- IAM - Identity and Access management, a resource within AWS used to create users, roles, and identities.
- Operations Playbook - An accessible record of operational evidence, analysis, findings,
- recommendations, escalation notes, procedures, and lessons learned.
