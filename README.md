# Centralized Alarms Solution for AWS Backup

The solution builds an alarm system to notify via email when a backup job or backup copy has failed and when a backup policy has been modified.

## Description

The solution leverages Amazon EventBridge event rules to capture the source event and forward the event to an Amazon EventBridge event bus in a Central Backup account. Then, Amazon EventBridge event bus sends the event to an Amazon SNS topic and it sends an email with the content of the event to the email subscribed.

Since the events are managed in a Central Backup account through an event bus, the solution can be easily customised to invoke a different target such as: Lambda function, SQS queue, API endpoint, etc. and integrate with an external alarm system.

### Backup and Copy Jobs

Backups performed in the workload accounts are monitored to be notified when a backup job or copy job fails. The diagram below shows the solution implemented:

![Backup Jobs Architecture](/images/backup-jobs.png)

When a backup or copy job fails AWS Backup emits an event that is captured by an Amazon EventBridge event rule. The event rule sends the event to the Amazon EventBridge event bus in the Central Backup account. Then, another event is invoked and sends the event to an Amazon SNS topic. Lastly, the notification is received by the email subscribed to the SNS topic.

A sample event of the backup job failed can be found below:

```
{
  "version": "0",
  "id": "710b0398-d48e-f3c3-afca-cfeb2fdaa656",
  "detail-type": "Backup Job State Change",
  "source": "aws.backup",
  "account": "1112233445566",
  "time": "2020-07-29T20:15:26Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "backupJobId": "34176239-e96d-4e1d-9fad-529dbb3c3556",
    "backupVaultArn": "arn:aws:backup:us-west-2:1112233445566:backup-vault:9ab3e749-82c6-4342-9320-5edbf4918b86_beta",
    "backupVaultName": "9ab3e749-82c6-4342-9320-5edbf4918b86_beta",
    "bytesTransferred": "0",
    "creationDate": "2020-07-29T20:13:07.392Z",
    "iamRoleArn": "arn:aws:iam::1112233445566:role/MockRCBackupIntegTestRole",
    "resourceArn": "arn:aws:cryo-mock:us-west-2:1112233445566:resource:dummy-fs-1",
    "resourceType": "CryoTestClient",
    "state": "FAILED",
    "statusMessage": "\"Backup job failed because backup vault arn:aws:backup:us-west-2:1112233445566:backup-vault:9ab3e749-82c6-4342-9320-5edbf4918b86_beta does not exist.\"",
    "startBy": "2020-07-30T04:13:07.392Z",
    "percentDone": 0
  }
}
```

A sample event of the backup copy job failed can be found below:

```
{
  "version": "0",
  "id": "4660bc92-a44d-c939-4542-cda503f14855",
  "detail-type": "Copy Job State Change",
  "source": "aws.backup",
  "account": "1112233445566",
  "time": "2020-07-15T20:37:34Z",
  "region": "us-east-1",
  "resources": [
    "arn:aws:ec2:us-west-2::image/ami-00179b33a7a88cac5"
  ],
  "detail": {
    "copyJobId": "47C8EF56-74D8-059D-1301-C5BE1D5C926E",
    "backupSizeInBytes": 22548578304,
    "creationDate": "2020-07-15T20:36:13.239Z",
    "iamRoleArn": "arn:aws:iam::1112233445566:role/RoleForEc2BackupWithNoDescribeTagsPermissions",
    "resourceArn": "arn:aws:ec2:us-west-2:1112233445566:instance/i-0515aee7de03f58e1",
    "resourceType": "EC2",
    "sourceBackupVaultArn": "arn:aws:backup:us-west-2:1112233445566:backup-vault:55aa945e-c46a-421b-aa27-f94b074e31b7_beta",
    "state": "FAILED",
    "statusMessage": "Access denied exception while trying to list tags",
    "completionDate": "2020-07-15T20:37:28.704Z",
    "destinationBackupVaultArn": "arn:aws:backup:us-west-2:1112233445566:backup-vault:55aa945e-c46a-421b-aa27-f94b074e31b7_beta",
    "destinationRecoveryPointArn": {}
  }
}
```

### Backup Policies

Backup policies managed in the management account are monitored to track unintended modifications. The diagram below shows the solution implemented:

![Backup Policies Architecture](/images/backup-policies.png)

When a backup policy is modified or deleted, the API call invokes an Amazon EventBridge event rule. The event rule sends the event to the Amazon EventBridge event bus in the Central Backup account. Then, another event is invoked and sends the event to an Amazon SNS topic. Lastly, the notification is received by the email subscribed to the SNS topic.

A sample event of the backup policy can be found below:

```
{
  "version": "0",
  "id": "41bfe6d9-c5ac-f6a1-e235-07c48fdb1e65",
  "detail-type": "AWS API Call via CloudTrail",
  "source": "aws.organizations",
  "account": "1112233445566",
  "time": "2022-01-06T00:01:57Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "eventVersion": "1.08",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROAIDPPEZS35WEXAMPLE:mendiof",
      "arn": "arn:aws:sts::1112233445566:assumed-role/admin/mendiof",
      "accountId": "1112233445566",
      "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "principalId": "AROAIDPPEZS35WEXAMPLE",
          "arn": "arn:aws:iam::1112233445566:role/admin",
          "accountId": "1112233445566",
          "userName": "admin"
        },
        "webIdFederationData": {},
        "attributes": {
          "creationDate": "2022-01-06T00:01:23Z",
          "mfaAuthenticated": "false"
        }
      }
    },
    "eventTime": "2022-01-06T00:01:43Z",
    "eventSource": "organizations.amazonaws.com",
    "eventName": "UpdatePolicy",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "1.2.3.4",
    "userAgent": "aws-internal/3 aws-sdk-java/1.12.125 Linux/4.9.273-0.1.\nac.226.84.332.metal1.x86_64 OpenJDK_64-Bit_Server_VM/25.312-b07 java/1.8.0_312",
    "requestParameters": {
      "content": "{\"plans\":{\"Daily\":{\"regions\":{\"@@assign\":[\"eu-west-1\"]},\"rules\":{\"Daily\":{\"lifecycle\":{\"delete_after_days\":{\"@@assign\":\"2\"}},\"target_backup_vault_name\":{\"@@assign\":\"workload\"}}},\"selections\":{\"tags\":{\"Resources\":{\"iam_role_arn\":{\"@@assign\":\"arn:aws:iam::$account:role/AWSBackupRole\"},\"tag_key\":{\"@@assign\":\"backup\"},\"tag_value\":{\"@@assign\":[\"tier2\"]}}}}}}}",
      "policyId": "p-89nw1aav4i",
      "description": "Daily",
      "name": "Daily"
    },
    "responseElements": {
      "policy": {
        "policySummary": {
          "id": "p-89nw1aav4i",
          "description": "Daily",
          "type": "BACKUP_POLICY",
          "awsManaged": false,
          "arn": "arn:aws:organizations::1112233445566:policy/o-7x5db8491h/backup_policy/p-89nw1aav4i",
          "name": "Daily"
        },
        "content": "{\"plans\":{\"Daily\":{\"regions\":{\"@@assign\":[\"eu-west-1\"]},\"rules\":{\"Daily\":{\"lifecycle\":{\"delete_aft\ner_days\":{\"@@assign\":\"2\"}},\"target_backup_vault_name\":{\"@@assign\":\"workload\"}}},\"selections\":{\"tags\":{\"Resources\":{\"iam_role_arn\":{\"@@assign\":\"arn:aws:iam::$account:role/AWSBackupRole\"},\"tag_key\":{\"@@assign\":\"backup\"},\"tag_value\":{\"@@assign\":[\"tier2\"]}}}}}}}"
      }
    },
    "requestID": "52a01cae-eec6-4119-8c50-57ab25c0a632",
    "eventID": "fb649f58-98eb-4d83-baed-35252756192f",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "1112233445566",
    "eventCategory": "Management"
  }
}
```

## Deployment Steps

The solution is deployed using the AWS CloudFormation templates provided within the cfn-templates folder. The management-account folder contains the assets for the Management account and the delegated-account folder contains the assets for the CloudFormation delegated administrator account.

Best practices recommend to use a CloudFormation delegated administrator account to deploy AWS CloudFormation StackSets. In case you do not have registered a delegated administrator account, the link below describes the procedure.

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-delegated-admin.html

### Delegated Administrator Account

The template main.yaml located within the delegated-account folder must be deployed in the CloudFormation delegated administrator account.

**Step 1**. Replace the parameters located in cfn-templates/delegated-account/parameters.json with your values. The following table describes the parameters used by this template.

| Parameter Name              | Parameter Description                                             |
| --------------------------- | ----------------------------------------------------------------- |
| pAWSBackupRegion            | The region where the AWS Backup resources are located             |
| pS3BucketNameArtifacts      | The Amazon S3 Bucket name where the artifacts are stored          |
| pS3BucketKeyPrefixArtifacts | The Amazon S3 key prefix where the artifacts are stored           |
| pWorkloadOUs                | A comma separated list of workload organizational units           |
| pCentralBackupOU            | The organizational unit of the Central Backup account             |
| pAWSOrganizationsID         | The AWS Organizations ID                                          |
| pCentralBackupAccountID     | The Central Backup account ID to create the SNS topic             |
| pEventBusName               | The EventBridge event bus name in the Central Backup account      |
| pSNSTopicName               | The SNS Topic name to receive events and send email notifications |
| PSNSSubscriptionEmail       | The SNS subscription email to receive the alarm notifications     |

**Step 2**. Create an S3 bucket in the Delegated Administrator account and upload the folder cfn-templates/delegated-account to the S3 bucket.

**Step 3**. In the Delegated Administrator account deploy the CloudFormation stack through the CLI as follows:

```
aws cloudformation create-stack --stack-name <STACK_NAME> \
--template-url https://<BUCKET_NAME>.s3.<AWS_REGION>.amazonaws.com/cfn-templates/delegated-account/main.yaml \
--parameters file://cfn_templates/delegated-account/parameters.json \
--capabilities CAPABILITY_NAMED_IAM \
--region <AWS_REGION>
```

### Management Account

The template main.yaml located within the management folder  must be deployed in the management account in the us-east-1 region. This is due to CloudTrail events for AWS Organizations are only available in the us-east-1 region.

**Step 1**. Replace the parameters located in cfn-templates/management-account/parameters.json with your values. The following table describes the parameters used by this template.

| Parameter Name          | Parameter Description                                          |
| ----------------------- | -------------------------------------------------------------- |
| pAWSBackupRegion        | The region where the AWS Backup resources are located          |
| pCentralBackupAccountID | The Central Backup account ID where the SNS topic is created   |
| PSNSSubscriptionEmail   | The SNS subscription email to receive the alarm notifications  |

**Step 2**. In the AWS Organizations management account, deploy the CloudFormation stack through the CLI as follows:

```
aws cloudformation create-stack --stack-name <STACK_NAME> \
--template-body file://cfn-templates/management-account/main.yaml \
--parameters file://cfn-templates/management-account/parameters.json \
--capabilities CAPABILITY_NAMED_IAM \
--region <AWS_REGION>
```

### Test the Solution

After the solution has been deployed you will receive email notifications to the email subscribed when a backup job or backup copy has failed and when a backup policy has been modified.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
