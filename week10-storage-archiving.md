week10-storage-archiving

## HarborTech Ticket Summary

**Ticket:** TKT-2026-0010  
**Client:** Bright Path Community Services  
**Environment:** AWS Academy Learner Lab, `us-east-1`  
**Scope:** Storage and recovery investigation; the lab resources are separate from the client's production resources.

Bright Path reported that its newest *documented* EC2 snapshot is 19 days old despite a maximum acceptable data loss of 24 hours. Three Linux servers need concurrent access to working files, uploaded documents require durable object storage, and completed records are rarely used after 90 days but must be retained for seven years. The objective is to evaluate AWS storage options, document direct CLI evidence, and recommend a recovery and retention approach without treating student lab operations as changes to the client environment.

## Client Impact

A 19-day-old documented snapshot does not demonstrate compliance with a 24-hour recovery point objective (RPO). A failure could result in unacceptable data loss unless other, newer recovery points exist. Using the wrong storage model could also interfere with shared file access or record retention. Seven-year recordkeeping requires deliberate lifecycle, access-control, and deletion-protection decisions.

## Investigation 1: AWS Environment Verification

### Commands Executed

```bash
aws sts get-caller-identity
aws --version
aws configure get region
echo "$AWS_DEFAULT_REGION"
```

### Actual CloudShell Evidence

- `aws sts get-caller-identity` returned an assumed-role session under `vocLabs` (account and personal session identifiers omitted from this public report).
- AWS CLI version: `aws-cli/2.39.5`.
- `aws configure get region` returned no configured value.
- `AWS_DEFAULT_REGION` was `us-east-1`.

### Analysis

**1. What do identity and Region results establish?** The identity command confirmed that CloudShell was authenticated using a lab role. The default Region environment variable identified `us-east-1` as the Region used for the investigation, even though the CLI configuration did not have a Region value set.

**2. Why verify account and Region?** Account and Region checks reduce the risk of creating or modifying resources in the wrong environment. They also make resource discovery and cleanup more reliable, especially when the configured Region is blank.

## Investigation 2: EBS Volume Investigation

### Commands Executed

```bash
aws ec2 describe-availability-zones --region us-east-1 --query 'AvailabilityZones[*].[ZoneName,State]' --output table
aws ec2 describe-volumes --region us-east-1 --query 'Volumes[*].[VolumeId,State,Size,VolumeType,AvailabilityZone]' --output table
aws ec2 create-volume --availability-zone us-east-1d --size 1 --volume-type gp3 --tag-specifications 'ResourceType=volume,Tags=[{Key=Purpose,Value=IT4560Week10}]' --region us-east-1
```

### Actual CloudShell Evidence

- Availability Zones `us-east-1a` through `us-east-1f` were listed as available.
- An existing **8 GiB gp3** volume was listed as `in-use` in `us-east-1d`; it was not the disposable practice volume and was not modified.
- Created practice volume: `vol-0a5c9819c1201c608`.
- Creation response: `gp3`, **1 GiB**, `us-east-1d`, initially `creating`, 3,000 IOPS, 125 MiB/s throughput, tag `Purpose=IT4560Week10`, encryption reported as `false`.
- Later inspection showed the practice volume as `available`, meaning it was not attached to an instance.

### Analysis

**1. What do state, size, type, and AZ indicate?** The small gp3 volume was a disposable block-storage test in a single Availability Zone. `creating` was the initial provisioning state; `available` later confirmed it existed and was unattached. An EBS volume is generally attached to a compatible EC2 instance in its Availability Zone, unlike a shared multi-server file system.

**2. Risks of selecting the wrong volume?** Snapshotting the wrong volume could back up irrelevant data and leave the intended application unprotected. Restoring or changing the wrong volume could cause downtime, incorrect data recovery, or unintended changes to another workload. Resource IDs, tags, attachment state, and ownership should be checked first.

## Investigation 3: EBS Snapshot and Recovery

### Commands Executed

```bash
aws ec2 create-snapshot --volume-id vol-0a5c9819c1201c608 --description 'IT4560 Week10 practice' --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Purpose,Value=IT4560Week10}]' --region us-east-1
aws ec2 describe-snapshots --snapshot-ids snap-0a91ace069efe7c94 --region us-east-1 --query 'Snapshots[*].[SnapshotId,State,Progress,VolumeId]' --output table
```

### Actual CloudShell Evidence

- Snapshot: `snap-0a91ace069efe7c94`.
- Source volume: `vol-0a5c9819c1201c608`.
- Snapshot status: `completed`, progress `100%`.
- No restore operation was performed; restore success was **not** verified.

### Analysis

**1. What does snapshot status show?** `completed` and `100%` show that AWS finished creating the practice EBS snapshot, providing a snapshot recovery point for that test volume. This does not establish whether a restored application would work correctly.

**2. Why does one successful snapshot not satisfy the client's RPO?** The practice snapshot belongs to the learner account, not Bright Path's production environment. A single completed snapshot also does not establish an ongoing backup schedule or prove that recovery points remain no more than 24 hours apart. Bright Path needs verified, monitored, appropriately frequent backups and recovery testing.

## Investigation 4: S3 Object Storage

### Commands Executed

```bash
BUCKET="it4560-week10-1794351389"
aws s3api create-bucket --bucket "$BUCKET" --region us-east-1
printf 'IT4560 Week10 practice\n' > week10-sample.txt
aws s3 cp week10-sample.txt "s3://$BUCKET/week10-sample.txt" --region us-east-1
aws s3api head-object --bucket "$BUCKET" --key week10-sample.txt --region us-east-1
```

### Actual CloudShell Evidence

- Bucket created: `it4560-week10-1794351389` in `us-east-1`.
- Upload succeeded: `week10-sample.txt`.
- `head-object` returned `ContentLength: 23`, `ContentType: text/plain`, `ServerSideEncryption: AES256`, and object metadata.
- An earlier upload attempt failed because `$BUCKET` was unset, producing an invalid empty bucket name. After setting the variable and creating the bucket, the upload succeeded.

### Analysis

**1. What did bucket and object inspection confirm?** The successful upload and metadata inspection confirmed that a named object was stored in an S3 bucket and could be addressed by bucket and object key. The response also showed server-side encryption metadata. This is a successful metadata check, not a comprehensive permission or durability test.

**2. How does object storage differ from block storage?** S3 stores objects accessed through API operations and object keys, making it suitable for uploaded documents and archives. EBS supplies block-level storage for EC2 workloads. The practice EBS volume was AZ-specific; the S3 object was accessed using its bucket and key without attaching a disk.

## Investigation 5: S3 Versioning and Lifecycle

### Commands Executed

```bash
printf 'IT4560 Week10 practice - version 2\n' > week10-sample.txt
aws s3 cp week10-sample.txt "s3://$BUCKET/week10-sample.txt" --region us-east-1
aws s3api list-object-versions --bucket "$BUCKET" --prefix week10-sample.txt --region us-east-1 --query 'Versions[*].[Key,VersionId,IsLatest,Size]' --output table
aws s3api get-bucket-lifecycle-configuration --bucket "$BUCKET" --region us-east-1
```

### Actual CloudShell Evidence

- Re-upload succeeded.
- The version listing showed a current **35-byte** object with a generated version ID and a prior **23-byte** object with version ID `null`.
- `get-bucket-lifecycle-configuration` returned `NoSuchLifecycleConfiguration`: no lifecycle configuration existed at the time of inspection.
- The visible version list supports that versioning was enabled for the later upload, but the exact versioning-enable command was not captured in the supplied evidence and should not be presented as an executed command without checking the CloudShell history.

### Analysis

**1. What do versioning and lifecycle results demonstrate?** S3 preserved a previous object version after a newer upload, illustrating how versioning can protect against accidental overwrites. The missing lifecycle configuration means there was no demonstrated automated transition or expiration policy for this practice bucket.

**2. What if an expiration rule ignored seven-year retention?** A premature expiration rule could delete required records before their retention period ends, potentially violating client policy or legal requirements. The client must approve retention dates, version handling, deletion rules, and any compliance protections before a lifecycle policy is applied.

## Investigation 6: HarborTech Storage Recommendation

### Commands Executed

```bash
aws ec2 describe-volumes --region us-east-1 --query 'Volumes[*].[VolumeId,State,Size,VolumeType]' --output table
aws ec2 describe-snapshots --snapshot-ids snap-0a91ace069efe7c94 --region us-east-1 --query 'Snapshots[*].[SnapshotId,State,Progress]' --output table
aws efs describe-file-systems --region us-east-1
```

### Actual CloudShell Evidence

- Existing volume: `vol-08227e90e182ef702`, `in-use`, 8 GiB, gp3 (left untouched).
- Practice volume: `vol-0a5c9819c1201c608`, `available`, 1 GiB, gp3.
- Practice snapshot: `snap-0a91ace069efe7c94`, `completed`, `100%`.
- EFS discovery returned `"FileSystems": []`: no EFS file systems were found in this lab Region/account. EFS was **not** deployed or tested.

### Analysis

**1. Which storage models best satisfy the requirements?** Use EBS for EC2 instance block storage and an automated, monitored snapshot/backup policy for recovery. Evaluate **Amazon EFS** for the three Linux servers needing simultaneous shared file access; the lab had no EFS system to test. Use **Amazon S3** for uploaded documents. For completed records, consider S3 lifecycle transitions to an appropriate Glacier storage class after 90 days, with retrieval-time and retention requirements verified. The EBS snapshot and S3 upload succeeded in the lab, but neither proves production readiness.

**2. Which decisions need verification or approval?** Confirm Bright Path's actual latest recovery points, backup frequency, recovery scope, 24-hour RPO, recovery-time objective (RTO), restore procedure, and monitoring. Verify EFS networking, access permissions, throughput, and cost. Confirm document access patterns, archive retrieval deadlines, exact seven-year retention rules, legal holds, S3 Object Lock requirements, versioning, encryption, lifecycle behavior, and authorized migration windows. Obtain approval before changing production data or retention policies.

### Recommended Client Architecture

| Client need | Proposed AWS approach | Reason and verification needed |
|---|---|---|
| EC2 application disks | EBS gp3 | Instance block storage; verify sizing, encryption, and backup scope |
| Maximum 24-hour data loss | Scheduled AWS Backup/EBS recovery points, monitoring, restore drills | A 19-day-old documented snapshot is insufficient evidence of RPO compliance |
| Three Linux servers sharing working files | Amazon EFS | Managed shared NFS file system; test connectivity, security, performance, and permissions |
| Uploaded documents | Amazon S3 | Durable object storage with bucket policies, encryption, and versioning as appropriate |
| Records rarely read after 90 days, retained seven years | S3 lifecycle transition to an appropriate Glacier class, with retention controls | Balance archive cost and retrieval time; prevent premature deletion and confirm compliance |

**Priority:** Investigate and remediate the backup/RPO gap first, then validate shared access and document storage, and only then implement approved archival automation.

## Investigation 7: Resource Cleanup

### Commands Executed and Verification

The practice snapshot, unattached volume, and versioned S3 bucket were deleted. The exact S3 object-version deletion steps were not fully preserved in the supplied screenshots; consult CloudShell history before adding those commands to the execution record.

```bash
aws ec2 delete-snapshot --snapshot-id snap-0a91ace069efe7c94 --region us-east-1
aws ec2 delete-volume --volume-id vol-0a5c9819c1201c608 --region us-east-1
aws ec2 describe-snapshots --snapshot-ids snap-0a91ace069efe7c94 --region us-east-1
aws ec2 describe-volumes --volume-ids vol-0a5c9819c1201c608 --region us-east-1
aws s3api delete-bucket --bucket "$BUCKET" --region us-east-1
aws s3api head-bucket --bucket "$BUCKET" --region us-east-1
```

### Actual CloudShell Evidence

- `describe-volumes` returned `InvalidVolume.NotFound` for `vol-0a5c9819c1201c608`.
- `describe-snapshots` returned `InvalidSnapshot.NotFound` for `snap-0a91ace069efe7c94`.
- `head-bucket` returned `404 Not Found` for `it4560-week10-1794351389` after bucket deletion.
- The pre-existing 8 GiB volume was not deleted.

### Analysis

**1. What confirms cleanup?** Follow-up lookups could no longer find the disposable EBS volume or snapshot, and S3 returned 404 for the practice bucket. These are the observed verification results supporting removal of the lab resources.

**2. Why verify cleanup?** Unused resources may generate charges and increase security exposure. Checking IDs, tags, and post-deletion status reduces accidental impact on shared resources and provides an audit trail that temporary assets were removed.

## Virtualization and Operational Connection

EBS provides virtual block devices to EC2 instances; snapshots capture point-in-time volume data for recovery. EFS provides shared file-system access across compatible Linux clients, while S3 exposes objects through an API rather than a mounted EBS disk. Choosing between these models depends on access pattern, concurrency, durability, recovery objectives, and retention—not simply the lowest storage price.

## Escalation Notes

- Escalate the 19-day documented snapshot age against the client's 24-hour RPO immediately; verify whether more recent backups exist.
- Ask the client to approve a backup policy and demonstrate restore testing before claiming recovery readiness.
- Request security and records-management approval for seven-year retention, deletion protection, encryption, and lifecycle transitions.
- Do not treat learner-lab test resources as production fixes. No client resources were modified.

## Lessons Learned

This lab demonstrated that a successful CLI operation and a business requirement are different kinds of evidence. A completed snapshot does not prove an RPO or a successful restore; an S3 object upload does not establish a seven-year retention policy. Explicit Region selection, resource identification, object version awareness, and verified cleanup are essential cloud-operations practices.

## Professional Vocabulary

- **RPO (Recovery Point Objective):** Maximum tolerable period of lost data.
- **RTO (Recovery Time Objective):** Target time to restore service.
- **EBS:** EC2-oriented block storage.
- **EFS:** Managed shared file storage using NFS.
- **S3:** Object storage using buckets and keys.
- **Snapshot:** Point-in-time EBS backup artifact.
- **Versioning:** Preservation of multiple object versions.
- **Lifecycle policy:** Rules that transition or expire stored objects.
- **Archive storage:** Lower-cost storage for infrequently accessed data, often with retrieval delays.
- **Least privilege:** Granting only the permissions needed for an operation.

---

**Evidence note:** Identifiers and statuses above were taken from the supplied CloudShell screenshots. Commands are included where the operation was shown or confirmed by the investigation; the exact commands for enabling S3 versioning and deleting its individual versions should be checked against shell history before asserting they were executed verbatim. Account numbers, user session IDs, and credentials are intentionally excluded from this public GitHub report.
torage-archiving.md…]()
