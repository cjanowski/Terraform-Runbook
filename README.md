# Terraform SRE Runbook

## Table of Contents
1. [Emergency Contacts](#emergency-contacts)
2. [State Management](#state-management)
3. [Common Issues and Resolutions](#common-issues-and-resolutions)
4. [Drift Detection and Remediation](#drift-detection-and-remediation)
5. [Import and Migration](#import-and-migration)
6. [Disaster Recovery](#disaster-recovery)
7. [Best Practices](#best-practices)
8. [Troubleshooting Commands](#troubleshooting-commands)
9. [Rollback Procedures](#rollback-procedures)
10. [Security Incidents](#security-incidents)

---

## Emergency Contacts

| Role | Contact | Escalation Path |
|------|---------|----------------|
| On-Call Engineer | [Contact Info] | Primary |
| Infrastructure Team Lead | [Contact Info] | Secondary |
| Cloud Platform Team | [Contact Info] | Tertiary |
| Vendor Support | [Support Number] | Emergency |

---

## State Management

### State File Corruption

**Symptoms:**
- `terraform plan` fails with state parsing errors
- Unable to read state file
- State file contains invalid JSON

**Diagnosis:**
```bash
# Validate state file
terraform state list

# Check state file directly
cat terraform.tfstate | jq .

# Check remote state
terraform state pull > current-state.json
cat current-state.json | jq .

# View state history (if using versioned backend)
# For S3: aws s3api list-object-versions --bucket <bucket-name> --prefix <path>
```

**Resolution:**
1. Pull latest working state from backup
2. Restore from version control if state is committed
3. Use backend versioning to restore previous version
4. Rebuild state using `terraform import` if no backup available

```bash
# Backup current state before any fixes
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).json

# For S3 backend with versioning
aws s3api get-object \
  --bucket <bucket-name> \
  --key <state-key> \
  --version-id <version-id> \
  restored-state.json

# Push restored state
terraform state push restored-state.json
```

---

### State Lock Issues

**Symptoms:**
- Error: "Error acquiring the state lock"
- Operations timeout waiting for lock
- Lock ID shown in error message

**Diagnosis:**
```bash
# Check backend configuration
cat backend.tf

# For DynamoDB (AWS)
aws dynamodb get-item \
  --table-name <lock-table> \
  --key '{"LockID":{"S":"<state-path>"}}'

# For Consul
consul kv get <lock-path>

# Check who has the lock and when
terraform force-unlock <lock-id>  # Use with caution!
```

**Resolution:**
1. Verify no active terraform operations are running
2. Check if CI/CD pipeline is holding lock
3. Investigate if previous operation crashed
4. Force unlock only if certain no operations are running
5. Clean up zombie locks from failed operations

```bash
# Force unlock (DANGEROUS - ensure no one is running terraform)
terraform force-unlock <lock-id>

# For DynamoDB
aws dynamodb delete-item \
  --table-name <lock-table> \
  --key '{"LockID":{"S":"<state-path>"}}'

# Verify unlock successful
terraform plan
```

---

### State Drift from Manual Changes

**Symptoms:**
- `terraform plan` shows unexpected changes
- Resources show as needing updates when none were made
- Differences between actual infrastructure and state

**Diagnosis:**
```bash
# Run plan to see drift
terraform plan

# Refresh state to sync with reality
terraform refresh

# Show specific resource
terraform state show <resource-address>

# Compare with actual infrastructure
# AWS example:
aws ec2 describe-instances --instance-ids <id>
```

**Resolution:**
1. Identify what changed manually
2. Update terraform code to match reality, or
3. Let terraform revert manual changes
4. Import resources if created outside terraform
5. Implement policy to prevent manual changes

```bash
# Update state to match reality without changes
terraform apply -refresh-only

# Import manually created resource
terraform import <resource-type>.<name> <resource-id>

# Remove resource from state (if no longer managed)
terraform state rm <resource-address>
```

---

## Common Issues and Resolutions

### Issue: Terraform Init Fails

**Symptoms:**
- Unable to initialize working directory
- Backend configuration errors
- Provider download failures

**Diagnosis:**
```bash
# Run init with debug output
TF_LOG=DEBUG terraform init

# Check backend configuration
cat backend.tf

# Verify credentials
# AWS
aws sts get-caller-identity

# GCP
gcloud auth list

# Azure
az account show

# Check network connectivity
curl -I https://registry.terraform.io
```

**Resolution:**
```bash
# Clear .terraform directory and retry
rm -rf .terraform .terraform.lock.hcl
terraform init

# Reconfigure backend
terraform init -reconfigure

# Migrate state to new backend
terraform init -migrate-state

# Use specific provider versions
terraform init -upgrade

# Skip provider plugins (offline)
terraform init -get-plugins=false
```

---

### Issue: Plan Shows Perpetual Drift

**Symptoms:**
- Same resources show changes on every plan
- Changes revert after apply
- Computed attributes causing updates

**Diagnosis:**
```bash
# Run plan with detailed output
terraform plan -out=tfplan
terraform show tfplan

# Check specific resource configuration
terraform state show <resource-address>

# Look for lifecycle blocks and ignore_changes
grep -r "ignore_changes" .

# Check provider version compatibility
terraform version
```

**Resolution:**
1. Add `ignore_changes` for computed/API-updated attributes
2. Use `lifecycle` blocks appropriately
3. Update provider to latest version
4. Check for default values conflicting with API

```hcl
# Example: Ignore changes to certain attributes
resource "aws_instance" "example" {
  # ... configuration ...

  lifecycle {
    ignore_changes = [
      tags["LastModified"],
      user_data,
      ami
    ]
  }
}
```

---

### Issue: Provider Authentication Failures

**Symptoms:**
- Authentication errors during plan/apply
- Credential validation failures
- Timeout errors connecting to API

**Diagnosis:**
```bash
# Check environment variables
env | grep AWS_
env | grep GOOGLE_
env | grep ARM_

# Verify credentials directly
# AWS
aws sts get-caller-identity

# GCP
gcloud auth application-default print-access-token

# Azure
az account show

# Check provider configuration
terraform providers
```

**Resolution:**
```bash
# Set credentials via environment variables
# AWS
export AWS_ACCESS_KEY_ID="<key>"
export AWS_SECRET_ACCESS_KEY="<secret>"
export AWS_REGION="us-east-1"

# GCP
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/keyfile.json"

# Azure
export ARM_CLIENT_ID="<client-id>"
export ARM_CLIENT_SECRET="<client-secret>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"
export ARM_TENANT_ID="<tenant-id>"

# Use assume role (AWS)
export AWS_PROFILE="<profile-name>"

# Refresh credentials
# AWS SSO
aws sso login --profile <profile>

# Re-run terraform
terraform plan
```

---

### Issue: Dependency Errors

**Symptoms:**
- "Resource X depends on Y but Y is not in state"
- Cycle errors in resource graph
- Resources created in wrong order

**Diagnosis:**
```bash
# Generate dependency graph
terraform graph | dot -Tpng > graph.png

# Show resources and dependencies
terraform state list
terraform state show <resource>

# Validate configuration
terraform validate

# Check for circular dependencies
terraform graph | grep cycle
```

**Resolution:**
1. Add explicit `depends_on` where needed
2. Remove circular dependencies
3. Split configurations into separate workspaces
4. Use data sources instead of resource references
5. Refactor module dependencies

```hcl
# Example: Explicit dependency
resource "aws_instance" "app" {
  # ... configuration ...

  depends_on = [
    aws_security_group.app,
    aws_db_instance.primary
  ]
}
```

---

### Issue: Timeout Errors

**Symptoms:**
- Operations timeout during apply
- Resource creation takes too long
- Waiter timeouts

**Diagnosis:**
```bash
# Check resource timeouts
terraform show | grep timeout

# Run with debug logging
TF_LOG=DEBUG terraform apply

# Check cloud provider status
# AWS
aws health describe-events

# Check for rate limiting
# AWS
aws service-quotas list-service-quotas --service-code ec2
```

**Resolution:**
```bash
# Increase timeouts in resource
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Resource-level timeout
resource "aws_db_instance" "example" {
  # ... configuration ...

  timeouts {
    create = "60m"
    update = "60m"
    delete = "60m"
  }
}

# Apply with parallelism adjustment
terraform apply -parallelism=1

# Increase provider timeout
provider "aws" {
  max_retries = 10
  # ... other config ...
}
```

---

## Drift Detection and Remediation

### Detecting Drift

```bash
# Standard drift check
terraform plan -detailed-exitcode
# Exit code 0: no changes
# Exit code 1: error
# Exit code 2: changes detected

# Refresh only mode (safe)
terraform plan -refresh-only

# Generate drift report
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > drift-report.json

# Compare specific resources
terraform state show <resource> > current-state.txt
# Compare with expected configuration
```

### Automated Drift Detection

```bash
# Scheduled drift detection script
#!/bin/bash
cd /path/to/terraform
terraform init -backend=true
terraform plan -detailed-exitcode > /dev/null 2>&1
EXIT_CODE=$?

if [ $EXIT_CODE -eq 2 ]; then
    echo "DRIFT DETECTED"
    terraform plan > drift-report-$(date +%Y%m%d).txt
    # Send alert
    # Notify team
fi
```

### Remediating Drift

```bash
# Option 1: Apply to fix drift
terraform apply -auto-approve

# Option 2: Update state to match reality
terraform apply -refresh-only -auto-approve

# Option 3: Update code to match reality
# Edit .tf files then apply

# Option 4: Import external changes
terraform import <resource_type>.<name> <id>
```

---

## Import and Migration

### Importing Existing Resources

```bash
# List resources to import
# Identify resource type and ID from cloud provider

# Import single resource
terraform import aws_instance.example i-1234567890abcdef0

# Import with module path
terraform import module.vpc.aws_vpc.main vpc-12345678

# Generate import blocks (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf

# Bulk import using for loop
for instance in $(aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output text); do
  terraform import "aws_instance.imported[\"$instance\"]" "$instance"
done
```

### State Migration

```bash
# Migrate from local to remote backend
# 1. Add backend configuration
cat >> backend.tf << EOF
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
EOF

# 2. Initialize with migration
terraform init -migrate-state

# Migrate between remote backends
# 1. Update backend.tf with new configuration
# 2. Run migration
terraform init -migrate-state

# Manual state migration
terraform state pull > local-backup.tfstate
# Update backend.tf
terraform init
terraform state push local-backup.tfstate
```

### Moving Resources Between States

```bash
# Move resource within same state
terraform state mv aws_instance.old_name aws_instance.new_name

# Move resource to module
terraform state mv aws_instance.example module.app.aws_instance.example

# Move between states
# Source state
terraform state pull > temp-state.json
terraform state rm aws_instance.example

# Destination state
cd ../destination
terraform state pull > dest-backup.json
# Extract resource from temp-state.json and add to dest state
terraform state push modified-state.json
```

---

## Disaster Recovery

### State Recovery Procedures

**Complete State Loss:**

```bash
# Step 1: Check for backups
# S3 versioning
aws s3api list-object-versions --bucket <bucket> --prefix <path>

# Terraform Cloud/Enterprise
# Use UI or API to download previous state versions

# Step 2: If no backup, rebuild via import
# Create resource configurations
# Import all resources
terraform import <resource_type>.<name> <id>

# Step 3: Verify state
terraform plan  # Should show no changes

# Step 4: Create backup
terraform state pull > recovered-state-$(date +%Y%m%d).json
```

### Corrupted State Recovery

```bash
# Attempt automatic fix
terraform state pull | jq . > fixed-state.json
terraform state push fixed-state.json

# Manual recovery
# 1. Backup current state
terraform state pull > corrupted-$(date +%Y%m%d).json

# 2. Get previous version
# For S3
aws s3api get-object \
  --bucket <bucket> \
  --key <key> \
  --version-id <previous-version> \
  recovered-state.json

# 3. Push recovered state
terraform state push recovered-state.json

# 4. Verify
terraform plan
```

### Emergency Rollback

```bash
# Quick rollback steps
# 1. Get state from before apply
terraform state pull > current-broken-state.json

# 2. Restore previous state
# From S3 versioning
aws s3api get-object \
  --bucket <bucket> \
  --key <key> \
  --version-id <previous-version> \
  previous-state.json

terraform state push previous-state.json

# 3. Revert code changes
git log --oneline
git checkout <previous-commit> -- .

# 4. Apply to fix infrastructure
terraform plan
terraform apply

# 5. Document incident
```

---

## Best Practices

### Pre-Apply Checklist

- [ ] Run `terraform fmt` to format code
- [ ] Run `terraform validate` to check syntax
- [ ] Review `terraform plan` output thoroughly
- [ ] Check for destructive changes (deletions, recreations)
- [ ] Verify credentials and permissions
- [ ] Ensure state lock is not held
- [ ] Backup current state
- [ ] Document changes in commit message
- [ ] Notify team of upcoming changes
- [ ] Have rollback plan ready

### Safe Apply Procedure

```bash
# Step 1: Format and validate
terraform fmt -recursive
terraform validate

# Step 2: Generate plan
terraform plan -out=tfplan

# Step 3: Review plan
terraform show tfplan

# Step 4: Save plan for review
terraform show -json tfplan > plan-review.json

# Step 5: Apply plan
terraform apply tfplan

# Step 6: Verify
terraform plan  # Should show no changes

# Step 7: Backup state
terraform state pull > backup-post-apply-$(date +%Y%m%d).json
```

### State Backup Strategy

```bash
# Automated backup script
#!/bin/bash
BACKUP_DIR="/backup/terraform"
PROJECT_NAME="my-project"
DATE=$(date +%Y%m%d-%H%M%S)

cd /path/to/terraform

# Pull state
terraform state pull > "${BACKUP_DIR}/${PROJECT_NAME}-${DATE}.json"

# Keep only last 30 days
find "${BACKUP_DIR}" -name "${PROJECT_NAME}-*.json" -mtime +30 -delete

# Upload to S3 for redundancy
aws s3 cp "${BACKUP_DIR}/${PROJECT_NAME}-${DATE}.json" \
  "s3://backup-bucket/terraform/${PROJECT_NAME}/${DATE}.json"
```

---

## Troubleshooting Commands

### State Inspection

```bash
# List all resources
terraform state list

# Show specific resource
terraform state show <resource-address>

# Show all attributes of resource
terraform state show -json <resource-address> | jq .

# Find resources matching pattern
terraform state list | grep <pattern>

# Show outputs
terraform output
terraform output -json
terraform output <output-name>
```

### Configuration Debugging

```bash
# Enable debug logging
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform-debug.log

# Trace level (very verbose)
export TF_LOG=TRACE

# Component-specific logging
export TF_LOG_CORE=TRACE
export TF_LOG_PROVIDER=TRACE

# Console output formatting
terraform console
> var.my_variable
> local.my_local
> module.my_module.output_name

# Validate configuration
terraform validate -json

# Check formatting issues
terraform fmt -check -recursive
```

### Provider Debugging

```bash
# Show provider versions
terraform providers

# Lock provider versions
terraform providers lock

# Upgrade providers
terraform init -upgrade

# Show provider schema
terraform providers schema -json > schema.json

# Test provider credentials
terraform console
> provider::aws::region
```

### Plan Analysis

```bash
# Generate detailed plan
terraform plan -out=tfplan

# Show human-readable plan
terraform show tfplan

# Show JSON plan
terraform show -json tfplan | jq .

# Show specific resource changes
terraform show -json tfplan | jq '.resource_changes[] | select(.address == "aws_instance.example")'

# Count resource changes
terraform show -json tfplan | jq '.resource_changes | group_by(.change.actions[0]) | map({action: .[0].change.actions[0], count: length})'

# Find destructive changes
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions[] | contains("delete"))'
```

---

## Rollback Procedures

### Code Rollback

```bash
# View recent commits
git log --oneline -10

# Revert to specific commit
git checkout <commit-hash> -- .

# Run plan to see changes
terraform plan

# Apply rollback
terraform apply

# If using branches
git checkout main
git revert <bad-commit-hash>
git push
```

### State Rollback

```bash
# List state versions (S3 example)
aws s3api list-object-versions \
  --bucket <bucket> \
  --prefix <state-key> \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Download specific version
aws s3api get-object \
  --bucket <bucket> \
  --key <state-key> \
  --version-id <version-id> \
  rollback-state.json

# Backup current state
terraform state pull > before-rollback-$(date +%Y%m%d).json

# Push rolled-back state
terraform state push rollback-state.json

# Verify
terraform plan
```

### Partial Rollback

```bash
# Rollback specific resources only
# 1. Get resource from old state
cat old-state.json | jq '.resources[] | select(.type == "aws_instance" and .name == "example")'

# 2. Remove from current state
terraform state rm aws_instance.example

# 3. Import with old configuration
terraform import aws_instance.example <instance-id>

# Or destroy and recreate
terraform destroy -target=aws_instance.example
git checkout <old-commit> -- instance.tf
terraform apply -target=aws_instance.example
```

---

## Security Incidents

### Exposed Credentials in State

**Immediate Actions:**
```bash
# 1. Rotate exposed credentials immediately
# AWS example
aws iam create-access-key --user-name <username>
aws iam delete-access-key --access-key-id <old-key> --user-name <username>

# 2. Remove sensitive data from state if possible
# This is difficult - state often needs sensitive data

# 3. Restrict state file access
# S3 example
aws s3api put-bucket-policy --bucket <bucket> --policy file://restrict-policy.json

# 4. Enable encryption at rest
# S3
aws s3api put-bucket-encryption \
  --bucket <bucket> \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# 5. Audit access logs
aws s3api get-bucket-logging --bucket <bucket>
```

### Unauthorized State Access

```bash
# Check who accessed state
# S3 access logs
aws s3api get-bucket-logging --bucket <bucket>
aws s3 cp s3://<log-bucket>/<prefix> . --recursive

# CloudTrail for API calls
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<bucket> \
  --max-results 50

# Review IAM policies
aws iam get-role-policy --role-name <role> --policy-name <policy>

# Restrict access
aws s3api put-bucket-policy --bucket <bucket> --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": "arn:aws:s3:::<bucket>/*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalAccount": "<account-id>"
      }
    }
  }]
}'
```

### Malicious Changes Detected

```bash
# 1. Lock state immediately
# Enable required state lock if not already

# 2. Review recent changes
terraform state pull | jq '.lineage, .serial'
git log --all --source --since="1 week ago"

# 3. Check for unauthorized resources
terraform state list
terraform state show <suspicious-resource>

# 4. Review Terraform logs
cat terraform-debug.log | grep -A 5 "apply"

# 5. Restore from known good state
terraform state push <verified-backup>.json

# 6. Change all credentials
# Rotate AWS keys, service account keys, etc.

# 7. Review and restrict permissions
terraform providers
# Check provider authentication methods
```

---

## Monitoring and Alerting

### State Monitoring

```bash
# Monitor state changes
# Script to run periodically
#!/bin/bash
STATE_FILE="s3://bucket/path/terraform.tfstate"
LAST_SERIAL=$(cat last-serial.txt)

CURRENT_SERIAL=$(aws s3 cp $STATE_FILE - | jq -r '.serial')

if [ "$CURRENT_SERIAL" != "$LAST_SERIAL" ]; then
  echo "State changed: serial $LAST_SERIAL -> $CURRENT_SERIAL"
  # Send notification
  echo "$CURRENT_SERIAL" > last-serial.txt
fi
```

### Drift Monitoring

```bash
# Continuous drift detection
#!/bin/bash
cd /path/to/terraform

terraform init -input=false
terraform plan -detailed-exitcode -out=drift.tfplan

if [ $? -eq 2 ]; then
  echo "DRIFT DETECTED"
  terraform show drift.tfplan > drift-report.txt
  # Send alert with drift-report.txt
fi
```

---

## CI/CD Integration

### GitLab CI Example

```yaml
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_STATE_NAME: ${CI_COMMIT_REF_SLUG}

.terraform:
  image: hashicorp/terraform:latest
  before_script:
    - cd ${TF_ROOT}
    - terraform init -backend-config="key=${TF_STATE_NAME}.tfstate"

validate:
  extends: .terraform
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check -recursive

plan:
  extends: .terraform
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan

apply:
  extends: .terraform
  stage: apply
  script:
    - terraform apply tfplan
  when: manual
  only:
    - main
```

---

## Useful Resources

- Terraform Documentation: https://www.terraform.io/docs/
- Terraform Registry: https://registry.terraform.io/
- Terraform Best Practices: https://www.terraform-best-practices.com/
- State Management Guide: https://www.terraform.io/docs/language/state/

---

*Last Updated: [Date]*  
*Maintained By: Cory Janowski*  
*Version: 1.0*
