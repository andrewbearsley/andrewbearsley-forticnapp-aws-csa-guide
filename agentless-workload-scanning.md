# Agentless Workload Scanning

Agentless Workload Scanning provides vulnerability assessment of EC2 instances by creating and analyzing EBS volume snapshots, without installing agents on target systems.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/122712/prerequisites" target="_blank">Agentless Workload Scanning for AWS</a>

Reference: <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">forticnapp-aws-agentless-workload-scanning-guide</a>

## CloudFormation Deployment

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
