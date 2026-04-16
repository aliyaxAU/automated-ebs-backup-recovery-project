# Automated EBS backup recovery

In this project, I set up EBS snapshots to automatically be created, copied, and cleaned up across two AWS regions: London (`eu-west-2`) and Sao Paulo (`sa-east-1`).

The project demonstrates operational skills like backup automation, disaster recovery, lifecycle management, and restore testing.

The system consists of three Lambda functions triggered by EventBridge and automatically:

- Creates EBS snapshots in the **primary region** (`eu-west-2`, London)
- Replicates snapshots to a **backup region** (`sa-east-1`, Sao Paulo)
- Cleans up snapshots older than **1 day** to control cost

In addition, the documentation includes a full disaster‑recovery procedure that demonstrates how to restore data from the replicated snapshots and validate that the system can be recovered end‑to‑end.

See the full step‑by‑step [implementation and recovery notes here](https://github.com/aliyaxAU/automated-ebs-backup-recovery-project/blob/main/docs/implementation_steps.md).