# Agent Installation (Optional)

The FortiCNAPP agent provides runtime threat detection, file integrity monitoring, and deeper host-level visibility beyond what agentless scanning covers.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/cli-reference/784655/lacework-agent-aws-install" target="_blank">Lacework Agent AWS Install</a>

## Prerequisites

Install the Lacework CLI if not already installed:

```bash
brew install lacework/tap/lacework-cli
```

Or:

```bash
curl https://raw.githubusercontent.com/lacework/go-sdk/main/cli/install.sh | sudo bash
```

## Installation Methods

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
