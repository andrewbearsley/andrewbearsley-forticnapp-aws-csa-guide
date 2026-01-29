# FortiCNAPP AWS Cloud Security Assessment

Deployment guide for running a FortiCNAPP Cloud Security Assessment on AWS using CloudFormation.

## Overview

A FortiCNAPP Cloud Security Assessment (CSA) provides visibility into your AWS environment through three capabilities:

- **Config and CloudTrail integration**: resource inventory, compliance posture, and audit log analysis
- **Agentless workload scanning**: vulnerability assessment of EC2 instances without installing agents
- **Agent installation (optional)**: runtime threat detection, file integrity monitoring, and host-level visibility

This guide covers integration via CloudFormation.

## Quickstart

1. Set up FortiCNAPP (account, license, instance): [Step 1](#setting-up-forticnapp)
2. Deploy Config and CloudTrail integration via CloudFormation
   - AWS Organizations with Control Tower: [Step 2a](aws-config-cloudtrail-control-tower.md)
   - AWS Organizations without Control Tower: [Step 2b](aws-config-cloudtrail-org.md)
   - Individual AWS accounts: [Step 2c](aws-config-cloudtrail-standalone.md)
3. Deploy Agentless Workload Scanning via CloudFormation: [Step 3](agentless-workload-scanning.md)
4. (Optional) Install agents on target instances: [Step 4](agent-installation.md)
5. Validate the deployment: [Validation](#validating-the-deployment)

## Prerequisites

- FortiCNAPP license
- AWS account access:
  - For Control Tower: management account access in the Control Tower region
  - For Organizations (no Control Tower): management account access
  - For single account: account with CloudFormation and IAM permissions

### Setting Up FortiCNAPP

1. Create a FortiCloud account (if you don't already have one):
   - Go to <a href="https://www.forticloud.com" target="_blank">forticloud.com</a> > Create Account

2. Register your FortiCNAPP license:
   - In FortiCloud, navigate to Dashboard > Register Now

3. Create FortiCNAPP instance (first time only):
   - Navigate to Services > Show More > Lacework FortiCNAPP
   - Choose a region (eg AS for Singapore, AU for Australia)
   - Wait a few minutes for the instance to be provisioned

4. (Optional) Add additional users: see [Adding FortiCNAPP Users](adding-forticnapp-users.md)

### Logging into FortiCNAPP

1. Log in to <a href="https://forticloud.com" target="_blank">FortiCloud</a>
2. Navigate to Services > Show More > Lacework FortiCNAPP

## Deployment Guides

| Step | Guide | Description |
|------|-------|-------------|
| 2a | [Config and CloudTrail - Control Tower](aws-config-cloudtrail-control-tower.md) | AWS Organizations with Control Tower |
| 2b | [Config and CloudTrail - Organizations](aws-config-cloudtrail-org.md) | AWS Organizations without Control Tower |
| 2c | [Config and CloudTrail - Single Account](aws-config-cloudtrail-standalone.md) | Single AWS account |
| 3 | [Agentless Workload Scanning](agentless-workload-scanning.md) | Vulnerability scanning of EC2 instances |
| 4 | [Agent Installation](agent-installation.md) | (Optional) Runtime threat detection |

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

## Appendix

- [Adding FortiCNAPP Users](adding-forticnapp-users.md)

## Resources

- <a href="https://docs.fortinet.com/product/forticnapp" target="_blank">FortiCNAPP Documentation</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide" target="_blank">FortiCNAPP Administration Guide</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/cli-reference" target="_blank">FortiCNAPP CLI Reference</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/399671/aws-control-tower-integration-using-cloudformation" target="_blank">AWS Control Tower Integration</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/177498/aws-integration-using-cloudformation" target="_blank">AWS Integration (CloudFormation)</a>
- <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">Agentless Workload Scanning Guide</a>
- <a href="https://github.com/andrewbearsley/forticnapp-cloud-integration" target="_blank">FortiCNAPP Cloud Integration Setup</a>
