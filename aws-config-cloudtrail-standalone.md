# AWS Config and CloudTrail Integration - Without Control Tower

For standalone AWS accounts or organizations not using Control Tower, deploy using the standard CloudFormation integration.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration Using CloudFormation</a>

## Prerequisites (for existing CloudTrail integration)

If you plan to use CloudTrail+Configuration with an existing trail, gather these details first:

1. In the AWS Console, navigate to CloudTrail > Trails
2. Select your trail and note:
   - S3 bucket name (under Storage location)
   - SNS topic ARN (under SNS notification delivery)

### Enabling SNS notifications (if not already enabled)

1. In the AWS Console, navigate to CloudTrail > Trails
2. Select your trail, click Edit
3. Scroll to SNS notification delivery
4. Check "Enabled"
5. Either select an existing SNS topic or create a new one
6. Save changes
7. Note the SNS topic ARN for use during FortiCNAPP setup

## Steps

1. Log into the AWS account where you want to deploy the integration.

2. In the FortiCNAPP Console, navigate to Settings > Integrations > Cloud Accounts > Add New.

3. Select Amazon Web Services, then select CloudFormation as the deployment method.

4. Choose integration type:
   - CloudTrail+Configuration: for accounts with CloudTrail enabled
   - Configuration: for accounts without CloudTrail

5. Follow the guided setup to launch the CloudFormation stack. The console provides a pre-configured template URL with your account parameters.

6. Create the stack and wait for completion.

7. Return to the FortiCNAPP Console to verify the integration status.
