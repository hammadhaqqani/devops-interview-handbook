# Architecture Exercise: Design CI/CD Pipeline

## Problem Statement

Design a comprehensive CI/CD pipeline for a microservices architecture with 20+ services, multiple environments (dev, staging, prod), and strict security and compliance requirements.

## Requirements

### Functional Requirements

1. **Build and Test**
   - Build Docker images for each service
   - Run unit tests
   - Run integration tests
   - Run security scans
   - Generate test reports

2. **Deployment**
   - Deploy to dev automatically
   - Deploy to staging after approval
   - Deploy to production with manual approval
   - Support blue-green and canary deployments
   - Rollback capability

3. **Quality Gates**
   - Code quality checks (linting, formatting)
   - Test coverage thresholds
   - Security vulnerability scanning
   - Infrastructure validation
   - Performance testing (staging)

### Non-Functional Requirements

1. **Speed**: Pipeline should complete in < 15 minutes for typical changes
2. **Reliability**: < 1% pipeline failure rate
3. **Security**: No secrets in code, signed artifacts, audit logs
4. **Scalability**: Support 100+ commits per day
5. **Cost**: Optimize for cost-effectiveness
6. **Compliance**: SOC 2, PCI DSS (if handling payments)

## Constraints and Assumptions

### Constraints

- **Platform**: GitLab CI (or GitHub Actions, Jenkins)
- **Infrastructure**: Kubernetes (EKS)
- **Container Registry**: AWS ECR
- **Compliance**: Must pass security scans before production

### Assumptions

- 20 microservices
- Average build time: 5 minutes per service
- Test execution: 10 minutes per service
- Deployment: 2 minutes per service
- Services deploy independently
- Not all services change in each commit

## Reference Architecture

### Pipeline Flow

```
Developer Push
     |
     v
[Source Control] GitLab/GitHub
     |
     v
[Trigger Pipeline]
     |
     ├──> [Lint & Format] (2 min)
     |         |
     |         v
     |    [Security Scan] (3 min)
     |         |
     |         v
     |    [Build Docker Image] (5 min)
     |         |
     |         v
     |    [Unit Tests] (5 min) ──┐
     |         |                 |
     |         v                 |
     |    [Integration Tests] (8 min) ──┐
     |         |                       |
     |         v                       |
     |    [Test Reports]              |
     |         |                       |
     |         v                       |
     |    [Quality Gates] ────────────┘
     |         |
     |         v
     |    [Push to ECR] (1 min)
     |         |
     |         v
     ├──> [Deploy to Dev] (Auto) (2 min)
     |         |
     |         v
     |    [Smoke Tests] (2 min)
     |         |
     |         v
     ├──> [Deploy to Staging] (Manual Approval) (2 min)
     |         |
     |         v
     |    [E2E Tests] (10 min)
     |         |
     |         v
     |    [Performance Tests] (5 min)
     |         |
     |         v
     └──> [Deploy to Prod] (Manual Approval) (2 min)
               |
               v
          [Post-Deploy Verification] (2 min)
```

### Component Breakdown

#### 1. Source Control and Triggers

**Component**: GitLab / GitHub

**Configuration**:
- Main branch protection
- Require merge requests
- Require approvals (2 for production)
- Branch naming conventions
- Commit message standards

**Pipeline Triggers**:
- Push to branch
- Merge request
- Tag creation (releases)
- Scheduled (nightly builds)
- Manual trigger

#### 2. Lint and Format Stage

**Purpose**: Code quality checks

**Tools**:
- ESLint (JavaScript/TypeScript)
- Prettier (formatting)
- Black (Python)
- gofmt (Go)

**Configuration**:
```yaml
lint:
  stage: validate
  script:
    - npm run lint
    - npm run format:check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request"
    - if: $CI_COMMIT_BRANCH == "main"
  allow_failure: false
```

**Quality Gates**:
- Must pass linting
- Code must be formatted
- Fail pipeline if violations

#### 3. Security Scanning Stage

**Purpose**: Detect vulnerabilities early

**Tools**:
- **SAST**: SonarQube, Snyk, Checkmarx
- **Dependency Scanning**: Snyk, OWASP Dependency-Check
- **Container Scanning**: Trivy, Clair, Snyk
- **Secrets Scanning**: GitGuardian, TruffleHog

**Configuration**:
```yaml
security:scan:
  stage: security
  script:
    - snyk test --severity-threshold=high
    - trivy fs --severity HIGH,CRITICAL .
    - git-secrets --scan
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request"
  allow_failure: false  # Fail on high/critical
```

**Quality Gates**:
- No high/critical vulnerabilities
- No secrets in code
- Fail pipeline on violations

#### 4. Build Stage

**Purpose**: Build Docker images

**Configuration**:
```yaml
build:
  stage: build
  script:
    - docker build
        --cache-from $CI_REGISTRY_IMAGE:latest
        --cache-from $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
        .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  only:
    changes:
      - "services/service-name/**/*"
  cache:
    key: docker-$CI_COMMIT_REF_SLUG
    paths:
      - .docker-cache/
```

**Optimizations**:
- Docker layer caching
- Multi-stage builds
- Build only changed services
- Parallel builds

#### 5. Test Stage

**Purpose**: Run automated tests

**Unit Tests**:
```yaml
test:unit:
  stage: test
  script:
    - npm ci --cache .npm --prefer-offline
    - npm run test:unit -- --coverage --reporter junit
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      junit: test-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/
```

**Integration Tests**:
```yaml
test:integration:
  stage: test
  services:
    - postgres:13
    - redis:6
  script:
    - npm ci
    - npm run test:integration
  only:
    - main
    - merge_requests
```

**Quality Gates**:
- Test coverage > 80%
- All tests must pass
- Fail pipeline on test failures

#### 6. Deploy to Dev

**Purpose**: Automatic deployment to development

**Configuration**:
```yaml
deploy:dev:
  stage: deploy
  environment:
    name: development
    url: https://dev.example.com
  script:
    - kubectl config use-context dev
    - kubectl set image deployment/$SERVICE_NAME \
        $SERVICE_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/$SERVICE_NAME
  only:
    - main
    - develop
  when: on_success
```

**Features**:
- Automatic on merge to main/develop
- No approval required
- Fast feedback

#### 7. Deploy to Staging

**Purpose**: Pre-production testing

**Configuration**:
```yaml
deploy:staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/$SERVICE_NAME \
        $SERVICE_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/$SERVICE_NAME
  when: manual
  only:
    - main
```

**Features**:
- Manual approval required
- E2E tests after deployment
- Performance testing
- UAT environment

#### 8. Deploy to Production

**Purpose**: Production deployment

**Configuration**:
```yaml
deploy:production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - kubectl config use-context production
    # Blue-green deployment
    - ./scripts/blue-green-deploy.sh $SERVICE_NAME $CI_COMMIT_SHA
    - kubectl rollout status deployment/$SERVICE_NAME-green
    # Switch traffic
    - kubectl patch service $SERVICE_NAME \
        -p '{"spec":{"selector":{"version":"green"}}}'
    # Verify
    - ./scripts/health-check.sh
    # Cleanup old deployment after 1 hour
    - sleep 3600
    - kubectl delete deployment $SERVICE_NAME-blue
  when: manual
  only:
    - main
    - tags
```

**Features**:
- Manual approval (2 approvers)
- Blue-green deployment
- Health checks
- Automatic rollback on failure
- Gradual traffic shift

#### 9. Post-Deployment Verification

**Purpose**: Verify deployment success

**Configuration**:
```yaml
verify:production:
  stage: verify
  script:
    - ./scripts/smoke-tests.sh
    - ./scripts/check-metrics.sh
    - ./scripts/verify-logs.sh
  when: on_success
  only:
    - main
```

**Checks**:
- Smoke tests
- Health endpoints
- Error rate monitoring
- Performance metrics

## Discussion Points

### Deployment Strategies

**1. Blue-Green Deployment**

**How it Works**:
- Deploy new version alongside old (green vs blue)
- Switch traffic instantly
- Keep old version for quick rollback

**Pros**:
- Zero downtime
- Instant rollback
- Easy to test new version

**Cons**:
- Requires 2x resources during deployment
- More complex

**2. Canary Deployment**

**How it Works**:
- Deploy new version to small subset (10%)
- Gradually increase traffic (10% → 50% → 100%)
- Monitor metrics, rollback if issues

**Pros**:
- Test in production with real traffic
- Gradual risk reduction
- Automatic rollback

**Cons**:
- More complex
- Requires traffic splitting

**3. Rolling Deployment**

**How it Works**:
- Gradually replace pods
- Default Kubernetes strategy

**Pros**:
- Simple
- No extra resources needed

**Cons**:
- Brief period with mixed versions
- Rollback takes time

### Security Considerations

**1. Secrets Management**

- Use secrets management service (AWS Secrets Manager, Vault)
- Never commit secrets
- Rotate secrets regularly
- Use IAM roles (no access keys)

**2. Artifact Signing**

- Sign Docker images
- Verify signatures before deployment
- Use Notary, Cosign

**3. Access Control**

- RBAC for pipeline access
- Require approvals for production
- Audit all deployments
- Limit who can approve

**4. Network Security**

- Deploy from CI/CD network only
- Use VPN for access
- Restrict kubectl access
- Network policies in Kubernetes

### Cost Optimization

**Strategies**:
- Cache dependencies (npm, pip, etc.)
- Docker layer caching
- Build only changed services
- Use spot instances for test runners
- Parallelize jobs
- Conditional execution (skip unnecessary jobs)

**Estimated Costs**:
- CI/CD platform: $500/month (GitLab Premium)
- Compute (runners): $1,000/month
- Container registry: $200/month
- Total: ~$1,700/month

### Scalability Considerations

**For 100+ Commits/Day**:
- Parallel pipelines (multiple runners)
- Build only changed services
- Efficient caching
- Queue management
- Resource allocation

**For 20+ Services**:
- Per-service pipelines
- Independent deployments
- Service-specific configurations
- Shared pipeline templates

### Monitoring and Observability

**Pipeline Metrics**:
- Duration per stage
- Success/failure rate
- Cache hit rate
- Resource usage

**Deployment Metrics**:
- Deployment frequency
- Lead time
- Mean time to recovery (MTTR)
- Change failure rate

**Tools**:
- GitLab/GitHub built-in metrics
- Prometheus + Grafana
- Custom dashboards

## Alternative Approaches

**1. GitOps (ArgoCD/Flux)**

- Declarative deployments
- Git as source of truth
- Automatic sync
- Better audit trail

**2. Spinnaker**

- Multi-cloud support
- Advanced deployment strategies
- Better for complex workflows

**3. Jenkins**

- More plugins
- More flexible
- Self-hosted option
- More complex

## Implementation Phases

### Phase 1: Basic Pipeline
- Build and test
- Deploy to dev
- Basic security scanning

### Phase 2: Quality Gates
- Test coverage requirements
- Security gates
- Code quality checks

### Phase 3: Multi-Environment
- Staging deployment
- Production deployment
- Approval workflows

### Phase 4: Advanced Features
- Blue-green/canary deployments
- Performance testing
- Advanced monitoring
- Cost optimization

## Key Takeaways

1. **Automation**: Automate everything possible
2. **Security**: Security scanning at every stage
3. **Quality Gates**: Fail fast on issues
4. **Speed**: Optimize for fast feedback
5. **Reliability**: Test thoroughly before production
6. **Monitoring**: Track metrics and improve
7. **Documentation**: Document processes and runbooks

## Follow-up Questions

1. How do you handle database migrations in CI/CD?
2. How do you test infrastructure changes?
3. How do you handle secrets in pipelines?
4. How do you implement feature flags?
5. How do you handle rollbacks?
