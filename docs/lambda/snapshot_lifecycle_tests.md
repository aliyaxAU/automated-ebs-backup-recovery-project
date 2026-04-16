# Test Lambda

## Test Snapshot Creation (Primary Region: eu‑west‑2) 

1. Manual test

Invoke Lambda manually:

```console
aws lambda invoke \
>   --function-name createSnapshots \
>   --payload '{}' \
>   response.json && cat response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

Output:

```console
{"status": "success", "snapshots_created": ["snap-06316560007833d66"], "count": 1}
```
After running the command above, check in the AWS Console:

EC2 select Snapshots → Region: `eu‑west‑2 (London)`

I can see a snapshot got created:

![alt text](<img/snapshot created ec2.png>)

2. EventBridge Cron job replication:

I confirm that EventBridge created snapshots automatically

![alt text](<img/create snapshot event bridge - cron.png>)

CloudWatch Logs confirm execution at the correct time (screenshot below taken after a few days):

![alt text](<img/cloudwatch log management create snapshots london.png>)

### ❗Important: CloudWatch Logs Timestamp Behavior

EventBridge Scheduler was configured to run the snapshot Lambda at 01:01 Europe/London. CloudWatch Logs, however, always record timestamps in UTC, regardless of the schedule’s timezone.

Because the UK was in BST (UTC+1) at the time, a 01:01 local execution appears in CloudWatch as 00:01 UTC.

This behaviour is expected and confirms that the function executed at the correct local time.

## Test Snapshot Replication

1. Manual replication:

Before running the command below, the snapshot from eu-west-2 should have a tag `NeedsReplication = true`

![alt text](<img/snapshots eu-west2 replication true.png>)

```console
$ aws lambda invoke   --function-name replicateSnapshots   --payload '{}'   response.json && cat response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

If it's succesfull it should return count 1

Output:

```console
{"status": "replicated", "count": 1, "snapshots": ["snap-034e2891f70a4866a"]}~ $ 
```
Snapshot is appearing in different region Sao Paulo Brazil

![alt text](<img/snapshot different region.png>)

2. EventBridge cron Replication (next days):

Next day, I confirm the replication ran automatically:

![alt text](<img/replicated snapshot successful.jpg>)

As set up, CloudWatch Logs show replication every 15 minutes:

![alt text](<img/cloudwatch log management replicate snapshots london.png>)


## Test Snapshot Cleanup

1. Manual Replication

The cleanup Lambda removes snapshots that are older than one day (the retention period I configured):

```console
~ $ aws lambda invoke   --function-name cleanupSnapshots   --payload '{}'   response.json && cat response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

Output:

```console
{"status": "cleanup", "deleted": ["snap-06316560007833d66", "snap-034e2891f70a4866a"], "count": 2}~ $ 
```
And the snapshot disappeared from:

- eu‑west‑2

![alt text](<img/no snapshot london.png>)

- sa‑east‑1

![alt text](<img/no snapshot sao paulo.png>)


2. EventBridge Scheduled Cleanup (CloudWatch Verification)  

CloudWatch logs show that the cleanup Lambda ran as planned and deleted the expected snapshots during the manual test.

![alt text](<img/cloudwatch log cleanup snapshots.png>)

### ❗Important: CloudWatch Logs Timestamp Behavior
EventBridge schedules use the configured timezone (Europe/London), but CloudWatch Logs always display timestamps in UTC. A 02:00 BST execution appears as 01:00 UTC in CloudWatch.