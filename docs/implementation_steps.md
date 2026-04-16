# Implementation Steps

This document describes the full step-by-step process used to build the Automated EBS Backup & Recovery System.

---

## Create Snapshots Lambda

This Lambda:
- Automatically creates EBS snapshots in the primary region daily at 01:00 (UTC)
- Returns a list of created snapshot IDs
- Logs all activity to CloudWatch for operational visibility


More details:

[Create Snapshot implementation steps](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/create_snapshot_and_upload.md#step-2-build-and-upload-files-for-lambda-code-for-createsnapshots-function)

[Testing the Snapshot Creation Lambda](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/snapshot_lifecycle_tests.md#test-snapshot-creation-primary-region-euwest2)

---

## Cross-Region Replication Lambda

This Lambda:
- Finds snapshots created by the system
- Copies them from the primary region to the backup region
- Logs all replication activity to CloudWatch for auditing


More details:

[Replicate Snapshot implementation steps](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/create_snapshot_and_upload.md#step-2-build-and-upload-files-for-lambda-code-for-createsnapshots-function)

[Testing the Replicate Snapshot Lambda](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/snapshot_lifecycle_tests.md#test-snapshot-creation-primary-region-euwest2)

---

## Cleanup Lambda

This Lambda:
- Finds snapshots older than 1 day
- Ensures only the most recent backups remain available
- Logs cleanup actions to CloudWatch for visibility and verification

More details:

[Cleanup Snapshot implementation steps](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/create_snapshot_and_upload.md#step-2-build-and-upload-files-for-lambda-code-for-createsnapshots-function)

[Testing the Cleanup Snapshot Lambda](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/snapshot_lifecycle_tests.md#test-snapshot-creation-primary-region-euwest2)

---

## EventBridge Scheduling

Three schedules are created:

### 1. Daily Snapshot Creation

- Rule Frequency: Daily at 01:00 (UTC)
- This schedule runa Lambda once per day to generate fresh EBS snapshots in the primary region (`eu‑west‑2`)

[More details ](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/eventbridge_configuration/eventbridge_configurarion.md#createsnapshots-schedule)

### 2. Snapshot recreation

- Rule Frequency: Every 15 minutes 
- This schedule runs the replication Lambda to find new snapshots and copy them to the disaster-recovery region (`sa-east-1`)

[More details](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/eventbridge_configuration/eventbridge_configurarion.md#replicatesnapshots)

### 3. Daily Cleanup

- Rule Frequency: Daily at 02:00 (UTC)
- This schedule runs the cleanup Lambda once a day to make sure that snapshot retention policies are followed in both regions.

[More details](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/eventbridge_configuration/eventbridge_configurarion.md#cleanup-snapshots)

---
