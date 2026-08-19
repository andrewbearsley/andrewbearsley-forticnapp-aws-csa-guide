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

   AWS needs two separate permissions here, and both must allow.

   **Identity side. An IAM policy, attached to the role.** The CloudFormation stack creates this for you from the KMS key ARN you supplied during setup. It sits in the inline `LaceworkCWSPolicy` on the `laceworkcwssarole` role:

   ```json
   {
     "Sid": "DecryptLogFiles",
     "Effect": "Allow",
     "Action": [ "kms:Decrypt" ],
     "Resource": [ "arn:aws:kms:<region>:<account-id>:key/<key-id>" ]
   }
   ```

   **Resource side. A key policy, attached to the key.** This is the one you may need to add:

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

   Both placeholders name the role the stack created. See **Finding the two values** below.

   Telling them apart:

   | | Identity policy | Key policy |
   |---|---|---|
   | Attached to | the role | the key |
   | `Principal` | absent, the role is implied | required, names the role |
   | `Resource` | the key ARN | `*`, meaning this key |
   | Answers | what may this role do | who may use this key |
   | Created by | the CloudFormation stack | you |

   A request succeeds only when both allow it.

   **Why both.** Most AWS services use a union model, where an identity policy or a resource policy is enough within one account. KMS inverts that. Every key has exactly one key policy, it cannot be deleted, and nothing may use the key unless that policy permits it. This keeps the key owner in the decision, so nobody can grant themselves decryption rights by editing IAM alone.

   The default key policy carries a statement that looks like this:

   ```json
   {
     "Sid": "Enable IAM User Permissions",
     "Effect": "Allow",
     "Principal": { "AWS": "arn:aws:iam::<account-id>:root" },
     "Action": "kms:*",
     "Resource": "*"
   }
   ```

   It does not mean the root user can do anything. It means the key delegates its access decisions to IAM in that one account. That delegation is the only reason KMS usually feels like every other service, and it does not reach across an account boundary.

   The check below tests whether your key still has it.

   Check the key policy before you change anything. This also gives you a backup:

   ```bash
   KEY_ARN="arn:aws:kms:<region>:<account-id>:key/<key-id>"

   aws kms get-key-policy --key-id "$KEY_ARN" --policy-name default \
     --query Policy --output text > key-policy-backup.json

   jq '[.Statement] | flatten | .[]
       | select((.Principal.AWS? | tostring) | endswith(":root"))' key-policy-backup.json
   ```

   If that returns the default `Enable IAM User Permissions` statement, the identity side alone is enough and you can skip the rest of this step. That shortcut applies only when the key and the role sit in the same account.

   Add the statement when the key policy has been locked down, or when the key lives in a different account to the role. Account-root delegation does not cross account boundaries.

   **Finding the two values.**

   The role name is `<forticnapp-tenant-name>-laceworkcwssarole`, so the tenant name is a prefix, not a suffix. Read it rather than assuming it.

   *From IAM.*

   1. Open **IAM > Roles** in the account where the FortiCNAPP stack ran.
   2. Search `laceworkcwssarole` and open the role.
   3. Copy the ARN shown at the top of the Summary panel.

   The copied ARN is the complete `Principal.AWS` value. The account ID and the role name are both in it, so you do not need to assemble them yourself.

   *From the CLI.* This prints the same ARN:

   ```bash
   aws iam list-roles \
     --query "Roles[?contains(RoleName, 'laceworkcwssarole')].Arn" --output text
   ```

   Do not expect the role ARN on the CloudFormation stack outputs. The integration templates do not publish it.

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

   **Add the statement to the existing key policy. Do not replace the policy.**

   The key policy already carries statements that let CloudTrail encrypt logs and let the account manage the key. Removing any of them breaks log delivery.

   Edit the policy JSON:

   - Open KMS, Customer managed keys, select the key, Key policy tab
   - Choose Edit. Switch to policy view if the console opens the visual editor
   - Add the statement to the existing `Statement` array
   - Save

   Your key already has statements. Keep every one of them and append the new statement to the end of the array. Note the comma after the preceding statement, which is the most common mistake.

   The result looks roughly like this. Your existing statements will differ, so treat the first two as illustrative and copy only the last one:

   ```json
   {
     "Version": "2012-10-17",
     "Id": "key-consolepolicy-3",
     "Statement": [
       {
         "Sid": "Enable IAM User Permissions",
         "Effect": "Allow",
         "Principal": { "AWS": "arn:aws:iam::<account-id>:root" },
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
           "AWS": "arn:aws:iam::<account-id>:role/<role-name>"
         },
         "Action": "kms:Decrypt",
         "Resource": "*"
       }
     ]
   }
   ```

   The last statement is the one you add. **Finding the two values** above gives the account ID and role name.

   Restore from `key-policy-backup.json` if anything goes wrong.

   <details>
   <summary>Applying from the CLI instead</summary>

   `put-key-policy` replaces the entire policy. It does not append. Only use this if you are comfortable with that.

   ```bash
   ROLE_ARN="arn:aws:iam::<account-id>:role/<role-name>"

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
