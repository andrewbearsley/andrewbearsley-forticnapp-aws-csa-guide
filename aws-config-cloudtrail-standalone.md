# AWS Config and CloudTrail Integration - Without Control Tower

For standalone AWS accounts or organizations not using Control Tower, deploy using the standard CloudFormation integration.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration Using CloudFormation</a>

## Prerequisites (for integrating with an existing CloudTrail)

If you plan to use CloudTrail+Configuration with an existing trail, gather these details first:

1. In the AWS Console, navigate to CloudTrail > Trails
2. Select your trail and note:
   - S3 bucket name (under Storage location)
   - SNS topic ARN (under SNS notification delivery)
   - KMS key ARN (under Log file SSE-KMS encryption, if enabled)
3. Enable SNS notifications (if not already enabled):
   - Select your trail, click Edit
   - Scroll to SNS notification delivery
   - Check "Enabled", and select "New"
   - Save changes
   - Note the SNS topic ARN for use during FortiCNAPP setup

## Steps

1. Log into the AWS account where you want to deploy the integration.

2. In the FortiCNAPP Console, navigate to Settings > Integrations > Cloud Accounts > Add New.

3. Select Amazon Web Services, then select CloudFormation as the deployment method.

4. Choose integration type:
   - CloudTrail+Configuration: for accounts with CloudTrail enabled
   - Configuration: for accounts without CloudTrail

5. Follow the guided setup to launch the CloudFormation stack. The console provides a pre-configured template URL with your account parameters.

   For existing CloudTrail integration:
   - Create new trail?: No
   - Existing Trail Setup:
     - Bucket name: enter the S3 bucket name from prerequisites
     - Topic ARN: enter the SNS topic ARN from prerequisites
   - Leave other settings as default

6. Create the stack and wait for completion.

7. Return to the FortiCNAPP Console to verify the integration status.

8. (Optional) Update the KMS Key Policy for cross-account role access. This is only required if CloudTrail logs are KMS encrypted.

   Find the Lacework role ARN in the CloudFormation stack outputs, or look it up in IAM:

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
         "arn:aws:iam::<account-id>:role/<lacework-account-name>-laceworkcwssarole"
       ]
     },
     "Action": [
       "kms:Decrypt"
     ],
     "Resource": "*"
   }
   ```
