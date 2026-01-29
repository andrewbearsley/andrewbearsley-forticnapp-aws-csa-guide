# Agentless Workload Scanning

This guide covers deploying FortiCNAPP AWS Agentless Workload Scanning using CloudFormation.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/966589/agentless-workload-scanning" target="_blank">Deploying Agentless Workload Scanning on AWS</a>

Reference: <a href="https://github.com/andrewbearsley/forticnapp-aws-agentless-workload-scanning-guide" target="_blank">forticnapp-aws-agentless-workload-scanning-guide</a>

## Prerequisites

- Access to FortiCNAPP UI
- Permissions for: CloudFormation, IAM, ECS, VPC, S3, CloudWatch, Organizations (read)

## Step 1: Deploy Scanning Account Infrastructure

Deploy from FortiCNAPP UI in your scanning account (where scanning infrastructure will run):

1. Log into FortiCNAPP
2. Navigate to Settings > Cloud Accounts
3. Click Add Cloud Account and select AWS Agentless Scanning
4. In Step 1 "Set up AWS Scanning Account", click Launch Stack
5. AWS CloudFormation console opens with template pre-filled
6. Fill in parameters:
   - Regions: e.g., `ap-southeast-2`
   - VPCQuotaCheck: `Yes`
   - ResourceNamePrefix: e.g., `lacework-agentless`
   - ResourceNameSuffix: e.g., `acme`
7. Click Create stack and wait for completion
8. Save the stack outputs: `CrossAccountRoleArn`, `ECSTaskRoleArn`, `S3BucketArn`, `ExternalId`

## Step 2: Deploy Snapshot Roles (Organization)

Deploy a CloudFormation stack from your management account that creates a StackSet to deploy snapshot roles across monitored accounts:

1. Log into AWS Console with your management account
2. Return to FortiCNAPP UI setup wizard
3. In Step 2 "Integrate AWS Scanning Account with your AWS Organization", click Launch Stack (this opens CloudFormation in your management account)
4. Fill in parameters using outputs from Step 1:
   - CrossAccountRoleArn: from Step 1
   - ECSTaskRoleArn: from Step 1
   - S3BucketArn: from Step 1
   - ExternalId: from Step 1
   - MonitoredAccountDeployment: `SERVICE_MANAGED`
   - MonitoredAccountIds: `r-xxxxx` (your Organization root ID or OU IDs)
5. Click Create stack and wait for completion

## Step 3: Verify Integration Status

In FortiCNAPP, navigate to Settings > Integrations > Cloud accounts. The status displays as Success if all resources are installed correctly.

## Template Source

CloudFormation templates are generated from the FortiCNAPP UI with account-specific parameters (LaceworkServerToken, ExternalId). Templates should be downloaded/launched from the UI rather than stored separately.

Important: The ExternalId is hardcoded in the template and cannot be changed after generation. It's required for IAM trust relationships with FortiCNAPP.

## Using Existing VPC (Optional)

CloudFormation creates a new VPC by default. To use an existing VPC, update the ECS service after deployment.

### Requirements

| Resource | Requirement |
|----------|-------------|
| Subnet | Must be in the same region as ECS service |
| Security Group | Must allow outbound HTTPS (port 443/tcp) |
| Private Subnets | Must have NAT gateway or VPC endpoints |

### Find ECS Resources

```bash
# Find the ECS cluster
CLUSTER=$(aws ecs list-clusters \
  --query 'clusterArns[?contains(@, `lacework`)]' \
  --output text | awk -F'/' '{print $NF}')

# Find the service
SERVICE=$(aws ecs list-services \
  --cluster $CLUSTER \
  --query 'serviceArns[0]' \
  --output text | awk -F'/' '{print $NF}')

echo "Cluster: $CLUSTER"
echo "Service: $SERVICE"
```

### Update ECS Service

```bash
# Replace with your actual subnet and security group IDs
EXISTING_SUBNET="subnet-xxxxxxxxx"
EXISTING_SECURITY_GROUP="sg-xxxxxxxxx"

# Update service network configuration
aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$EXISTING_SUBNET],
    securityGroups=[$EXISTING_SECURITY_GROUP],
    assignPublicIp=DISABLED
  }" \
  --force-new-deployment

# Wait for service to stabilize
aws ecs wait services-stable --cluster $CLUSTER --services $SERVICE
```

### Verify the Update

```bash
# Check service status
aws ecs describe-services \
  --cluster $CLUSTER \
  --services $SERVICE \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'

# Verify tasks are in correct subnet
TASK_ARN=$(aws ecs list-tasks \
  --cluster $CLUSTER \
  --service-name $SERVICE \
  --query 'taskArns[0]' \
  --output text)

aws ecs describe-tasks \
  --cluster $CLUSTER \
  --tasks $TASK_ARN \
  --query 'tasks[0].attachments[0].details[?name==`subnetId`].value' \
  --output text
```

## Resources Provisioned

Scanning Account Stack:
- VPC with network infrastructure
- ECS cluster and service
- S3 bucket for scan results
- IAM roles (cross-account role, task role, task execution role)
- CloudWatch log groups

Snapshot Roles Stack (via StackSets):
- IAM snapshot roles in all monitored accounts
- Deployed from management account across AWS Organization

## References

- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/966589/agentless-workload-scanning" target="_blank">Deploying Agentless Workload Scanning on AWS</a>
- <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/269317/agentless-workload-scanning-faqs" target="_blank">Agentless Workload Scanning FAQs</a>
