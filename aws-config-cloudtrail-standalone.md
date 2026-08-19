# AWS Config and CloudTrail Integration - Without Control Tower

For standalone AWS accounts or organizations not using Control Tower, deploy using the standard CloudFormation integration.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/821177/cloudformation-configuration" target="_blank">CloudFormation configuration</a>

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

8. Update the KMS Key Policy. Required if the CloudTrail logs are KMS encrypted and the key policy does not already delegate to IAM.

   AWS needs two separate permissions here, and both must allow:

   - The KMS key ARN you supplied during setup grants the **identity side**, on the FortiCNAPP role.
   - The key policy grants the **resource side**, on the key itself.

   Check the key policy before you change anything:

   ```bash
   KEY_ARN="arn:aws:kms:<region>:<account-id>:key/<key-id>"
   aws kms get-key-policy --key-id "$KEY_ARN" --policy-name default \
     --query Policy --output text > key-policy-backup.json
   jq '.Statement[] | select(.Principal.AWS | tostring | endswith(":root"))' key-policy-backup.json
   ```

   If that returns the default `Enable IAM User Permissions` statement, the identity side alone is enough and you can skip the rest of this step. That shortcut applies only when the key and the role sit in the same account.

   Add the statement when the key policy has been locked down, or when the key lives in a different account to the role. Account-root delegation does not cross account boundaries.

   Find the role ARN in the CloudFormation stack outputs, or look it up in IAM:

   ```bash
   aws iam list-roles \
     --query "Roles[?contains(RoleName, 'laceworkcwssarole')].Arn" --output text
   ```

   Discover the name rather than assuming it. The suffix varies between deployments.

   Statement to add:

   ```json
   {
     "Sid": "AllowLaceworkDecryptCloudTrailLogs",
     "Effect": "Allow",
     "Principal": {
       "AWS": "arn:aws:iam::<account-id>:role/<role-name>"
     },
     "Action": "kms:Decrypt",
     "Resource": "*"
   }
   ```

   `"Resource": "*"` is correct. In a key policy the resource is the key that carries the policy.

   Apply it from the command line:

   ```bash
   ROLE_ARN="arn:aws:iam::<account-id>:role/<role-name>"

   jq --arg role "$ROLE_ARN" '.Statement += [{
     "Sid": "AllowLaceworkDecryptCloudTrailLogs",
     "Effect": "Allow",
     "Principal": {"AWS": $role},
     "Action": "kms:Decrypt",
     "Resource": "*"
   }]' key-policy-backup.json > key-policy-new.json

   jq -r '.Statement[].Sid' key-policy-new.json

   aws kms put-key-policy --key-id "$KEY_ARN" --policy-name default \
     --policy file://key-policy-new.json
   ```

   `put-key-policy` replaces the whole policy. It does not append. Always merge into the existing document. Writing the statement on its own removes the root statement and locks the key.

9. Verify. Open the CloudTrail dashboard in the FortiCNAPP console and check the API Error Information table. "Access Denied" decryption errors mean the key policy is still missing the statement. They stop once it applies.
