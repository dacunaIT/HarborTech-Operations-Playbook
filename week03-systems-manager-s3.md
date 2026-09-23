# Week 3: Systems Manager and S3

## HarborTech Ticket Summary
Ticket TKT-2026-0003 involved two operational needs for Bright Path Community Services. The first involved Dana performing the same maintenance task repeatedly across five EC2 instances. The second involved hosting a simple public resource page containing program hours, contact information, images, and downloadable files.

The goal of the investigation was to determine which AWS services were appropriate for these needs, verify the required prerequisites, test S3 static website hosting in the Learner Lab, and document any sandbox restrictions.

## Client Impact
Repeated manual maintenance across multiple EC2 instances can create inconsistent results, take more time, and increase the chance of human error. A centralized management tool can make repeated administration more consistent and efficient.

For Bright Path, using a traditional web server for a simple static public page would add unnecessary infrastructure and maintenance. S3 static website hosting is better suited for static files that do not require server-side processing.

## AWS Services Involved
The main AWS services and features involved in this investigation were:

- AWS Systems Manager
- Run Command
- Session Manager
- Inventory
- Parameter Store
- Amazon S3
- S3 Static Website Hosting
- AWS CloudShell
- AWS CLI
- AWS STS

## Virtualization Connection
AWS Systems Manager provides a centralized management layer for virtual machines such as EC2 instances. Instead of manually connecting to each virtual machine individually, administrators can use Systems Manager features to manage multiple instances from one location.

Amazon S3 supports workloads that do not require a traditional virtual machine or web server. Static website hosting allows HTML, images, CSS, and downloadable files to be served directly from an S3 bucket.

## Evidence Reviewed
I reviewed and documented evidence from both the Systems Manager portion of the assignment and the S3 hands-on investigation.

For Dana's EC2 maintenance requirement, the evidence showed that the same maintenance command needs to be executed consistently across five EC2 instances. Because the task is repeated across several systems, I identified it as a candidate for centralized management rather than manually connecting to and maintaining each instance separately.

I reviewed the Systems Manager prerequisites that must be satisfied before those EC2 instances can be treated as managed nodes. The instances need the SSM Agent installed and running so they can communicate with Systems Manager. They also need an IAM instance role that provides the permissions required for Systems Manager operations. In addition, the instances need network connectivity to the appropriate Systems Manager service endpoints. If any of these prerequisites are missing, Run Command should not be expected to work successfully.

I also reviewed the difference between several Systems Manager capabilities. Run Command is appropriate for sending the same non-interactive command to multiple managed instances. Session Manager is more appropriate when an administrator needs an interactive shell session. Inventory is used to collect information about the software, configuration, and characteristics of managed systems. Parameter Store is used to centrally store configuration values that applications or automation can retrieve by parameter name.

For the Bright Path website requirement, I verified that the requested content was static in nature. The site only needed program hours, contact information, a short service description, images, and downloadable public files. Because there was no requirement for server-side processing, database transactions, authentication, or dynamic user sessions, I determined that S3 static website hosting was an appropriate service model.

During the hands-on portion, I created the S3 bucket `brightpath-diego-week3-2026` in the `us-east-1` Region. I then uploaded the object `index.html` to the root of the bucket. I verified that the upload completed successfully in the S3 console.

I opened the bucket Properties tab and enabled Static Website Hosting. I selected bucket hosting and configured `index.html` as the index document. After saving the configuration, S3 generated the following website endpoint:

`http://brightpath-diego-week3-2026.s3-website-us-east-1.amazonaws.com`

I tested the website endpoint directly in the browser. The request returned `403 Forbidden` with the message `AccessDenied`. This confirmed that the website hosting configuration itself had been enabled, but the object was not publicly readable under the Learner Lab's current access controls.

The assignment instructions stated that Learner Lab public-access controls may be restricted and that denied public-read actions should be documented rather than bypassed. Based on the observed `403 Forbidden` result, I documented the sandbox restriction and did not attempt to change the environment in a way that would bypass those controls.

I then opened AWS CloudShell in the `us-east-1` Region and ran:

`aws sts get-caller-identity`

The command completed successfully and returned the current AWS account, user/session identifier, and ARN for the Learner Lab assumed role. This verified that the AWS CLI session was authenticated and operating under the expected Learner Lab identity.

Next, I created a local working directory in CloudShell using:

`mkdir -p brightpath-site`

I created a revised version of the website file at:

`brightpath-site/index.html`

After updating the file, I synchronized the local directory with the S3 bucket using:

`aws s3 sync ./brightpath-site s3://brightpath-diego-week3-2026/`

The CLI returned upload output showing that:

`brightpath-site/index.html`

was successfully uploaded to:

`s3://brightpath-diego-week3-2026/index.html`

This provided command-line evidence that the website content could be updated through the AWS CLI rather than only through the S3 graphical console.

The final evidence showed that the S3 bucket, object upload, static website configuration, website endpoint, public-access denial, AWS CLI identity verification, and CLI-based site update were all successfully tested and documented within the limits of the Learner Lab environment.

## Operational Analysis
Dana's maintenance task is better suited for centralized automation than manual administration. AWS Systems Manager Run Command can execute the same command across multiple managed EC2 instances. This reduces the need to log in to each server individually and helps ensure that the same maintenance action is performed consistently.

Session Manager would be more appropriate when an administrator needs interactive shell access to an instance. Inventory is used to collect information about managed systems. Parameter Store is useful for storing configuration values centrally so applications or automation can retrieve them by name.

Bright Path's website requirement is suitable for S3 static website hosting because the content consists of static HTML and other public files. The site does not require server-side processing, a database, application authentication, or dynamic user sessions.

## Recommendation
For Dana's repeated EC2 maintenance, I recommend AWS Systems Manager Run Command. It is designed to run commands across one or more managed instances without requiring administrators to manually connect to each system.

Before Run Command should be expected to work, the EC2 instances must meet the required Systems Manager prerequisites. The SSM Agent must be installed and running, the instances must have the correct IAM permissions, and they must have connectivity to the Systems Manager service.

For Bright Path's public resource page, I recommend Amazon S3 static website hosting because the site only contains static content such as program hours, contact information, images, and downloadable files.

## Escalation Notes
The S3 website configuration was completed successfully, but the website endpoint returned `403 Forbidden` with `AccessDenied`.

The Learner Lab appears to restrict the public-read configuration required for S3 static website hosting. I documented the restriction and did not attempt to bypass the sandbox controls.

In a normal AWS account, the required public-access configuration would need to be reviewed and approved before the website could be publicly accessible.

The EC2 instances would also need to be verified as Systems Manager managed nodes before Dana's maintenance task could be automated. Any missing SSM Agent, IAM role, or network configuration would need to be corrected or escalated.

## Lessons Learned
Week 3 showed me that the correct AWS service depends on the type of operational task being performed.

Systems Manager can reduce repeated manual work by providing centralized management and automation for EC2 instances. Run Command is useful for repeated non-interactive maintenance, while Session Manager is more appropriate for interactive access.

I also learned that Parameter Store centralizes configuration values but does not automatically rewrite existing files. Applications or automation must be changed to retrieve the parameter by name.

S3 static website hosting is a good option for simple static content because it removes the need to maintain a separate virtual machine and web server. However, it is not appropriate for applications that require server-side processing, authentication, databases, or dynamic sessions.

The lab also reinforced the importance of respecting sandbox restrictions instead of trying to bypass denied permissions.

## Professional Vocabulary

### Systems Manager
An AWS service that provides centralized management and operational tools for AWS resources such as EC2 instances.

### Managed Node
A machine that has been configured so AWS Systems Manager can manage it.

### SSM Agent
Software running on a machine that allows the instance to communicate with AWS Systems Manager.

### Run Command
A Systems Manager feature that runs commands remotely on one or more managed instances.

### Session Manager
A Systems Manager feature that provides interactive shell access to managed instances without requiring traditional SSH access.

### Inventory
A Systems Manager feature that collects information about managed instances, including software and system configuration data.

### Parameter Store
A Systems Manager capability used to centrally store configuration values or other parameters that applications and automation can retrieve by name.

### Amazon S3
An AWS object storage service used to store files and other objects in buckets.

### Static Website Hosting
An S3 feature that allows static files such as HTML, CSS, images, and documents to be served as a website.

### AWS CLI
A command-line interface used to manage AWS services by running commands from a terminal.

### AWS STS
AWS Security Token Service, which provides temporary security credentials and can be used to identify the current AWS caller.

### Least Privilege
The practice of giving a user, role, or system only the permissions required to complete its assigned task.
