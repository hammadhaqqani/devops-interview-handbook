# CI/CD Interview Questions

## Table of Contents
- [CI/CD Fundamentals](#cicd-fundamentals)
- [Jenkins](#jenkins)
- [GitLab CI](#gitlab-ci)
- [GitHub Actions](#github-actions)
- [Pipeline Design](#pipeline-design)
- [Best Practices](#best-practices)

---

## CI/CD Fundamentals

### Q1: What is CI/CD and what are the benefits?

**Difficulty:** Junior

**Answer:**

**CI (Continuous Integration):**
- Developers merge code frequently (multiple times per day)
- Automated builds and tests run on each merge
- Early detection of integration issues
- Fast feedback loop

**CD (Continuous Delivery/Deployment):**
- **Continuous Delivery**: Code is always deployable, manual deployment
- **Continuous Deployment**: Automatic deployment to production
- Automated deployment pipelines
- Reduce deployment risk

**Benefits:**
- **Faster Releases**: Automate manual steps
- **Higher Quality**: Automated testing catches bugs early
- **Reduced Risk**: Small, frequent changes easier to rollback
- **Faster Feedback**: Know immediately if code works
- **Consistency**: Same process every time
- **Documentation**: Pipeline serves as deployment documentation

**CI/CD Pipeline Stages:**
1. Source (Git)
2. Build (compile, package)
3. Test (unit, integration, e2e)
4. Deploy (staging, production)

**Real-world Context:** Instead of manual builds and deployments taking hours, CI/CD automates everything. Push code → tests run → deploy automatically.

**Follow-up:** What's the difference between Continuous Delivery and Continuous Deployment? (Delivery: auto to staging, manual to prod. Deployment: auto to prod)

---

### Q2: Explain the difference between CI and CD.

**Difficulty:** Junior

**Answer:**

**CI (Continuous Integration):**
- Focus: Integrating code changes frequently
- Activities: Build, test, code quality checks
- Frequency: On every commit/merge
- Goal: Catch integration issues early

**CD - Continuous Delivery:**
- Focus: Keep code deployable
- Activities: Automated deployment to staging
- Frequency: After CI passes
- Goal: Code always ready for production (manual trigger)

**CD - Continuous Deployment:**
- Focus: Automatically deploy to production
- Activities: Automated deployment to production
- Frequency: After CI passes
- Goal: Fastest time to market

**Key Difference:**
- **CI**: Build and test
- **Continuous Delivery**: CI + auto deploy to staging, manual to prod
- **Continuous Deployment**: CI + auto deploy to prod

**Real-world Context:** 
- CI: Every commit triggers build and tests
- Continuous Delivery: Auto deploy to staging, manual approval for prod
- Continuous Deployment: Auto deploy to prod (requires high confidence)

**Follow-up:** When would you use Continuous Deployment vs Delivery? (Deployment: high test coverage, feature flags, canary deployments. Delivery: need manual approval, compliance)

---

### Q3: What are the typical stages of a CI/CD pipeline?

**Difficulty:** Mid

**Answer:**

**Typical Pipeline Stages:**

1. **Source/Checkout:**
   - Get code from repository
   - Checkout specific branch/commit

2. **Build:**
   - Compile code
   - Install dependencies
   - Create artifacts (JAR, Docker image, etc.)

3. **Test:**
   - Unit tests
   - Integration tests
   - Code coverage
   - Linting/static analysis

4. **Package:**
   - Create deployment artifacts
   - Build Docker images
   - Version artifacts

5. **Deploy to Staging:**
   - Deploy to test environment
   - Run smoke tests
   - Integration testing

6. **Deploy to Production:**
   - Manual approval (Continuous Delivery)
   - Or automatic (Continuous Deployment)
   - Blue-green or canary deployment

7. **Post-Deployment:**
   - Health checks
   - Monitoring verification
   - Rollback if issues

**Example Pipeline:**
```
Commit → Build → Test → Package → Deploy Staging → 
  Test Staging → Approve → Deploy Prod → Verify
```

**Real-world Context:** Web application pipeline: Checkout → npm install → npm test → build → Docker build → deploy to ECS staging → e2e tests → approve → deploy to ECS prod.

**Follow-up:** What happens if a stage fails? (Pipeline stops, notification sent, fix issue and retry)

---

## Jenkins

### Q4: Explain Jenkins architecture: Master and Agents.

**Difficulty:** Mid

**Answer:**

**Jenkins Master:**
- Central server that manages builds
- Schedules builds, monitors agents
- Stores configuration, build history
- Serves web UI
- Can execute builds directly (not recommended)

**Jenkins Agents (Nodes):**
- Machines that execute builds
- Can be physical or virtual
- Different OS/environments
- Offloads work from master
- Can be static or dynamic (cloud agents)

**Communication:**
- Master communicates with agents via SSH or JNLP
- Agents report status back to master
- Master distributes builds to available agents

**Benefits of Agents:**
- Distribute load
- Use different environments (Windows, Linux, macOS)
- Isolate builds
- Scale horizontally

**Real-world Context:** Master on single server. Agents on multiple servers (Linux for builds, Windows for tests, macOS for iOS builds).

**Follow-up:** What's the difference between static and dynamic agents? (Static: always running, Dynamic: created on-demand, destroyed after)

---

### Q5: What is a Jenkinsfile and how does it work?

**Difficulty:** Mid

**Answer:**

Jenkinsfile is a text file that defines the pipeline as code, stored in repository.

**Benefits:**
- Version controlled with code
- Reviewable (code review)
- Reusable across projects
- Same process for all branches

**Syntax:**
- **Declarative Pipeline**: Simpler, structured syntax
- **Scripted Pipeline**: Groovy-based, more flexible

**Declarative Example:**
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
}
```

**Scripted Example:**
```groovy
node {
    stage('Build') {
        sh 'mvn clean package'
    }
    stage('Test') {
        sh 'mvn test'
    }
}
```

**Real-world Context:** Pipeline defined in Jenkinsfile in repo. Any branch can have different pipeline. Changes reviewed like code.

**Follow-up:** What's the difference between declarative and scripted pipelines? (Declarative: simpler, structured. Scripted: more flexible, Groovy)

---

### Q6: Explain Jenkins pipeline stages, steps, and post actions.

**Difficulty:** Mid

**Answer:**

**Stages:**
- Logical divisions of pipeline (Build, Test, Deploy)
- Run sequentially (or parallel)
- Can have conditions (when, if)

**Steps:**
- Individual commands within a stage
- Execute shell commands, call plugins
- Atomic operations

**Post Actions:**
- Run after stages complete (or fail)
- Always, success, failure, unstable, changed
- Use for: notifications, cleanup, reports

**Example:**
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
                failure {
                    mail to: 'team@example.com',
                         subject: "Build Failed",
                         body: "Build failed. Check ${env.BUILD_URL}"
                }
            }
        }
    }
    post {
        always {
            cleanWs()  // Clean workspace
        }
        success {
            echo 'Pipeline succeeded!'
        }
    }
}
```

**Real-world Context:** After test stage, always publish test results. On failure, send email notification. After pipeline, always clean workspace.

**Follow-up:** What's the difference between post in stage and post in pipeline? (Stage post: after that stage, Pipeline post: after all stages)

---

### Q7: How do you handle secrets in Jenkins?

**Difficulty:** Mid

**Answer:**

**Jenkins Credentials:**
- Store secrets securely (encrypted)
- Types: Username/password, SSH keys, API tokens, files
- Can be scoped: Global or folder-specific

**Using Credentials:**
```groovy
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY',
                    passwordVariable: 'AWS_SECRET_KEY'
                )]) {
                    sh 'aws s3 cp file.txt s3://bucket/'
                }
            }
        }
    }
}
```

**Best Practices:**
- Use Credentials plugin, not plain text
- Rotate secrets regularly
- Use least privilege
- Don't log secrets (use withCredentials)
- Use external secret management (HashiCorp Vault, AWS Secrets Manager)

**Real-world Context:** Pipeline needs AWS credentials. Store in Jenkins Credentials, reference by ID. Credentials injected as env vars, not logged.

**Follow-up:** How do you prevent secrets from appearing in logs? (Use withCredentials, don't echo secrets, mask passwords in console)

---

## GitLab CI

### Q8: What is GitLab CI/CD and how does it work?

**Difficulty:** Mid

**Answer:**

GitLab CI/CD is built into GitLab, provides CI/CD capabilities without separate tool.

**Components:**
- **GitLab Runner**: Executes jobs (like Jenkins agents)
- **.gitlab-ci.yml**: Pipeline configuration file
- **GitLab CI/CD**: Orchestrates pipelines

**How it Works:**
1. Push code to GitLab
2. GitLab detects `.gitlab-ci.yml`
3. Creates pipeline with jobs
4. GitLab Runner executes jobs
5. Results displayed in GitLab UI

**Example .gitlab-ci.yml:**
```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  script:
    - npm test

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  only:
    - main
```

**Real-world Context:** Push to GitLab → pipeline runs automatically → build → test → deploy to prod (if main branch).

**Follow-up:** What's the difference between GitLab CI and Jenkins? (GitLab CI: integrated, YAML-based. Jenkins: separate tool, more plugins, Groovy)

---

### Q9: Explain GitLab CI stages, jobs, and artifacts.

**Difficulty:** Mid

**Answer:**

**Stages:**
- Groups of jobs that run in sequence
- Jobs in same stage run in parallel
- Defined in `stages:` section

**Jobs:**
- Individual tasks (build, test, deploy)
- Run on GitLab Runner
- Can run in parallel or sequence
- Have scripts, environment, tags

**Artifacts:**
- Files created during job
- Passed to subsequent jobs
- Can be downloaded from UI
- Expire after time period

**Example:**
```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - mvn clean package
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

test:
  stage: test
  script:
    - mvn test
  dependencies:
    - build  # Get artifacts from build job

deploy:
  stage: deploy
  script:
    - scp target/*.jar server:/app/
  only:
    - main
```

**Real-world Context:** Build job creates JAR file. Test job uses JAR. Deploy job deploys JAR. Artifacts passed between jobs.

**Follow-up:** What's the difference between artifacts and cache? (Artifacts: job outputs, passed to next jobs. Cache: dependencies, speed up builds)

---

### Q10: How do you use GitLab CI variables and environments?

**Difficulty:** Mid

**Answer:**

**Variables:**
- Key-value pairs for configuration
- Can be defined: Project, Group, Instance level
- Protected: Only available to protected branches
- Masked: Hidden in logs

**Using Variables:**
```yaml
deploy:
  script:
    - echo $CI_COMMIT_SHA
    - echo $DEPLOY_TOKEN
  variables:
    DEPLOY_ENV: "production"
```

**Predefined Variables:**
- `CI_COMMIT_SHA`: Commit hash
- `CI_COMMIT_REF_NAME`: Branch/tag name
- `CI_JOB_NAME`: Job name
- `CI_PIPELINE_ID`: Pipeline ID

**Environments:**
- Represent deployment targets (staging, production)
- Track deployments
- Can have URLs, manual deployments
- Show deployment history

**Example:**
```yaml
deploy_staging:
  stage: deploy
  script:
    - deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com

deploy_prod:
  stage: deploy
  script:
    - deploy.sh production
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

**Real-world Context:** Use variables for API keys, environment names. Use environments to track deployments, show URLs, enable manual deployments.

**Follow-up:** What's the difference between project and group variables? (Project: specific to project, Group: shared across projects in group)

---

## GitHub Actions

### Q11: What is GitHub Actions and how does it work?

**Difficulty:** Mid

**Answer:**

GitHub Actions is CI/CD platform integrated into GitHub, allows automation of workflows.

**Components:**
- **Workflows**: Automated processes (defined in YAML)
- **Events**: Triggers (push, pull_request, schedule)
- **Jobs**: Run on runners (GitHub-hosted or self-hosted)
- **Steps**: Individual tasks within jobs
- **Actions**: Reusable components

**Workflow File:**
- Stored in `.github/workflows/`
- YAML format
- Triggered by events

**Example:**
```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'
    - name: Install dependencies
      run: npm install
    - name: Run tests
      run: npm test
```

**Real-world Context:** Push to GitHub → workflow runs → checkout code → setup environment → install deps → test → deploy.

**Follow-up:** What's the difference between GitHub Actions and GitLab CI? (GitHub Actions: GitHub-native, marketplace. GitLab CI: GitLab-native, integrated)

---

### Q12: Explain GitHub Actions workflows, jobs, and steps.

**Difficulty:** Mid

**Answer:**

**Workflows:**
- Automated processes defined in YAML
- Triggered by events (push, PR, schedule)
- Can have multiple jobs

**Jobs:**
- Set of steps that run on same runner
- Run in parallel by default (or sequentially with `needs`)
- Can have conditions (`if`)
- Run on different runners (OS, matrix)

**Steps:**
- Individual tasks within job
- Can run commands or use actions
- Share data via outputs
- Run sequentially

**Example:**
```yaml
name: Build and Test

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Build
      run: npm run build
    - name: Upload artifact
      uses: actions/upload-artifact@v2
      with:
        name: dist
        path: dist/

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
    - uses: actions/checkout@v2
    - name: Install dependencies
      run: npm install
    - name: Run tests
      run: npm test
```

**Real-world Context:** Build job compiles code, creates artifact. Test job runs tests. Test depends on build (needs: build).

**Follow-up:** How do jobs share data? (Artifacts: upload-artifact/download-artifact actions, or job outputs)

---

### Q13: How do you use GitHub Actions secrets and environments?

**Difficulty:** Mid

**Answer:**

**Secrets:**
- Encrypted variables stored in GitHub
- Can be: Repository, Organization, Environment secrets
- Accessed via `${{ secrets.SECRET_NAME }}`
- Never printed in logs

**Using Secrets:**
```yaml
deploy:
  steps:
  - name: Deploy
    env:
      AWS_ACCESS_KEY: ${{ secrets.AWS_ACCESS_KEY }}
      AWS_SECRET_KEY: ${{ secrets.AWS_SECRET_KEY }}
    run: |
      aws s3 cp file.txt s3://bucket/
```

**Environments:**
- Deployment targets (staging, production)
- Can have: Secrets, protection rules, reviewers
- Manual approval for production
- Environment-specific secrets

**Example:**
```yaml
deploy:
  environment: production
  steps:
  - name: Deploy to production
    env:
      API_KEY: ${{ secrets.PROD_API_KEY }}
    run: deploy.sh
```

**Protection Rules:**
- Required reviewers (manual approval)
- Wait timer
- Deployment branches (only main)

**Real-world Context:** Store AWS credentials as secrets. Use environment for production with required reviewers. Deploy requires manual approval.

**Follow-up:** What's the difference between repository and environment secrets? (Repository: available to all workflows, Environment: specific to environment, can have reviewers)

---

## Pipeline Design

### Q14: How do you design a CI/CD pipeline for microservices?

**Difficulty:** Senior

**Answer:**

**Challenges:**
- Multiple services, different tech stacks
- Independent deployment
- Service dependencies
- Shared infrastructure

**Pipeline Design:**

**1. Per-Service Pipelines:**
- Each service has own pipeline
- Independent builds/deployments
- Triggered by changes to that service

**2. Shared Pipeline Template:**
- Common stages (build, test, deploy)
- Service-specific configuration
- Reusable across services

**3. Dependency Management:**
- Build order if services depend on each other
- Use artifacts/package repositories
- Version APIs for backward compatibility

**4. Testing Strategy:**
- Unit tests per service
- Integration tests with dependencies
- Contract testing (Pact)
- End-to-end tests

**5. Deployment Strategy:**
- Canary deployments per service
- Feature flags
- Database migrations handled separately
- Rollback per service

**Example Structure:**
```
service-a/
  ├── .gitlab-ci.yml
  └── src/

service-b/
  ├── .gitlab-ci.yml
  └── src/

shared/
  └── pipeline-template.yml
```

**Real-world Context:** 10 microservices. Each has own pipeline. Shared template for common steps. Services deploy independently. Integration tests verify compatibility.

**Follow-up:** How do you handle database migrations in microservices? (Separate migration pipeline, versioned migrations, backward compatible changes)

---

### Q15: Explain blue-green and canary deployment strategies in CI/CD.

**Difficulty:** Senior

**Answer:**

**Blue-Green Deployment:**
- Two identical environments (blue = current, green = new)
- Deploy new version to green
- Switch traffic from blue to green
- Keep blue as rollback option

**Benefits:**
- Instant rollback (switch back to blue)
- Zero downtime
- Easy to test green before switching

**CI/CD Integration:**
```yaml
deploy_green:
  script:
    - kubectl apply -f k8s/green/
    - kubectl rollout status deployment/green-app
    - ./smoke-tests.sh green

switch_traffic:
  script:
    - kubectl patch service app -p '{"spec":{"selector":{"version":"green"}}}'
```

**Canary Deployment:**
- Deploy new version to small subset (e.g., 10%)
- Monitor metrics (errors, latency)
- Gradually increase traffic (10% → 50% → 100%)
- Rollback if issues detected

**Benefits:**
- Test in production with real traffic
- Gradual risk reduction
- Automatic rollback based on metrics

**CI/CD Integration:**
```yaml
deploy_canary:
  script:
    - kubectl set image deployment/app app=myapp:v2
    - kubectl scale deployment/app --replicas=1  # 10% traffic
    - sleep 300  # Monitor
    - kubectl scale deployment/app --replicas=5  # 50% traffic
    - sleep 300
    - kubectl scale deployment/app --replicas=10  # 100%
```

**Real-world Context:** Blue-green: Switch entire traffic instantly, instant rollback. Canary: Gradual rollout, catch issues early, automatic rollback.

**Follow-up:** When would you use blue-green vs canary? (Blue-green: simple apps, instant switch. Canary: complex apps, gradual risk reduction)

---

### Q16: How do you implement infrastructure testing in CI/CD?

**Difficulty:** Senior

**Answer:**

**Infrastructure Testing:**
- Validate infrastructure code (Terraform, CloudFormation)
- Test infrastructure changes before applying
- Ensure compliance and security

**Testing Types:**

**1. Syntax/Validation:**
```yaml
test_terraform:
  script:
    - terraform init
    - terraform validate
    - terraform fmt -check
```

**2. Plan Review:**
```yaml
terraform_plan:
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

review_plan:
  script:
    - terraform show tfplan
  when: manual
```

**3. Security Scanning:**
- Use tools: Checkov, Terrascan, tfsec
- Scan for security misconfigurations
- Fail pipeline on high-severity issues

**4. Cost Estimation:**
- Use Infracost for Terraform
- Estimate cost changes
- Alert on significant increases

**5. Compliance Testing:**
- Use Open Policy Agent (OPA)
- Enforce policies (tagging, naming)
- Fail on policy violations

**Example:**
```yaml
test_infrastructure:
  script:
    - terraform init
    - terraform validate
    - checkov -d .
    - infracost breakdown --path .
```

**Real-world Context:** Terraform changes → validate → security scan → cost estimate → plan review → apply. Fail pipeline on security issues.

**Follow-up:** How do you test infrastructure changes in staging before production? (Apply to staging first, run tests, then promote to prod)

---

## Best Practices

### Q17: What are CI/CD best practices?

**Difficulty:** Senior

**Answer:**

**Pipeline Design:**
- Keep pipelines fast (cache dependencies)
- Fail fast (run quick tests first)
- Parallelize where possible
- Use pipeline templates for consistency

**Security:**
- Never commit secrets (use secret management)
- Scan for vulnerabilities (SAST, DAST)
- Use least privilege for deployment credentials
- Sign artifacts and verify signatures

**Testing:**
- Test in production-like environment
- Use test data management
- Run tests in parallel
- Maintain high test coverage

**Deployment:**
- Use infrastructure as code
- Version everything (code, config, infrastructure)
- Implement rollback strategy
- Monitor deployments

**Monitoring:**
- Track pipeline metrics (duration, success rate)
- Alert on failures
- Log everything
- Use deployment dashboards

**Code Quality:**
- Lint and format code
- Run static analysis
- Enforce code review
- Use branch protection

**Real-world Context:** Fast pipelines (< 10 min), security scanning, parallel tests, canary deployments, monitoring, rollback capability.

**Follow-up:** How do you optimize slow pipelines? (Cache dependencies, parallelize jobs, run only relevant tests, use faster runners)

---

### Q18: How do you handle database migrations in CI/CD?

**Difficulty:** Senior

**Answer:**

**Challenges:**
- Database changes are stateful
- Need backward compatibility
- Rollback is complex
- Production data at risk

**Strategies:**

**1. Separate Migration Pipeline:**
- Run migrations before application deployment
- Can be manual or automated
- Test migrations in staging first

**2. Versioned Migrations:**
- Each migration has version number
- Track applied migrations
- Idempotent migrations (can run multiple times)

**3. Backward Compatible Changes:**
- Add columns as nullable first
- Deploy application that handles both
- Make columns required in next release
- Remove old columns later

**4. Blue-Green with Database:**
- Migrate database before switching
- Or use database replication
- Switch application, then migrate

**5. Feature Flags:**
- Deploy code with feature flags
- Migrate database
- Enable feature flag
- Remove old code later

**Example:**
```yaml
migrate_db:
  script:
    - flyway migrate
  environment: staging
  when: manual

deploy_app:
  script:
    - kubectl apply -f k8s/
  needs:
    - migrate_db
```

**Real-world Context:** Schema change: Add nullable column → Deploy app that handles both → Make required → Remove old code. Each step in separate deployment.

**Follow-up:** How do you rollback a database migration? (Create rollback migration, or restore from backup. Prefer backward compatible changes)

---

### Q19: Explain CI/CD for containerized applications.

**Difficulty:** Mid

**Answer:**

**Pipeline Stages:**

**1. Build:**
- Build application
- Run tests
- Create artifacts

**2. Build Docker Image:**
- Create Dockerfile
- Build image with tag (commit SHA, branch)
- Scan image for vulnerabilities

**3. Push to Registry:**
- Push to container registry (Docker Hub, ECR, GCR)
- Tag appropriately (latest, version, branch)

**4. Deploy:**
- Update Kubernetes manifests
- Deploy to cluster
- Health checks

**Example:**
```yaml
build:
  script:
    - npm install
    - npm test
    - npm run build

build_image:
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker tag myapp:$CI_COMMIT_SHA myapp:latest
    - docker push myapp:$CI_COMMIT_SHA
    - docker push myapp:latest

deploy:
  script:
    - kubectl set image deployment/app app=myapp:$CI_COMMIT_SHA
    - kubectl rollout status deployment/app
```

**Best Practices:**
- Use multi-stage builds (smaller images)
- Tag images with commit SHA
- Scan images for vulnerabilities
- Use image signing
- Cache Docker layers

**Real-world Context:** Build app → test → build Docker image → push to ECR → update K8s deployment → verify health.

**Follow-up:** How do you optimize Docker builds in CI/CD? (Multi-stage builds, layer caching, build args, use BuildKit)

---

### Q20: How do you implement security scanning in CI/CD pipelines?

**Difficulty:** Senior

**Answer:**

**Security Scanning Types:**

**1. SAST (Static Application Security Testing):**
- Scan source code for vulnerabilities
- Tools: SonarQube, Checkmarx, Snyk
- Run early in pipeline

**2. Dependency Scanning:**
- Scan dependencies for known vulnerabilities
- Tools: Snyk, OWASP Dependency-Check, npm audit
- Update dependencies regularly

**3. Container Scanning:**
- Scan Docker images for vulnerabilities
- Tools: Trivy, Clair, Snyk
- Before pushing to registry

**4. Infrastructure Scanning:**
- Scan IaC (Terraform, CloudFormation)
- Tools: Checkov, Terrascan, tfsec
- Before applying infrastructure

**5. Secrets Scanning:**
- Detect secrets in code
- Tools: GitGuardian, TruffleHog, git-secrets
- Prevent committing secrets

**Example Pipeline:**
```yaml
sast:
  script:
    - sonar-scanner

dependency_scan:
  script:
    - snyk test

container_scan:
  script:
    - trivy image myapp:$CI_COMMIT_SHA

infrastructure_scan:
  script:
    - checkov -d terraform/

secrets_scan:
  script:
    - git-secrets --scan
```

**Best Practices:**
- Fail pipeline on high-severity issues
- Review and fix issues promptly
- Keep scanning tools updated
- Use multiple tools (defense in depth)
- Integrate with security team

**Real-world Context:** Pipeline runs: SAST → Dependency scan → Build → Container scan → Deploy. Fail on high-severity vulnerabilities.

**Follow-up:** How do you handle false positives in security scans? (Tune tool rules, suppress known false positives, review with security team)

---

## Summary

CI/CD is essential for modern software delivery. Master pipeline design, security, testing, and deployment strategies. Practice with different tools and understand when to use each.

**Next Steps:**
- Set up CI/CD for a sample project
- Practice with Jenkins, GitLab CI, GitHub Actions
- Learn deployment strategies (blue-green, canary)
- Study security best practices
