# AWS Config and CloudTrail Integration - With Control Tower

For AWS Organizations using Control Tower, deploy via the Control Tower CloudFormation template. This enrolls all existing and future accounts automatically.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/821177/cloudformation-configuration" target="_blank">CloudFormation configuration</a>

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
   - Note the account ID in that ARN. Control Tower normally creates this key in the management account, not the Log Archive account. Step 8 depends on it.

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

8. Update the KMS Key Policy for cross-account role access. Required if the CloudTrail S3 logs are KMS encrypted.

   AWS needs two separate permissions here, and both must allow. Supplying the ARN at step 6 does not remove the need for this step.

   **Identity side. An IAM policy, attached to the role.** The CloudFormation stack creates this for you from the KMS Key Identifier ARN at step 6. It sits in the inline `LaceworkCWSPolicy` on the `laceworkcwssarole` role:

   ```json
   {
     "Sid": "DecryptLogFiles",
     "Effect": "Allow",
     "Action": [ "kms:Decrypt" ],
     "Resource": [ "arn:aws:kms:<region>:<key-owner-account-id>:key/<key-id>" ]
   }
   ```

   **Resource side. A key policy, attached to the key.** You add this yourself, in step 8:

   ```json
   {
     "Sid": "AllowLaceworkDecryptCloudTrailLogs",
     "Effect": "Allow",
     "Principal": {
       "AWS": "arn:aws:iam::<log-archive-account-id>:role/<role-name>"
     },
     "Action": "kms:Decrypt",
     "Resource": "*"
   }
   ```

   Telling them apart:

   | | Identity policy | Key policy |
   |---|---|---|
   | Attached to | the role | the key |
   | `Principal` | absent, the role is implied | required, names the role |
   | `Resource` | the key ARN | `*`, meaning this key |
   | Answers | what may this role do | who may use this key |
   | Created by | the CloudFormation stack | you |

   A request succeeds only when both allow it.

   Under Control Tower the CloudTrail KMS key normally lives in the management account, while the `laceworkcwssarole` role is created in the Log Archive account. That makes this a cross-account grant, so the key policy must name the role. Account-root delegation does not cross account boundaries.

   Find the role ARN. Run this in the Log Archive account:

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
       "AWS": "arn:aws:iam::<log-archive-account-id>:role/<role-name>"
     },
     "Action": "kms:Decrypt",
     "Resource": "*"
   }
   ```

   `"Resource": "*"` is correct. In a key policy the resource is the key that carries the policy.

   **Add the statement to the existing key policy. Do not replace the policy.**

   The key policy already carries statements that let CloudTrail encrypt logs and let the account manage the key. Removing any of them breaks log delivery across the landing zone.

   Take a backup first. Run this in the account that owns the key:

   ```bash
   KEY_ARN="arn:aws:kms:<region>:<key-owner-account-id>:key/<key-id>"

   aws kms get-key-policy --key-id "$KEY_ARN" --policy-name default \
     --query Policy --output text > key-policy-backup.json
   ```

   Then edit the policy JSON:

   - Open KMS in the account that owns the key
   - Customer managed keys, select the key, Key policy tab
   - Choose Edit. Switch to policy view if the console opens the visual editor
   - Add the statement to the existing `Statement` array
   - Save

   Your key already has statements. Keep every one of them and append the new statement to the end of the array. Note the comma after the preceding statement, which is the most common mistake.

   A Control Tower CloudTrail key looks roughly like this once the statement is added. Your existing statements will differ, so treat the first two as illustrative and copy only the last one:

   ```json
   {
     "Version": "2012-10-17",
     "Id": "key-consolepolicy-3",
     "Statement": [
       {
         "Sid": "Enable IAM User Permissions",
         "Effect": "Allow",
         "Principal": { "AWS": "arn:aws:iam::<key-owner-account-id>:root" },
         "Action": "kms:*",
         "Resource": "*"
       },
       {
         "Sid": "Allow CloudTrail to encrypt logs",
         "Effect": "Allow",
         "Principal": { "Service": "cloudtrail.amazonaws.com" },
         "Action": "kms:GenerateDataKey*",
         "Resource": "*"
       },
       {
         "Sid": "AllowLaceworkDecryptCloudTrailLogs",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::<log-archive-account-id>:role/<role-name>"
         },
         "Action": "kms:Decrypt",
         "Resource": "*"
       }
     ]
   }
   ```

   Restore from `key-policy-backup.json` if anything goes wrong.

   <details>
   <summary>Applying from the CLI instead</summary>

   `put-key-policy` replaces the entire policy. It does not append. Only use this if you are comfortable with that.

   ```bash
   ROLE_ARN="arn:aws:iam::<log-archive-account-id>:role/<role-name>"

   # Normalise Statement to an array, then append
   jq --arg role "$ROLE_ARN" '
     .Statement = ((.Statement | if type == "array" then . else [.] end) + [{
       "Sid": "AllowLaceworkDecryptCloudTrailLogs",
       "Effect": "Allow",
       "Principal": {"AWS": $role},
       "Action": "kms:Decrypt",
       "Resource": "*"
     }])' key-policy-backup.json > key-policy-new.json

   # Check before you apply: the new file must keep every original Sid and add one
   diff <(jq -r '[.Statement]|flatten|.[].Sid // "NO_SID"' key-policy-backup.json | sort) \
        <(jq -r '[.Statement]|flatten|.[].Sid // "NO_SID"' key-policy-new.json | sort)

   aws kms put-key-policy --key-id "$KEY_ARN" --policy-name default \
     --policy file://key-policy-new.json
   ```

   The `diff` must show exactly one added line. Anything else means stop.

   AWS runs a lockout safety check that rejects a policy leaving the key unmanageable, so the worst case is blocked. That check does not protect the CloudTrail statements, so a bad merge still breaks log delivery.

   Roll back with:

   ```bash
   aws kms put-key-policy --key-id "$KEY_ARN" --policy-name default \
     --policy file://key-policy-backup.json
   ```

   </details>

9. Verify. Open the CloudTrail dashboard in the FortiCNAPP console and check the API Error Information table. "Access Denied" decryption errors mean the key policy is still missing the statement. They stop once it applies.
