# FortiCNAPP AWS Cloud Security Assessment

Deployment guide for running a FortiCNAPP Cloud Security Assessment on AWS using CloudFormation.

## Overview

A FortiCNAPP Cloud Security Assessment (CSA) provides visibility into your AWS environment through three capabilities:

- Config and CloudTrail integration: resource inventory, compliance posture, and audit log analysis
- Agentless workload scanning: vulnerability assessment of EC2 instances without installing agents
- Agent installation (optional): runtime threat detection, file integrity monitoring, and host-level visibility

This guide covers deploying all three via CloudFormation.

## Quickstart

1. Deploy Config and CloudTrail integration via CloudFormation
   - With Control Tower: [Step 1a](#with-aws-control-tower)
   - Without Control Tower: [Step 1b](#without-aws-control-tower)
2. Deploy Agentless Workload Scanning via CloudFormation: [Step 2](#step-2-agentless-workload-scanning)
3. (Optional) Install agents on target instances: [Step 3](#step-3-optional-agent-installation)
4. Validate the deployment: [Validation](#validating-the-deployment)

## Prerequisites

- Access to FortiCNAPP Console
- AWS account access:
  - For Control Tower: management account access in the Control Tower region
  - For standalone: account with CloudFormation and IAM permissions

### Logging into FortiCNAPP

1. Log in to <a href="https://forticloud.com" target="_blank">FortiCloud</a>
2. Navigate to Services > Show More > Lacework FortiCNAPP

## Step 1: AWS Config and CloudTrail Integration

This integration gives FortiCNAPP access to AWS resource inventory (Config) and audit logs (CloudTrail) for compliance assessment and anomaly detection.

### With AWS Control Tower

For AWS Organizations using Control Tower, deploy via the Control Tower CloudFormation template. This enrolls all existing and future accounts automatically.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/399671/aws-control-tower-integration-using-cloudformation" target="_blank">AWS Control Tower Integration Using CloudFormation</a>

Reference: <a href="https://github.com/andrewbearsley/forticnapp-cloud-integration" target="_blank">forticnapp-cloud-integration</a>

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

### Without AWS Control Tower

For standalone AWS accounts or organizations not using Control Tower, deploy using the standard CloudFormation integration.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration Using CloudFormation</a>

1. Log into the AWS account where you want to deploy the integration.

2. In the FortiCNAPP Console, navigate to Settings > Integrations > Cloud Accounts > Add New.

3. Select AWS, then select CloudFormation as the deployment method.

4. Follow the guided setup to launch the CloudFormation stack. The console provides a pre-configured template URL with your account parameters.

5. Create the stack and wait for completion.

6. Return to the FortiCNAPP Console to verify the integration status.

## Step 2: Agentless Workload Scanning

Agentless Workload Scanning provides vulnerability assessment of EC2 instances by creating and analyzing EBS volume snapshots, without installing agents on target systems.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/122712/prerequisites" target="_blank">Agentless Workload Scanning for AWS</a>

Reference: <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">forticnapp-aws-agentless-workload-scanning-guide</a>

### CloudFormation Deployment

1. In the FortiCNAPP Console, navigate to Settings > Integrations > Cloud Accounts.

2. Select your AWS integration, then enable Agentless Workload Scanning.

3. Choose CloudFormation as the deployment method.

4. Follow the guided setup to launch the CloudFormation stack. The template deploys:
   - ECS Fargate cluster with scheduled tasks
   - S3 bucket for scan results and metadata
   - VPC networking (or uses an existing VPC)
   - IAM roles for cross-account access and snapshot creation
   - CloudWatch event rules and log groups

5. Create the stack and wait for completion.

6. Return to the FortiCNAPP Console to verify the scanning status.

See the <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">agentless workload scanning guide</a> for detailed architecture and deployment options including Terraform.

## Step 3 (Optional): Agent Installation

The FortiCNAPP agent provides runtime threat detection, file integrity monitoring, and deeper host-level visibility beyond what agentless scanning covers.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/cli-reference/784655/lacework-agent-aws-install" target="_blank">Lacework Agent AWS Install</a>

The Lacework CLI supports three installation methods for AWS EC2 instances:

EC2 Instance Connect:

```bash
lacework agent aws-install ec2ic --instance-id i-1234567890abcdef0 --region us-east-1
```

EC2 SSH:

```bash
lacework agent aws-install ec2ssh --instance-id i-1234567890abcdef0 --region us-east-1 --ssh-user ec2-user
```

EC2 Systems Manager:

```bash
lacework agent aws-install ec2ssm --instance-id i-1234567890abcdef0 --region us-east-1
```

Install the Lacework CLI first if not already installed:

```bash
brew install lacework/tap/lacework-cli
```

Or:

```bash
curl https://raw.githubusercontent.com/lacework/go-sdk/main/cli/install.sh | sudo bash
```

## How It Works

### Config Integration

FortiCNAPP reads AWS Config data to build a resource inventory across your accounts. This enables:

- Compliance assessments against CIS, SOC 2, PCI-DSS, HIPAA and other frameworks
- Resource configuration auditing
- Misconfiguration detection
- Multi-account visibility (with Control Tower, all accounts are enrolled automatically)

### CloudTrail Integration

FortiCNAPP ingests CloudTrail audit logs to provide:

- API activity monitoring and anomaly detection
- User behavior analytics
- Threat detection based on unusual API patterns
- Investigation timelines for security incidents

### Agentless Workload Scanning

Scans EC2 instances for vulnerabilities without installing agents:

1. CloudWatch event rules trigger ECS tasks on a schedule
2. ECS tasks create snapshots of EC2 instance EBS volumes
3. Snapshots are analyzed for OS and application vulnerabilities
4. Snapshots are deleted after scanning
5. Scan results are stored in S3 and retrieved by FortiCNAPP

### Agent (Optional)

The FortiCNAPP agent runs on EC2 instances and provides:

- Runtime threat detection and response
- File integrity monitoring (FIM)
- Process-level visibility
- Network connection monitoring
- Host intrusion detection

## Validating the Deployment

### CLI Validation

```bash
lacework cloud-account list
```

This lists all configured integrations and their status.

### Console Validation

1. Navigate to Settings > Integrations > Cloud Accounts
2. Verify each integration shows status: Enabled and state: Ok
3. For agentless scanning, check that the last successful scan time is recent

### Expected Integrations

After completing all steps, you should see:

- Config integration (AwsCfg or AwsCfgOrg)
- CloudTrail integration (AwsCtSqs or similar)
- Agentless workload scanning (AwsSidekick or AwsSidekickOrg)
- Agent entries (if agents were installed)

## Resources

- <a href="https://docs.fortinet.com/product/forticnapp" target="_blank">FortiCNAPP Documentation</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide" target="_blank">FortiCNAPP Administration Guide</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/cli-reference" target="_blank">FortiCNAPP CLI Reference</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/399671/aws-control-tower-integration-using-cloudformation" target="_blank">AWS Control Tower Integration</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration (CloudFormation)</a>
- <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">Agentless Workload Scanning Guide</a>
- <a href="https://github.com/andrewbearsley/forticnapp-cloud-integration" target="_blank">FortiCNAPP Cloud Integration Setup</a>

## Appendix

### Adding FortiCNAPP Users

To grant other users access to FortiCNAPP, configure permissions in FortiCloud IAM.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/720640/iam-users" target="_blank">IAM Users</a>

1. In <a href="https://forticloud.com" target="_blank">FortiCloud</a>, navigate to Services > IAM.

2. Create a Permission Profile (skip if one already exists):
   - Navigate to Permission Profiles > Add New
   - Enter a Permission Profile Name (eg FortiCNAPP)
   - Click Add Portal, select Lacework FortiCNAPP
   - Toggle on Access
   - Save the profile

3. Add the user:
   - Navigate to Users > Add New > IAM User
   - Enter user details (name, email, etc.)
   - Click Next
   - Choose user type (eg Organisation or Local)
   - Select the Permission Profile created in step 2
   - Click Next
   - Review and Confirm

4. Assign FortiCNAPP user group:
   - In FortiCNAPP, navigate to Settings > Users
   - Find the user, click ... > Edit
   - Select user group: Admin
   - Save
