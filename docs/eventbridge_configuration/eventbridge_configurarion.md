# EventBridge Setup

I'm going to make schedules for the lambda functions I created in the [create snapshot and upload step.](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/lambda/create_snapshot_and_upload.md)

In AWS `Amazon Event Bridge` select `Schedules` and create schedules.

## CreateSnapshots schedule

Type: `EventBridge Scheduled Rule`
Frequency: `Daily at 01:00 (UTC)`

![alt text](<img/createsnapshots eventbridge.png>)

Target: `createSnapshots Lambda (eu‑west‑2)`

![alt text](<img/createsnapshots eventbridge target.png>)

## ReplicateSnapshots

Every 15 minutes this schedule runs the replication Lambda to find new snapshots and copy them to the disaster-recovery region (`sa-east-1`). The short time between backups makes sure that they are copied soon after they are made, which lowers the Recovery Point Objective (RPO). This schedule turns the system into a backup pipeline that works in multiple regions and can handle faults, making it useful for disaster recovery.

Type: `EventBridge Recurring Schedule`

Every 15 minutes

![alt text](<img/eventbridge replicate.png>)

![alt text](<img/replicatesnapshots eventbridge target.png>)

## Cleanup Snapshots

Type: `EventBridge Scheduled Rule`
Frequency: `Daily at 02:00 (UTC)`

This schedule runs the cleanup Lambda once a day to make sure that snapshot retention policies are followed in both regions. Snapshots which are older than one day are automatically found and deleted. This keeps storage costs predictable and under control.

 This automated step in lifecycle management shows good operational discipline and cost-cutting practices.

![alt text](<img/eventbridge cleanup.png>)

![alt text](<img/eventbridge cleanup target.png>)