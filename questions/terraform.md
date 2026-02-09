# Terraform Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [State Management](#state-management)
- [Modules](#modules)
- [Workspaces & Environments](#workspaces--environments)
- [Advanced Topics](#advanced-topics)
- [Best Practices](#best-practices)

---

## Core Concepts

### Q1: What is Infrastructure as Code (IaC) and what are the benefits?

**Difficulty:** Junior

**Answer:**

Infrastructure as Code (IaC) is managing infrastructure through code files rather than manual processes.

**Benefits:**
- **Version Control**: Track changes, rollback, audit history
- **Consistency**: Same infrastructure every time
- **Speed**: Deploy infrastructure quickly and repeatedly
- **Documentation**: Code serves as documentation
- **Collaboration**: Team can review and contribute
- **Disaster Recovery**: Recreate infrastructure from code
- **Cost**: Reduce manual errors, optimize resources

**Terraform vs Others:**
- **Terraform**: Declarative, multi-cloud, state management
- **CloudFormation**: AWS-native, JSON/YAML
- **Ansible**: Configuration management, procedural
- **Pulumi**: Use programming languages (Python, TypeScript)

**Real-world Context:** Instead of clicking through AWS console to create 50 resources, write Terraform code once and apply to dev/staging/prod.

**Follow-up:** What's the difference between declarative and imperative IaC? (Declarative: describe desired state. Imperative: describe steps to achieve state)

---

### Q2: Explain Terraform's core concepts: Providers, Resources, Data Sources, and Variables.

**Difficulty:** Mid

**Answer:**

**Providers:**
- Plugins that interact with APIs (AWS, Azure, GCP, Kubernetes)
- Define resource types and data sources
- Configured with credentials and settings
- Example: `provider "aws" { region = "us-east-1" }`

**Resources:**
- Infrastructure components to create/manage
- Represent actual infrastructure (EC2 instance, S3 bucket)
- Have lifecycle (create, read, update, delete)
- Example: `resource "aws_instance" "web" { ... }`

**Data Sources:**
- Read-only information from existing infrastructure
- Don't create resources, just fetch data
- Used for referencing existing resources
- Example: `data "aws_ami" "latest" { ... }`

**Variables:**
- Input parameters for Terraform code
- Make code reusable and configurable
- Can have defaults, validation, descriptions
- Example: `variable "instance_type" { type = string }`

**Real-world Context:** Use data source to find latest AMI, variable for instance type, resource to create EC2 instance, provider to authenticate to AWS.

**Follow-up:** What's the difference between resource and data source? (Resource creates/manages infrastructure, data source reads existing infrastructure)

---

### Q3: What is Terraform state and why is it important?

**Difficulty:** Mid

**Answer:**

Terraform state is a file (terraform.tfstate) that maps Terraform configuration to real-world resources.

**Purpose:**
- **Mapping**: Links resources in code to actual infrastructure
- **Metadata**: Stores resource attributes, dependencies
- **Performance**: Tracks what exists to avoid unnecessary API calls
- **Dependency Tracking**: Knows resource relationships

**State File Contains:**
- Resource addresses and IDs
- Resource attributes (outputs, computed values)
- Dependency graph
- Terraform version

**Why Important:**
- Without state, Terraform doesn't know what exists
- Enables updates and deletes
- Tracks drift (changes outside Terraform)

**Real-world Context:** You create EC2 instance with Terraform. State file stores instance ID. Next run, Terraform knows instance exists and won't recreate it.

**Follow-up:** What happens if you lose the state file? (Terraform thinks resources don't exist, will try to create duplicates. Need to import existing resources or recreate)

---

### Q4: Explain Terraform's execution plan and apply workflow.

**Difficulty:** Junior

**Answer:**

**Workflow:**

1. **terraform init**: Downloads providers, initializes backend
2. **terraform plan**: Creates execution plan (dry-run)
   - Compares desired state (code) with current state (state file)
   - Shows what will be created, updated, destroyed
   - No changes made
3. **terraform apply**: Executes the plan
   - Creates/updates/destroys resources
   - Updates state file
   - Can auto-approve or require confirmation

**Plan Output:**
```
+ aws_instance.web
  + ami = "ami-12345"
  + instance_type = "t3.medium"
  
~ aws_security_group.web
  ~ ingress.0.cidr_blocks = ["0.0.0.0/0"] -> ["10.0.0.0/8"]
```

**Best Practice:** Always run `terraform plan` before `terraform apply` to review changes.

**Real-world Context:** Before deploying to production, run plan, review changes, get approval, then apply.

**Follow-up:** What's the difference between plan and apply? (Plan shows what will happen, apply actually makes changes)

---

### Q5: What are Terraform outputs and how are they used?

**Difficulty:** Junior

**Answer:**

Outputs expose values from Terraform configuration for use elsewhere.

**Use Cases:**
- Pass values between modules
- Display important information (IPs, URLs)
- Use in other Terraform configurations
- Integrate with CI/CD pipelines

**Syntax:**
```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip
  description = "Public IP of the web server"
}
```

**Accessing Outputs:**
- After apply: `terraform output instance_ip`
- In other configs: `module.vpc.vpc_id`
- Sensitive outputs: Mark as sensitive, hidden in plan

**Real-world Context:** VPC module outputs VPC ID. Application module uses that VPC ID to create resources in the VPC.

**Follow-up:** How do you pass outputs between modules? (Reference module output: `module.vpc.vpc_id`)

---

## State Management

### Q6: What is remote state and why would you use it?

**Difficulty:** Mid

**Answer:**

Remote state stores Terraform state file in a shared location (S3, Terraform Cloud, etc.) instead of locally.

**Benefits:**
- **Team Collaboration**: Multiple people can work on same infrastructure
- **State Locking**: Prevents concurrent modifications (DynamoDB)
- **Security**: State not stored on developer machines
- **Backup**: State backed up automatically
- **Consistency**: Everyone uses same state

**Backend Configuration:**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    dynamodb_table = "terraform-locks"
  }
}
```

**Real-world Context:** Team of 5 developers. Without remote state, each has local state file, conflicts occur. With remote state in S3, everyone uses same state.

**Follow-up:** What is state locking? (Prevents two people from applying simultaneously, uses DynamoDB for S3 backend)

---

### Q7: Explain Terraform state locking and how it works.

**Difficulty:** Mid

**Answer:**

State locking prevents concurrent Terraform operations on the same state file.

**How it Works:**
- When `terraform apply` starts, acquires lock
- Lock stored in backend (DynamoDB for S3, etc.)
- Other operations wait or fail if lock exists
- Lock released when operation completes
- If operation crashes, lock expires after timeout

**Backend Support:**
- **S3 + DynamoDB**: DynamoDB table stores locks
- **Terraform Cloud**: Built-in locking
- **Consul**: Distributed locking
- **Local**: No locking (problematic for teams)

**Configuration:**
```hcl
terraform {
  backend "s3" {
    dynamodb_table = "terraform-locks"  # Lock table
    encrypt = true
  }
}
```

**Real-world Context:** Developer A runs apply. Developer B tries to apply simultaneously. B's operation waits or fails with lock error, preventing state corruption.

**Follow-up:** What happens if a lock is stuck? (Manually delete lock from DynamoDB table, or wait for timeout)

---

### Q8: What is Terraform state drift and how do you detect it?

**Difficulty:** Mid

**Answer:**

State drift occurs when infrastructure is changed outside of Terraform (manual changes, other tools, bugs).

**Detection:**
- `terraform plan`: Compares desired state with actual state
- Shows differences as updates or destroys/recreates
- `terraform refresh`: Updates state file with current infrastructure (doesn't change infrastructure)

**Example Drift:**
- Code: `instance_type = "t3.medium"`
- Actual: Instance changed to `t3.large` manually
- Plan shows: `~ instance_type = "t3.medium" -> "t3.large"`

**Handling Drift:**
1. **Accept**: Update code to match reality, then apply
2. **Fix**: Revert manual change, then apply
3. **Import**: If resource deleted, import or recreate

**Prevention:**
- Use Terraform for all changes
- Restrict manual access
- Use policy as code (Sentinel, OPA)

**Real-world Context:** Someone manually changes security group rule. Next Terraform run detects drift and shows the change in plan.

**Follow-up:** What's the difference between plan and refresh? (Plan compares code vs state, refresh updates state from actual infrastructure)

---

### Q9: How do you import existing infrastructure into Terraform?

**Difficulty:** Mid

**Answer:**

Import brings existing infrastructure under Terraform management.

**Process:**
1. Write resource block in Terraform code (matching existing resource)
2. Run `terraform import <resource_address> <resource_id>`
3. Terraform adds resource to state file
4. Run `terraform plan` to see differences
5. Update code to match actual resource (or apply to match code)

**Example:**
```bash
terraform import aws_instance.web i-1234567890abcdef0
```

**Challenges:**
- Need to write resource block first
- May need multiple imports for related resources
- Code may not match actual resource exactly
- Use `terraform show` to see current state

**Tools:**
- `terraformer`: Auto-generates Terraform code from existing resources
- Manual import: More control, more work

**Real-world Context:** Company has 100 EC2 instances created manually. Want to manage with Terraform. Import each instance, then manage going forward.

**Follow-up:** What if you import a resource but the code doesn't match? (Plan will show differences, update code or apply to sync)

---

## Modules

### Q10: What are Terraform modules and why use them?

**Difficulty:** Mid

**Answer:**

Modules are reusable Terraform configurations that encapsulate related resources.

**Benefits:**
- **Reusability**: Write once, use many times
- **Organization**: Group related resources
- **Abstraction**: Hide complexity, expose simple interface
- **Testing**: Test module independently
- **Versioning**: Pin module versions

**Module Structure:**
```
modules/vpc/
  ├── main.tf
  ├── variables.tf
  ├── outputs.tf
  └── README.md
```

**Using Modules:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr = "10.0.0.0/16"
  environment = "prod"
}
```

**Real-world Context:** Create VPC module once. Use it for dev, staging, prod with different variables. Consistent infrastructure across environments.

**Follow-up:** What's the difference between local and remote modules? (Local: `./modules/vpc`, Remote: Git, S3, Terraform Registry)

---

### Q11: Explain module composition and when to use it.

**Difficulty:** Senior

**Answer:**

Module composition is using modules within modules to build complex infrastructure.

**Patterns:**
- **Composition**: Module calls other modules
- **Abstraction Layers**: Higher-level modules compose lower-level modules
- **Reusability**: Common modules used across different compositions

**Example:**
```
module "app" {
  source = "./modules/app"
  
  vpc_id = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}

module "networking" {
  source = "./modules/networking"
  ...
}
```

**When to Use:**
- Building complex architectures
- Reusing common patterns
- Separating concerns (networking, compute, data)
- Different teams own different modules

**Best Practices:**
- Keep modules focused (single responsibility)
- Use outputs to pass data between modules
- Document module interfaces clearly
- Version modules independently

**Real-world Context:** E-commerce platform module composes: VPC module, RDS module, ECS module, ALB module. Each module can be reused elsewhere.

**Follow-up:** How do you handle circular dependencies between modules? (Restructure to remove circular dependency, use data sources)

---

### Q12: What is the Terraform Registry and how do you use it?

**Difficulty:** Junior

**Answer:**

Terraform Registry is a public repository of Terraform modules and providers.

**Types:**
- **Public Modules**: Community-contributed modules
- **Private Modules**: Your organization's modules (Terraform Cloud)
- **Providers**: Official and community providers

**Using Registry Modules:**
```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "3.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

**Benefits:**
- Pre-built, tested modules
- Versioned and maintained
- Documentation and examples
- Community support

**Finding Modules:**
- Search registry.terraform.io
- Filter by provider, category
- Check downloads, ratings
- Review source code

**Real-world Context:** Need VPC with public/private subnets. Use `terraform-aws-modules/vpc/aws` instead of writing from scratch.

**Follow-up:** How do you publish your own module? (Push to GitHub, add to registry via Terraform Cloud, or use private registry)

---

## Workspaces & Environments

### Q13: What are Terraform workspaces and how do they differ from modules?

**Difficulty:** Mid

**Answer:**

Workspaces allow multiple state files for the same configuration, typically used for environments.

**How Workspaces Work:**
- Same Terraform code
- Different state files per workspace
- Switch workspaces: `terraform workspace select prod`
- List workspaces: `terraform workspace list`

**Use Case:**
```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select dev
terraform apply
```

**Workspaces vs Modules:**
- **Workspaces**: Same code, different state (environments)
- **Modules**: Reusable code components

**Limitations:**
- Can't use workspace-specific variables easily
- State files can diverge
- Not recommended for production (use separate directories/modules instead)

**Better Alternative:**
- Separate directories per environment
- Use modules for shared code
- Different variable files per environment

**Real-world Context:** Small team uses workspaces for dev/staging/prod. Larger teams use separate directories with modules.

**Follow-up:** Why might workspaces not be ideal for production? (State files can diverge, harder to manage, less isolation)

---

### Q14: How do you manage multiple environments (dev/staging/prod) with Terraform?

**Difficulty:** Mid

**Answer:**

**Approach 1: Separate Directories**
```
environments/
  ├── dev/
  │   ├── main.tf
  │   └── terraform.tfvars
  ├── staging/
  └── prod/
```

**Approach 2: Workspaces** (simpler, less isolation)
```bash
terraform workspace select dev
terraform apply -var-file=dev.tfvars
```

**Approach 3: Modules with Environment Variables**
```hcl
module "infrastructure" {
  source = "../modules"
  environment = var.environment
}
```

**Best Practices:**
- Use separate state backends per environment
- Different variable files per environment
- Environment-specific modules or configurations
- CI/CD pipelines per environment
- Restrict access (prod requires approval)

**Real-world Context:** 
- Dev: Auto-apply on merge
- Staging: Apply on release
- Prod: Manual approval required, run plan first

**Follow-up:** How do you prevent accidental prod changes? (Separate state backends, IAM permissions, approval gates in CI/CD)

---

## Advanced Topics

### Q15: Explain Terraform providers and how to use multiple providers.

**Difficulty:** Mid

**Answer:**

Providers are plugins that interact with APIs to manage resources.

**Multiple Providers:**
- Can use multiple providers in same configuration
- Configure each provider separately
- Use `alias` for multiple instances of same provider

**Example:**
```hcl
provider "aws" {
  region = "us-east-1"
  alias = "us_east"
}

provider "aws" {
  region = "eu-west-1"
  alias = "eu_west"
}

resource "aws_instance" "us" {
  provider = aws.us_east
  ...
}

resource "aws_instance" "eu" {
  provider = aws.eu_west
  ...
}
```

**Use Cases:**
- Multi-region deployments
- Multi-cloud (AWS + Azure)
- Different credentials per provider
- Cross-account resources

**Real-world Context:** Deploy to us-east-1 and eu-west-1. Use provider aliases to manage resources in both regions.

**Follow-up:** How do you authenticate multiple AWS providers? (Use different AWS profiles, assume roles, or explicit credentials)

---

### Q16: What are Terraform provisioners and when should you avoid them?

**Difficulty:** Mid

**Answer:**

Provisioners run scripts on local or remote machines after resource creation.

**Types:**
- **local-exec**: Run on machine running Terraform
- **remote-exec**: Run on created resource (SSH)
- **file**: Copy files to resource

**Example:**
```hcl
resource "aws_instance" "web" {
  ...
  
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo systemctl start httpd"
    ]
  }
}
```

**When to Avoid:**
- **Not Idempotent**: Can cause issues on re-runs
- **State Management**: Hard to track what provisioner did
- **Failure Handling**: If provisioner fails, resource created but not configured
- **Better Alternatives**: Use user_data, AMIs, configuration management (Ansible)

**Best Practices:**
- Use only when necessary
- Make idempotent
- Use null_resource for complex provisioning
- Prefer cloud-init/user_data for EC2
- Use configuration management tools instead

**Real-world Context:** Avoid provisioners. Use user_data for EC2 initialization, or Ansible for configuration management after Terraform creates infrastructure.

**Follow-up:** What's a better alternative to remote-exec? (user_data scripts, pre-built AMIs, or separate configuration management step)

---

### Q17: Explain Terraform functions and give examples.

**Difficulty:** Mid

**Answer:**

Terraform provides built-in functions for data manipulation.

**Common Functions:**

**String Functions:**
- `join(separator, list)`: Join list into string
- `split(separator, string)`: Split string into list
- `upper(string)`, `lower(string)`: Case conversion
- `substr(string, offset, length)`: Substring

**Collection Functions:**
- `length(list)`: Get length
- `element(list, index)`: Get element by index
- `lookup(map, key, default)`: Get map value
- `merge(map1, map2)`: Merge maps

**Numeric Functions:**
- `max(numbers...)`, `min(numbers...)`: Min/max
- `abs(number)`: Absolute value

**Type Conversion:**
- `tostring(value)`, `tonumber(value)`: Type conversion

**Example:**
```hcl
locals {
  name = join("-", [var.environment, var.app_name])
  subnet_cidrs = [for i in range(3) : cidrsubnet(var.vpc_cidr, 8, i)]
}
```

**Real-world Context:** Generate resource names dynamically: `join("-", [var.env, var.app, "vpc"])` → "prod-webapp-vpc"

**Follow-up:** What are for expressions? (Loop-like syntax: `[for x in list : x * 2]`)

---

### Q18: What are Terraform locals and when to use them?

**Difficulty:** Mid

**Answer:**

Locals are intermediate values computed from variables or other locals.

**Syntax:**
```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
  
  name_prefix = "${var.environment}-${var.app_name}"
}
```

**Use Cases:**
- Compute values used multiple times
- Simplify complex expressions
- Create reusable data structures
- Avoid repetition

**Benefits:**
- DRY (Don't Repeat Yourself)
- Easier to read and maintain
- Single source of truth

**Example:**
```hcl
locals {
  subnet_count = length(var.availability_zones)
  subnet_cidrs = [for i in range(local.subnet_count) : 
    cidrsubnet(var.vpc_cidr, 8, i)]
}

resource "aws_subnet" "private" {
  count = local.subnet_count
  cidr_block = local.subnet_cidrs[count.index]
}
```

**Real-world Context:** Compute subnet CIDRs from VPC CIDR. Use locals to calculate once, reference multiple times.

**Follow-up:** What's the difference between locals and variables? (Locals: computed internally, Variables: input from outside)

---

## Best Practices

### Q19: What are Terraform best practices for production?

**Difficulty:** Senior

**Answer:**

**State Management:**
- Use remote state (S3 + DynamoDB)
- Enable state locking
- Encrypt state at rest
- Version state files
- Backup state regularly

**Code Organization:**
- Use modules for reusability
- Separate environments (directories)
- Use variables, not hardcoded values
- Document with comments and READMEs
- Version control everything

**Security:**
- Never commit secrets (use variables, secrets manager)
- Use least privilege IAM roles
- Enable encryption
- Review plan before apply
- Use policy as code (Sentinel, OPA)

**CI/CD:**
- Run plan in CI, require approval for apply
- Run in separate pipeline stages
- Use separate state backends per environment
- Tag resources for cost tracking
- Use terraform fmt and validate

**Testing:**
- Test modules independently
- Use terraform validate
- Review plan output
- Test in dev before prod

**Real-world Context:** Production setup: Remote state in S3 with encryption, DynamoDB locking, plan runs in CI, manual approval for apply, separate backends per environment.

**Follow-up:** How do you handle secrets in Terraform? (Use AWS Secrets Manager, environment variables, or Terraform Cloud variables marked sensitive)

---

### Q20: How do you handle Terraform errors and troubleshooting?

**Difficulty:** Mid

**Answer:**

**Common Errors:**

**1. State Lock Errors:**
- Another operation in progress
- Stuck lock: Delete from DynamoDB (carefully!)

**2. Resource Already Exists:**
- Import existing resource
- Or delete and recreate

**3. Dependency Errors:**
- Check resource dependencies
- Use `depends_on` explicitly if needed
- Verify resource IDs in state

**4. Provider Authentication:**
- Check credentials
- Verify IAM permissions
- Check provider configuration

**Troubleshooting Steps:**
1. **Read Error Message**: Usually descriptive
2. **Check State**: `terraform show` to see current state
3. **Validate Configuration**: `terraform validate`
4. **Check Plan**: `terraform plan` shows what will happen
5. **Refresh State**: `terraform refresh` to sync with reality
6. **Check Logs**: Enable TF_LOG for detailed logs

**Debugging:**
```bash
export TF_LOG=DEBUG
terraform apply
```

**Real-world Context:** Apply fails with "resource already exists". Check if manually created, import it, or verify state file is correct.

**Follow-up:** What's the difference between validate and plan? (Validate: syntax check, Plan: execution simulation)

---

## Summary

Terraform is a powerful IaC tool. Master these concepts: state management, modules, providers, and best practices. Practice writing and organizing Terraform code, and understand how to troubleshoot common issues.

**Next Steps:**
- Practice writing Terraform modules
- Set up remote state with S3 + DynamoDB
- Work through Terraform tutorials
- Study for HashiCorp Terraform Associate certification
