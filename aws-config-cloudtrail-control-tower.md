# AWS Config and CloudTrail Integration - With Control Tower

For AWS Organizations using Control Tower, deploy via the Control Tower CloudFormation template. This enrolls all existing and future accounts automatically.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/399671/aws-control-tower-integration-using-cloudformation" target="_blank">AWS Control Tower Integration Using CloudFormation</a>

Reference: <a href="https://github.com/andrewbearsley/forticnapp-cloud-integration" target="_blank">forticnapp-cloud-integration</a>

## Steps

1. Create a FortiCNAPP API key: in the FortiCNAPP Console, navigate to Settings > API Keys > Add New. Download the API key JSON file.

2. Log into the AWS Control Tower management account in the region where Control Tower is deployed.

3. Gather Control Tower details:
   - Log Archive Account ID (eg 123456789012)
   - Log Archive Account Name (eg aws-controltower-LogArchiveAccount)
   - Audit Account ID (eg 123456789012)
   - Audit Account Name (eg aws-controltower-AuditAccount)
   - CloudTrail Name (eg aws-controltower-BaselineCloudTrail)

4. Check if the CloudTrail S3 bucket (in the Log Archive Account) has KMS encryption enabled.
   - If enabled, note the KMS Key Identifier ARN (eg arn:aws:kms:us-west-2:123456789012:key/12345678-1234-1234-1234-123456789012)

5. Check if the CloudTrail S3 bucket has SNS notifications enabled.
   - If enabled, note the SNS Topic ARN
   - If not enabled, create a new SNS topic and note the ARN

6. Launch the CloudFormation stack:

   https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?templateURL=https://lacework-alliances.s3.us-west-2.amazonaws.com/lacework-control-tower-cfn/templates/control-tower-integration.template.yaml

   Parameters:
   - Stack Name (eg Lacework-Control-Tower-Integration)
   - FortiCNAPP URL (your account subdomain)
   - FortiCNAPP Access Key ID and Secret Key from your API key file
   - Capability Type: CloudTrail+Config (default)
   - Monitor Existing Accounts: Yes (default)
   - Existing AWS Control Tower CloudTrail Name
   - KMS Key Identifier ARN (if CloudTrail logs are encrypted)
   - Log Account Name and Audit Account Name (update if necessary)

7. Create the stack and wait for completion.

8. (Optional) Update the KMS Key Policy for cross-account role access. This is only required if CloudTrail S3 logs are KMS encrypted.

   Find the Lacework role ARN:

   Option A: Check CloudFormation stack outputs for the role ARN.

   Option B: Look up the role in IAM in the log archive account:

   ```bash
   aws iam list-roles --query "Roles[?contains(RoleName, 'laceworkcwssarole')].Arn" --output text
   ```

   Add this statement to the KMS key policy:

   ```json
   {
     "Sid": "Allow Lacework to decrypt logs",
     "Effect": "Allow",
     "Principal": {
       "AWS": [
         "arn:aws:iam::<log-archive-account-id>:role/<lacework-account-name>-laceworkcwssarole"
       ]
     },
     "Action": [
       "kms:Decrypt"
     ],
     "Resource": "*"
   }
   ```
