# Create Snapshot and Upload to Lambda

In the following section, I make a new snapshot in Lambda and build and upload the code for the Lambda function.

**❗Important: Sensitive identifiers (account ID, IAM role names) are intentionally replaced with placeholders.**

## Step 1 Set up Snapshot Lambda

This Lambda runs in `eu‑west‑2` (London) and creates snapshots for all EBS volumes

I will be creating Lambda  with the followings settings:

- Author from scratch
- Name: `CreateSnapshots`
- Runtime: `Python 3.14`
- Architecture: `x86_64`

![alt text](<img/create lambda function in eu-west2 region.png>)

In the next step I will get the files ready so that they can be upload to the lambda function.

## Step 2 Build and Upload files for Lambda code for createSnapshots function.

I am going to create a new file `lambda_function.py` in CloudShell and upload to lambda.


The content:

```python
import boto3
from datetime import datetime

ec2 = boto3.client("ec2", region_name="eu-west-2")

def lambda_handler(event, context):
    volumes = ec2.describe_volumes()["Volumes"]
    snapshot_ids = []

    for vol in volumes:
        volume_id = vol["VolumeId"]

        snapshot = ec2.create_snapshot(
            VolumeId=volume_id,
            Description=f"Automated snapshot for {volume_id}",
            TagSpecifications=[
                {
                    "ResourceType": "snapshot",
                    "Tags": [
                        {"Key": "CreatedBy", "Value": "SnapshotAutomation"},
                        {"Key": "VolumeId", "Value": volume_id},
                        {"Key": "SourceRegion", "Value": "eu-west-2"},
                        {"Key": "NeedsReplication", "Value": "True"},
                        {"Key": "Timestamp", "Value": datetime.utcnow().isoformat()}
                    ]
                }
            ]
        )

        snapshot_ids.append(snapshot["SnapshotId"])

    return {
        "status": "created",
        "snapshots": snapshot_ids,
        "count": len(snapshot_ids)
    }
```

 then the file needs to be zipped:
 
` zip function.zip lambda_function.py`

and deployed:

```console
aws lambda create-function \
  --function-name createSnapshots \
  --runtime python3.12 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
 \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip
```

then invoked:

```console
aws lambda invoke \
  --function-name createSnapshots \
  --payload '{}' \
  response.json && cat response.json
```

and the snapshot got created successfully:

```console
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
{"status": "success", "snapshots_created": [], "count": 0}~ $ 
```

## Step 3. Creating Replicate snapshots function

Purpose of this step is to get snapshots created by (`createSnapshots`) Lambda and copy them from `eu-west-2` to `sa-east-1`

I will create a new file in CloudShell `replicate.py` with the following content:

```python
import boto3

SOURCE_REGION = "eu-west-2"
TARGET_REGION = "sa-east-1"

ec2_source = boto3.client("ec2", region_name=SOURCE_REGION)
ec2_target = boto3.client("ec2", region_name=TARGET_REGION)

def lambda_handler(event, context):
    snapshots = ec2_source.describe_snapshots(
        Filters=[
            {"Name": "tag:CreatedBy", "Values": ["SnapshotAutomation"]},
            {"Name": "tag:NeedsReplication", "Values": ["True"]}
        ],
        OwnerIds=["self"]
    )["Snapshots"]

    copied = []

    for snap in snapshots:
        snap_id = snap["SnapshotId"]

        response = ec2_target.copy_snapshot(
            SourceRegion=SOURCE_REGION,
            SourceSnapshotId=snap_id,
            Description=f"Replicated snapshot of {snap_id}"
        )

        new_snap_id = response["SnapshotId"]
        copied.append(new_snap_id)

        ec2_target.create_tags(
            Resources=[new_snap_id],
            Tags=[
                {"Key": "CreatedBy", "Value": "SnapshotAutomation"},
                {"Key": "ReplicatedFrom", "Value": snap_id},
                {"Key": "SourceRegion", "Value": SOURCE_REGION}
            ]
        )

        ec2_source.create_tags(
            Resources=[snap_id],
            Tags=[{"Key": "NeedsReplication", "Value": "False"}]
        )

    return {
        "status": "replicated",
        "count": len(copied),
        "snapshots": copied
    }

```

Then will zip, deploy and replicate the function

`zip replicate.zip replicate.py` 

```console
aws lambda create-function \
  --function-name replicateSnapshots \
  --runtime python3.12 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --handler replicate.lambda_handler \
  --zip-file fileb://replicate.zip
```

and invoke

```python
 aws lambda invoke   --function-name replicateSnapshots   --payload '{}'   response.json && cat response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
{"status": "replicated", "count": 0, "snapshots": []}

```

## Step 4. Create Cleanup Snapshot

Cleanup snapshots are used to get rid of old EBS snapshots so that AWS doesn't have to pay for more storage space over time.

I'm going to create fille `cleanup.py` with the following content:

`import boto3
from datetime import datetime, timedelta, timezone

REGIONS = ["eu-west-2", "sa-east-1"]
RETENTION_DAYS = 1

def lambda_handler(event, context):
    cutoff = datetime.now(timezone.utc) - timedelta(days=RETENTION_DAYS)
    deleted = []

    for region in REGIONS:
        ec2 = boto3.client("ec2", region_name=region)

        snapshots = ec2.describe_snapshots(
            Filters=[{"Name": "tag:CreatedBy", "Values": ["SnapshotAutomation"]}],
            OwnerIds=["self"]
        )["Snapshots"]

        for snap in snapshots:
            if snap["StartTime"] < cutoff:
                ec2.delete_snapshot(SnapshotId=snap["SnapshotId"])
                deleted.append(snap["SnapshotId"])

    return {
        "status": "cleanup",
        "deleted": deleted,
        "count": len(deleted)
    }`


and will zip:

`zip cleanup.zip cleanup.py`

```python
updating: cleanup.py (deflated 49%)
~ $ aws lambda update-function-code \
>   --function-name cleanupSnapshots \
>   --zip-file fileb://cleanup.zip
{
    "FunctionName": "cleanupSnapshots",
    "FunctionArn": "arn:aws:lambda:eu-west-2:<ACCOUNT_ID>:function:cleanupSnapshots",
    "Runtime": "python3.12",
    "Role": "arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>",
    "Handler": "cleanup.lambda_handler",
    "CodeSize": 598,
    "Description": "",
    "Timeout": 30,
    "MemorySize": 256,
    "LastModified": "2026-04-10T14:17:13.000+0000",
    "CodeSha256": "8ItqfK9nrD2iwiLu4sj/NXqnsqO061MTpzRMVo9B3N4=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "eee74286-3c7c-40d9-8a0d-cd23ddc82844",
    "State": "Active",
    "LastUpdateStatus": "InProgress",
    "LastUpdateStatusReason": "The function is being created.",
    "LastUpdateStatusReasonCode": "Creating",
    "PackageType": "Zip",
    "Architectures": [
        "x86_64"
    ],
    "EphemeralStorage": {
        "Size": 512
    },
    "SnapStart": {
        "ApplyOn": "None",
        "OptimizationStatus": "Off"
    },
    "RuntimeVersionConfig": {
        "RuntimeVersionArn": "arn:aws:lambda:eu-west-2::runtime:dbfa6aec8278470c1512458be8c7a99b2d63682d2e2d1e8d276dbf05b7f99755"
    },
    "LoggingConfig": {
        "LogFormat": "Text",
        "LogGroup": "/aws/lambda/cleanupSnapshots"
    }
}
```

and invoke 

```console
`~ $ aws lambda invoke \
>   --function-name cleanupSnapshots \
>   --payload '{}' \
>   response.json && cat response.json
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
 
}
```