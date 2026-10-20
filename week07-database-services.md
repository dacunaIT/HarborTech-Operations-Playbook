[week07-database-services.md](https://github.com/user-attachments/files/33736741/week07-database-services.md)
# Week 7: Managed Database Services

## HarborTech Ticket Summary

**Ticket:** TKT-2026-0007 — Database Operations Assessment  
**Client:** Café business running a transactional MySQL database on Amazon EC2  
**Role:** HarborTech Cloud Operations Intern

The client’s database is functioning, but maintaining the guest operating system, applying database patches, performing and verifying backups, and planning recovery take substantial staff time. HarborTech was asked to assess whether continued self-management on EC2 is appropriate or whether a managed database service would be a better operational fit. This is an advisory assessment, not authorization to migrate the client's production database.

The associated AWS Academy Module 6 activity used a separate **Mom & Pop Café** practice environment to migrate a local database to **Amazon RDS for MariaDB**. I used those hands-on results as evidence of managed database deployment and migration processes, not as proof that the HarborTech client's production system was changed.

## Client Impact

Manual patching, server maintenance, and backup verification can increase overtime and operational overhead. Inadequate backup testing or recovery preparation can increase data-loss and downtime risks. A suitable managed relational database could reduce repetitive infrastructure work, while an incompatible service choice or poorly planned migration could interrupt café transactions. Availability, costs, compatibility, and recovery requirements must be assessed before recommending a production change.

## AWS Services Involved

- **Amazon EC2:** Hosts the current café application and its original self-managed database in the training environment.
- **Amazon RDS:** Managed relational database service evaluated for reduced maintenance; the activity provisioned an RDS MariaDB instance.
- **Amazon Aurora:** MySQL-compatible managed relational option to compare against standard RDS when performance, scale, availability, and budget justify it.
- **Amazon DynamoDB:** Managed NoSQL service suited to independent, key-accessed records without relational joins; not a direct drop-in replacement for the café's transactional relational database.
- **Amazon VPC, subnets, and security groups:** Control network placement and database connectivity.
- **AWS Systems Manager Parameter Store:** Stores the café application's database endpoint setting (`/mompopcafe/dbUrl`).
- **Amazon CloudWatch:** Supplies RDS monitoring metrics such as database connections, CPU utilization, and storage capacity; live monitoring was not verified in my recorded results.
- **AWS Database Migration Service (DMS):** An option to evaluate for a larger or low-downtime migration; it was **not** used in this activity.

## Virtualization Connection

An EC2 instance virtualizes computing hardware, but the customer still manages the guest operating system and database software. Amazon RDS abstracts more infrastructure and routine database administration, including underlying host management and supported patching and backup functions. The customer still owns schema design, application compatibility, access decisions, database configuration, and recovery requirements. A database subnet group spanning two Availability Zones supports RDS placement requirements; it does **not** by itself make a deployment Multi-AZ.

## Evidence Reviewed

The HarborTech service request identifies a MySQL transactional workload using structured tables and SQL queries. It does **not** supply actual transaction volumes, recovery time objective (RTO), recovery point objective (RPO), migration budget, or detailed compatibility requirements.

In the separate AWS Academy lab, I collected the following environment information:

| Item | Observed value |
| Region | `us-east-1` |
| Café EC2 instance type | `t2.small` |
| Café EC2 Availability Zone | `us-east-1a` |
| VPC | `vpc-04508b49b3a42a1bf` |
| VPC CIDR | `10.200.0.0/20` |
| Existing public subnet CIDR | `10.200.0.0/24` |
| New database subnet 1 | `subnet-0cc64c7762aae699d` in `us-east-1a`, `10.200.2.0/23` |
| New database subnet 2 | `subnet-0812b963406fe7499` in `us-east-1b`, `10.200.10.0/23` |
| Database security group | `sg-070017f0c047b1b09` |
| RDS instance | `MomPopCafeDBInstance` — MariaDB `10.6.28`, `db.t3.micro`, 20 GB |
| RDS status | `available` |
| RDS backup retention | 1 day (observed lab setting) |
| Café website before migration | 2 orders, totaling $48.00 |
| Migrated `product` table | 9 rows returned |

The security group verification showed **TCP port 3306** allowed from the café application's security group (`sg-0a23056e36437c049`) rather than open to the public internet. The RDS instance was created as **not publicly accessible**.

## AWS Academy Activity: Actions and Results

### 1. Environment discovery and network preparation

I connected to the CLI Host using PuTTY and the AWS Academy-provided SSH key, configured the AWS CLI for `us-east-1`, and used `aws sts get-caller-identity` to confirm the lab identity. I queried EC2 instances, VPC CIDR ranges, existing subnets, and available Availability Zones to plan non-overlapping private address ranges.

I created a database security group and used a source security group reference for the MySQL inbound rule. I then verified that `FromPort` and `ToPort` were `3306`, `IpProtocol` was `tcp`, and the allowed source was the café EC2 security group.

Commands used included:

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=MomPopCafeInstance"
aws ec2 describe-vpcs --vpc-ids vpc-04508b49b3a42a1bf --query 'Vpcs[*].CidrBlock'
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-04508b49b3a42a1bf" --query 'Subnets[*].[SubnetId,CidrBlock]'
aws ec2 describe-security-groups --group-ids sg-070017f0c047b1b09 --query 'SecurityGroups[*].[GroupName,GroupId,IpPermissions]'
```

I created two new subnets in different Availability Zones and an RDS database subnet group. The returned subnet statuses were **Active**, and the database subnet group status was **Complete**.

### 2. RDS engine compatibility issue and resolution

The lab's provided RDS creation command specified MariaDB **10.6.14**. My first attempt failed with:

```text
InvalidParameterCombination: Cannot find version 10.6.14 for mariadb
```

I did not keep retrying the unsupported engine version. I checked engine availability and instance-class compatibility:

```bash
aws rds describe-db-engine-versions \
  --engine mariadb \
  --query 'DBEngineVersions[*].EngineVersion' \
  --output table

aws rds describe-orderable-db-instance-options \
  --engine mariadb \
  --engine-version 10.6.28 \
  --db-instance-class db.t3.micro \
  --query 'OrderableDBInstanceOptions[*].[EngineVersion,DBInstanceClass]' \
  --output table
```

**Actual result:** The available engine-version list included `10.6.28`, and the instance-options query returned `10.6.28` with `db.t3.micro`. I retried creation using this validated combination. A later `describe-db-instances` response reported an endpoint, `us-east-1a`, a backup window of `08:45–09:15` UTC, a backup retention period of **1 day**, and status **`available`**.

**Interpretation:** AWS service versions change, so documented lab commands must be checked against the actual supported options. The instance became available, but this status alone did not prove application migration success.

### 3. Backup and database migration

I connected to the Mom & Pop Café EC2 application server, exported its local database, and inspected the SQL backup file:

```bash
mysqldump --user=root --password='[REDACTED]' \
  --databases mom_pop_db --add-drop-database > mompopdb-backup.sql

ls -lh mompopdb-backup.sql
grep -E 'CREATE TABLE|INSERT INTO' mompopdb-backup.sql
```

**Actual result:** The backup file existed and was approximately **5.9 KB**. It contained `CREATE TABLE` and `INSERT INTO` statements for `order`, `order_item`, `product`, and `product_group`, including the pre-migration order records.

My first import attempt failed with `ERROR 2005 (HY000): Unknown MySQL server host` because the endpoint entered was incorrect. I checked the endpoint shown in the RDS console under **Connectivity & security**, corrected the host value, and retried:

```bash
mysql --user=root -p \
  --host='[VERIFIED-RDS-ENDPOINT]' \
  < mompopdb-backup.sql
```

**Actual result:** The corrected import returned to the shell without an error. I connected interactively to the target database and ran:

```sql
SELECT * FROM product;
```

**Actual result:** `9 rows in set (0.00 sec)`.

**Interpretation:** This verifies that the migrated RDS database was accessible and that the expected nine product records were present. It does not, by itself, verify all tables, complete data integrity, or the application's post-cutover behavior.

### 4. Permissions restriction and incomplete application cutover

To look up the RDS endpoint from the café EC2 server, I attempted:

```bash
aws rds describe-db-instances \
  --db-instance-identifier MomPopCafeDBInstance \
  --region us-east-1 \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
```

**Actual result:** The first attempt without an explicit Region requested Region configuration. After adding `--region us-east-1`, the command returned **`AccessDenied`** for `rds:DescribeDBInstances` under the instance's assumed IAM role. I did not alter the role or attach additional permissions; I obtained the endpoint through the permitted RDS console view instead.

The activity then instructed changing the Systems Manager Parameter Store value **`/mompopcafe/dbUrl`** so the website would point to RDS. **I was restricted from changing that parameter in the Learner Lab**, so I did not perform or claim a successful application endpoint switch.

**Result and limitation:** The RDS database was provisioned and its product data was verified through a direct database connection. However, the website was **not** confirmed to be using RDS. Although the site showed **two orders before migration**, I could not complete the required before-and-after website verification. The live CloudWatch **DB Connections** demonstration was also not completed in the recorded work.

## Operational Analysis

The client's structured SQL tables and transactions are a relational workload. **Amazon RDS for MySQL** is the first service I would evaluate because it preserves the general relational model while reducing administration of underlying infrastructure and routine database maintenance. RDS automated backups and configurable maintenance features can reduce repetitive work, but backup retention, restoration tests, patch scheduling, and recovery design still require customer decisions.

**Aurora MySQL-Compatible Edition** should also be assessed if the client's performance, scaling, or availability needs justify its additional features and cost. It should not be assumed to be the best option without workload evidence. **DynamoDB** is a better conceptual fit for separate customer-preference records retrieved by known keys, with no join requirement and variable traffic; it would not be a straightforward replacement for this transactional relational application.

The training lab used **RDS for MariaDB**, which demonstrates similar managed relational database operations but does not independently establish compatibility with the client's production **MySQL** application. The correct production engine and version must be validated.

## Recommendation

**Recommend evaluating Amazon RDS for MySQL first**, followed by a nonproduction proof of concept. Compare it with Aurora MySQL-Compatible Edition for measured performance, availability, scaling, and cost requirements. Continue on EC2 if the client requires unsupported database configurations or operating-system-level access.

Before recommending production migration, collect:

- Actual traffic, peak transactions, query patterns, storage growth, and performance metrics.
- MySQL version, schema, extensions, stored procedures, dependencies, and application compatibility.
- Required availability, **RTO**, **RPO**, backup retention, and tested restore procedures.
- Estimated costs, migration budget, allowed downtime, connectivity, security, and rollback requirements.

Assess whether AWS DMS or a dump/restore method best fits the validated downtime and data-volume constraints. Test the full application against a nonproduction target, compare record counts and representative transactions, and confirm rollback capability before requesting approval. AWS operates managed infrastructure; the client remains accountable for application code, credentials and access design, schema, data, and business continuity decisions.

## Escalation Notes

- **Learner Lab restriction:** I could not update `/mompopcafe/dbUrl` in Systems Manager Parameter Store. Instructor or lab administrator assistance would be required to authorize that training step. I did not bypass the restriction.
- **IAM boundary:** The café EC2 instance role was denied `rds:DescribeDBInstances`. I used the RDS console to look up the endpoint and did not request excessive IAM permissions merely to complete an information lookup.
- **Production authority:** HarborTech supervisor and client approval are required before any real production engine change, database migration, application reconfiguration, downtime window, or cutover.
- **Unresolved verification:** The site was not redirected to RDS, so post-cutover application behavior and before-and-after order-count consistency remain unverified. RDS monitoring observations also remain to be collected.

## Lessons Learned

This activity demonstrated why managed databases can reduce infrastructure responsibilities without removing the customer's ownership of data and applications. Database service selection depends on workload fit, not popularity. I also learned that training instructions can contain outdated engine versions, so I should check supported configurations before deploying. Finally, an available database and a successful SQL query are narrower evidence than a fully tested application migration. Permission denials are operational boundaries to document and escalate, not workarounds to bypass.

## Professional Vocabulary

- **Relational database:** Stores structured data in related tables and supports SQL queries and transactions.
- **NoSQL:** Nonrelational data models, often designed around specific access patterns rather than joins.
- **Amazon RDS:** Managed relational database service that handles supported infrastructure and routine operational tasks.
- **Amazon Aurora:** AWS-managed relational engine with MySQL- and PostgreSQL-compatible editions and distinct performance and availability features.
- **Amazon DynamoDB:** Managed NoSQL database designed for scalable key-based data access.
- **Multi-AZ:** Database deployment architecture designed for higher availability across Availability Zones; not enabled merely by creating subnets in multiple zones.
- **AWS DMS:** Database Migration Service, used to help migrate and replicate data between supported database endpoints.
- **Schema conversion:** Adapting database structures or SQL features when moving between different engines or versions.
- **Managed service:** A service where the provider performs defined infrastructure and maintenance responsibilities.
- **Workload fit:** Selecting a service according to actual data structures, access patterns, performance, availability, compatibility, and cost requirements.
- **RTO / RPO:** Recovery time objective (acceptable restoration time) and recovery point objective (acceptable potential data loss measured in time).
