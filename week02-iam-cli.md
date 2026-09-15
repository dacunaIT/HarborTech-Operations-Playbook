# Week 2: IAM and AWS CLI Investigation

## HarborTech Ticket Summary
Ticket TKT-2026-0002 involved an authorization problem for Riverside Goods. Marcus Webb can successfully sign in to the AWS account, but
he cannot perform the inventory work assigned to him. His job requires him to list the `riverside-inventory` S3 bucket,
read inventory report objects, and upload approved inventory files. The evidence shows that Marcus has an IAM user but no 
listed job-function group membership and no directly attached permission policy. A request to list the bucket returns `AccessDenied`.
The client proposed attaching `AmazonS3FullAccess` directly to Marcus so the issue could be resolved quickly.

My task was to determine whether that access level was appropriate, identify the likely authorization gap, recommend a least-privilege
direction, investigate IAM behavior in the Learner Lab, and document what should be escalated to authorized HarborTech staff.

## Client Impact
Marcus is able to authenticate to AWS, but his missing authorization prevents him from performing the inventory duties
required by Riverside Goods. Without the correct S3 permissions, he cannot list the inventory bucket, read inventory reports,
or upload approved inventory files.

Granting too little access prevents Marcus from doing his job, while granting too much access creates unnecessary security risk.
The access solution therefore needs to support the required inventory tasks without granting administrative or unrelated permissions.

## AWS Services Involved

This investigation involved AWS Identity and Access Management (IAM), Amazon S3, AWS CloudShell, the AWS Command Line Interface, 
AWS Security Token Service caller identity, and AWS Regions.

IAM was used to examine concepts such as users, roles, trust policies, managed policies, inline policies, authentication,
and authorization. Amazon S3 is the service involved in Marcus's required inventory work. CloudShell and the AWS CLI
were used to inspect the Learner Lab identity and LabRole. The active Learner Lab Region was confirmed as `us-east-1`.

## Virtualization Connection
Cloud resources such as compute, storage, and networking are software-defined, but access to those virtual resources is still 
controlled by identity and authorization rules. IAM determines which users, roles, and services can manage cloud resources
and what actions they are allowed to perform.

Virtualization abstracts the physical infrastructure, but it does not remove the need for access controls.
A user can successfully sign in to a cloud account and still be unable to manage a virtual resource if the required permissions 
have not been granted.


## Evidence Reviewed

The Riverside Goods evidence packet showed that Marcus successfully authenticated to AWS using his assigned IAM user credentials. 
This proves that his identity was recognized, but it does not prove that he is authorized to perform his required S3 actions.
The business requirement states that Marcus must be able to list the `riverside-inventory` bucket, read inventory report objects,
and upload approved inventory files. The permission record showed no listed job-function group membership and no directly
attached permission policy. A request to list the bucket returned `AccessDenied`.
In the Learner Lab, I confirmed the active Region as us-east-1 by checking the Region selector in the upper-right corner
of the AWS Management Console.

I ran:
```bash
aws sts get-caller-identity
```
Relevant output: 
```text
Account: [REDACTED]
Arn: arn:aws:sts::[REDACTED]:assumed-role/voclabs/[REDACTED]
```

This confirmed that the CLI was authenticated to the Learner Lab account through an assumed voclabs role.
It established the caller identity, but not the full set of permissions available to that identity.

I then ran:
```bash
aws iam get-role --role-name LabRole
```
Relevant output: 
```text
RoleName: LabRole
Action: sts:AssumeRole
MaxSessionDuration: 3600
RoleLastUsed Region: us-east-1
```

The AssumeRolePolicyDocument also listed multiple AWS service principals. This shows the trust relationship for LabRole and
identifies who or what may assume it. It does not define the role's actual permission set.

Next, I ran:
```bash
aws iam list-attached-role-policies --role-name LabRole
```
Relevant output: 
```text
AmazonSSMManagedInstanceCore
AmazonEKSClusterPolicy
AmazonEC2ContainerRegistryReadOnly
AmazonEKSWorkerNodePolicy
3 Learner Lab VocLabPolicy policies
```

The command returned seven attached managed policies. 

I then ran:
```bash
aws iam list-role-policies --role-name LabRole
```
Relevant output: 
```json
{
  "PolicyNames": []
}
```
This shows that LabRole has attached managed policies but no inline policies.

I compared the CLI evidence with the IAM console. The console also showed seven
attached permission policies and the same trust relationship. I also observed within the console:
AccessDenied
Action: access-analyzer:ListPolicyGenerations
Context: no identity-based policy allows the action

This confirmed an additional Learner Lab permission boundary.

## Operational Analysis
The evidence shows that Marcus's problem is an authorization gap, not an authentication failure.
His successful login proves that AWS recognizes his IAM credentials. The `AccessDenied` result when listing the inventory
bucket proves that he does not currently have the required permission for that S3 action.

The lack of listed group membership or a directly attached permission policy supports the conclusion that the required job 
permissions have not been assigned. However, the evidence does not prove every possible policy path in the production account,
so the final implementation should still be reviewed by an authorized HarborTech team member.

The Learner Lab investigation also demonstrated the difference between trust and permissions.
LabRole's trust policy defines which entities may assume the role. Its attached managed policies define permission
sources for actions the role may perform. The two should not be treated as the same thing.

## Recommendation

I do not recommend attaching `AmazonS3FullAccess` directly to Marcus because that would grant broader access than his
Inventory Coordinator responsibilities require.
A least-privilege solution should grant only the S3 actions needed for his job. Marcus needs `s3:ListBucket`
for the `riverside-inventory` bucket and appropriate `s3:GetObject` and `s3:PutObject` permissions for the approved inventory objects
or prefixes inside that bucket.

The access should be scoped only to the required Riverside Goods resources instead of granting access to unrelated S3 buckets
or unrelated AWS services.

## Escalation Notes
The supported finding is that Marcus can authenticate successfully but does not have the authorization required for his inventory work. 
The recommended access direction is a narrowly scoped S3 policy based on least privilege rather than `AmazonS3FullAccess`.

The actual production IAM change should be reviewed and implemented by an authorized HarborTech team member who has 
permission to modify Riverside Goods access. As the intern, my role is to document the evidence,
identify the likely permission gap, and recommend the correct access direction rather than make the production permission change directly.

## Lessons Learned
Week 2 showed that authentication and authorization must be investigated separately. A successful sign-in only proves that an
identity can authenticate. It does not prove that the identity is allowed to perform a specific AWS action.
I also learned that IAM roles have both trust relationships and permission sources.
Trust policies determine who or what may assume a role, while permission policies determine what actions the role may perform
after assumption.
The CLI provided useful evidence because each command produced a specific result that could be interpreted
and compared with the AWS console. I also learned that least privilege requires matching permissions to actual business 
requirements instead of granting broad access simply because it resolves an error quickly.


## Professional Vocabulary
**Authentication:** The process of proving who a user or system is, usually through credentials.
**Authorization:** The process of determining what an authenticated identity is allowed to do.
**IAM:** AWS Identity and Access Management, the service used to control identities, roles, policies, and permissions.
**Policy:** A set of permission rules that defines which AWS actions are allowed or denied.
**Least Privilege:** The security principle of granting only the permissions required to perform a job.
**AccessDenied:** An AWS response indicating that the current user is not authorized to perform the requested action.
**AWS CLI:** The AWS Command Line Interface, which allows AWS services to be investigated and changed using commands.
**CloudShell:** A browser-based AWS shell environment.
**Caller Identity:** The AWS account and identity making an API request.
**Resource Scope:** The specific AWS resources to which a permission applies.

