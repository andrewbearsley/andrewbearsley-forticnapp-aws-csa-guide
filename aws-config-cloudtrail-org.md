# AWS Config and CloudTrail Integration - AWS Organizations (without Control Tower)

For AWS Organizations not using Control Tower, deploy via the Organization CloudFormation template. This enables automatic onboarding and offboarding of AWS accounts within your FortiCNAPP tenant.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration Using CloudFormation</a>

Reference: <a href="https://github.com/lacework-alliances/aws-org-cf-lacework" target="_blank">lacework-alliances/aws-org-cf-lacework</a>

## Features

- Automatic account provisioning and deprovisioning
- Event-driven notifications when account changes occur
- Role and permission management for new/updated/deleted accounts

## Steps

1. Create a FortiCNAPP API key: in the FortiCNAPP Console, navigate to Settings > API Keys > Add New. Download the API key JSON file.

2. Log into the AWS Organizations management account.

3. Launch the CloudFormation stack:

   <a href="https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?templateURL=https://lacework-alliances.s3.us-west-2.amazonaws.com/lacework-organization-cfn/templates/lacework-aws-cfg-manage.template.yml" target="_blank">Launch Stack</a>

   Or use the template URL directly:
   ```
   https://lacework-alliances.s3.us-west-2.amazonaws.com/lacework-organization-cfn/templates/lacework-aws-cfg-manage.template.yml
   ```

4. Enter the required parameters:
   - FortiCNAPP Account Name (your account subdomain)
   - FortiCNAPP Access Key ID and Secret Key from your API key file

5. Create the stack and wait for completion.

6. Return to the FortiCNAPP Console to verify the integration status.
