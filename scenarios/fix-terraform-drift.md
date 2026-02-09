# Scenario: Fix Terraform Drift

## Problem Statement

After running `terraform plan`, you discover that your Terraform state doesn't match the actual infrastructure. Some resources were modified outside of Terraform, and some resources exist in state but not in reality. You need to reconcile the drift.

## Environment

- **Infrastructure**: AWS (EC2, S3, RDS)
- **Terraform Version**: 1.5+
- **State Backend**: S3 with DynamoDB locking
- **Team Size**: 5 developers
- **State**: Managed remotely in S3

## Symptoms

Running `terraform plan` shows:

```
# aws_instance.web will be destroyed and recreated
-/+ aws_instance.web (forces replacement)
  ~ ami = "ami-12345" -> "ami-67890" (forces replacement)
  
# aws_s3_bucket.data is missing from state
- aws_s3_bucket.data

# aws_security_group.web has been modified
~ aws_security_group.web
  ~ ingress {
      ~ cidr_blocks = ["10.0.0.0/8"] -> ["0.0.0.0/0"]
    }
```

## Step-by-Step Resolution

### Step 1: Understand the Drift

```bash
# Run plan to see all differences
terraform plan -out=tfplan

# Review the plan carefully
terraform show tfplan

# Check what resources are affected
terraform plan | grep -E "^[~+-]"
```

**Types of Drift:**

1. **Resources Modified Outside Terraform**
   - Someone changed infrastructure manually
   - Another tool modified resources
   - AWS console changes

2. **Resources Missing from State**
   - Resource deleted manually
   - State file corrupted/lost
   - Import never completed

3. **Resources Missing from Infrastructure**
   - Resource deleted outside Terraform
   - Failed creation not reflected in state

### Step 2: Investigate Root Cause

```bash
# Check Terraform state
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Check when state was last modified
aws s3 ls s3://terraform-state-bucket/prod/ --recursive

# Review CloudTrail logs (if enabled)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=web-instance
```

**Questions to Ask:**
- Who made manual changes?
- When did the drift occur?
- Was there a state file issue?
- Are there multiple Terraform workspaces?

### Step 3: Document Current State

```bash
# Export current state
terraform state pull > current-state.json

# List all resources in state
terraform state list > state-resources.txt

# Get current infrastructure (using AWS CLI)
aws ec2 describe-instances --filters "Name=tag:Name,Values=web" > actual-infra.json
```

### Step 4: Fix Drift - Strategy Selection

**Option A: Accept Manual Changes (Update Code)**

If manual changes are intentional and should be kept:

```bash
# 1. Update Terraform code to match reality
# Edit main.tf to reflect current AMI

# 2. Refresh state to match infrastructure
terraform refresh

# 3. Verify plan shows no changes
terraform plan
```

**Option B: Revert Manual Changes (Apply Code)**

If manual changes should be reverted:

```bash
# 1. Apply Terraform to revert changes
terraform apply

# 2. Verify infrastructure matches code
terraform plan
```

**Option C: Import Missing Resources**

If resources exist but not in state:

```bash
# 1. Get resource ID
aws s3api list-buckets --query 'Buckets[?Name==`data-bucket`].Name'

# 2. Import into state
terraform import aws_s3_bucket.data data-bucket

# 3. Verify import
terraform state show aws_s3_bucket.data
```

**Option D: Remove from State (Resource Deleted)**

If resource deleted but still in state:

```bash
# 1. Remove from state (doesn't delete resource, already gone)
terraform state rm aws_s3_bucket.data

# 2. Verify plan
terraform plan
```

### Step 5: Handle Specific Scenarios

#### Scenario 1: EC2 Instance AMI Changed

**Problem:** Instance AMI changed manually, Terraform wants to recreate.

**Solution A - Keep New AMI:**
```hcl
# Update code
resource "aws_instance" "web" {
  ami = "ami-67890"  # New AMI
  # ... other config
}
```

**Solution B - Revert to Original:**
```bash
# Terraform will recreate with original AMI
terraform apply
```

#### Scenario 2: Security Group Rules Modified

**Problem:** Security group rules changed manually.

**Solution:**
```bash
# Option 1: Refresh state (accepts manual changes)
terraform refresh

# Option 2: Apply to revert (restores code state)
terraform apply
```

#### Scenario 3: S3 Bucket Missing from State

**Problem:** Bucket exists but not in Terraform state.

**Solution:**
```bash
# Import bucket
terraform import aws_s3_bucket.data data-bucket-name

# Add resource to code if missing
# terraform state show will show current config
terraform state show aws_s3_bucket.data
```

#### Scenario 4: Resource Deleted but in State

**Problem:** Resource deleted manually, still in state.

**Solution:**
```bash
# Remove from state
terraform state rm aws_instance.deleted-instance

# If you want to recreate it, just apply
terraform apply
```

### Step 6: Verify Resolution

```bash
# Run plan - should show no changes
terraform plan

# If there are still differences, review carefully
terraform plan -detailed-exitcode

# Exit code 0: No changes
# Exit code 1: Error
# Exit code 2: Changes detected
```

### Step 7: Prevent Future Drift

**1. State Locking:**
```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "terraform-locks"  # Enable locking
    encrypt = true
  }
}
```

**2. IAM Policies:**
```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:*",
    "s3:*",
    "rds:*"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestTag/ManagedBy": "Terraform"
    }
  }
}
```

**3. Automated Drift Detection:**
```bash
#!/bin/bash
# CI/CD pipeline check
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  echo "ERROR: Infrastructure drift detected!"
  terraform plan
  exit 1
fi
```

**4. Regular State Validation:**
```bash
# Weekly drift check
terraform plan -out=/dev/null
```

## Root Cause Analysis

### Why Drift Occurs:

1. **Manual Changes**
   - Developers making quick fixes
   - Emergency changes
   - Lack of process

2. **Multiple Tools**
   - Using Terraform + CloudFormation
   - Using Terraform + Console
   - Using Terraform + Ansible

3. **State File Issues**
   - State file corruption
   - Lost state file
   - Wrong state file used

4. **Team Collaboration**
   - Multiple people applying changes
   - No coordination
   - State file conflicts

## Prevention Strategies

### 1. Enforce Infrastructure as Code

- Block manual changes via IAM
- Require all changes through Terraform
- Use tags to identify Terraform-managed resources

### 2. State Management

- Use remote state (S3 + DynamoDB)
- Enable state locking
- Version state files
- Regular backups

### 3. Process and Training

- Team training on Terraform
- Code review for Terraform changes
- Documented change process

### 4. Automation

- CI/CD for Terraform
- Automated drift detection
- Alerts on manual changes (CloudTrail)

### 5. Monitoring

- CloudTrail alerts for manual changes
- Regular `terraform plan` in CI/CD
- State file monitoring

## Example: Complete Fix Workflow

```bash
# 1. Check current drift
terraform plan > drift-report.txt

# 2. Review and categorize changes
# - Acceptable manual changes → Update code
# - Unacceptable changes → Revert
# - Missing resources → Import
# - Orphaned state → Remove

# 3. For each category, apply appropriate fix

# Accept manual AMI change
terraform refresh  # Update state
# Update code to match
vim main.tf  # Change AMI to match

# Revert security group change
terraform apply  # Revert to code state

# Import missing bucket
terraform import aws_s3_bucket.data my-bucket-name

# Remove deleted resource from state
terraform state rm aws_instance.old-instance

# 4. Verify no drift
terraform plan
# Should show: No changes. Infrastructure is up-to-date.

# 5. Commit fixes
git add .
git commit -m "Fix Terraform drift: update AMI, import bucket, remove deleted resource"
```

## Follow-up Questions

1. How do you handle drift in a multi-environment setup?
2. What if the drift affects production during business hours?
3. How do you prevent state file conflicts with multiple team members?
4. What's your strategy for handling drift in CI/CD pipelines?

## Key Takeaways

- Always investigate root cause before fixing
- Choose strategy based on business impact
- Use `terraform refresh` carefully (can hide issues)
- Import missing resources rather than recreating
- Remove deleted resources from state
- Prevent drift through process and tooling
- Regular drift detection in CI/CD
- Document all manual changes and reasons
