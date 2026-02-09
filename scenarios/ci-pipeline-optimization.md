# Scenario: CI Pipeline Optimization

## Problem Statement

Your CI/CD pipeline is taking too long (45+ minutes) and failing frequently. The team is frustrated with slow feedback, and deployments are delayed. You need to optimize the pipeline for speed and reliability.

## Environment

- **CI Platform**: GitLab CI
- **Application**: Node.js microservices (10 services)
- **Pipeline Stages**: Build → Test → Deploy
- **Current Duration**: 45 minutes average
- **Failure Rate**: ~15%

## Current Pipeline Analysis

### Current Pipeline Structure

```yaml
# .gitlab-ci.yml (simplified)
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
    - npm install  # Installing again!
    - npm test
    - npm run lint
    - npm run e2e

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
  only:
    - main
```

### Performance Issues Identified

1. **Redundant Operations:**
   - `npm install` runs in both build and test stages
   - No caching of dependencies
   - Building Docker images from scratch every time

2. **Sequential Execution:**
   - All tests run sequentially
   - No parallelization
   - Long-running E2E tests block pipeline

3. **Inefficient Testing:**
   - Running all tests on every commit
   - No test filtering
   - E2E tests run for every change

4. **Resource Constraints:**
   - Using shared runners (slow)
   - No resource allocation optimization
   - No parallel jobs

5. **No Artifact Optimization:**
   - Large artifacts transferred between stages
   - No artifact compression
   - Unnecessary files included

## Optimization Strategies

### Strategy 1: Implement Caching

**Problem:** Dependencies installed every time

**Solution:**

```yaml
# Cache node_modules
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

build:
  stage: build
  script:
    - npm ci  # Faster, uses package-lock.json
    - npm run build
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull-push

test:
  stage: test
  script:
    - npm ci  # Uses cache if available
    - npm test
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull
```

**Benefits:**
- Reduces install time from 5min to 30sec (cache hit)
- Saves bandwidth
- Faster pipeline execution

### Strategy 2: Parallelize Jobs

**Problem:** Tests run sequentially

**Solution:**

```yaml
test:unit:
  stage: test
  script:
    - npm ci
    - npm run test:unit
  parallel:
    matrix:
      - TEST_SUITE: [unit, integration, e2e]

# Or separate jobs
test:unit:
  stage: test
  script:
    - npm ci
    - npm run test:unit

test:integration:
  stage: test
  script:
    - npm ci
    - npm run test:integration

test:e2e:
  stage: test
  script:
    - npm ci
    - npm run test:e2e
  only:
    - main
    - merge_requests
```

**Benefits:**
- Tests run in parallel
- Reduces test stage from 20min to 8min (longest test suite)

### Strategy 3: Use Docker Layer Caching

**Problem:** Building Docker images from scratch

**Solution:**

```yaml
build:docker:
  stage: build
  script:
    - docker build
        --cache-from $CI_REGISTRY_IMAGE:latest
        --cache-from $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        -t $CI_REGISTRY_IMAGE:latest
        .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

**Benefits:**
- Docker build time: 15min → 3min (cache hit)
- Faster deployments

### Strategy 4: Conditional Execution

**Problem:** Running all tests for every change

**Solution:**

```yaml
test:unit:
  stage: test
  script:
    - npm ci
    - npm run test:unit
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request"
    - if: $CI_COMMIT_BRANCH == "main"
    - changes:
        - "src/**/*"
        - "*.js"
        - "package.json"

test:e2e:
  stage: test
  script:
    - npm ci
    - npm run test:e2e
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request" && $E2E_TESTS == "true"
  allow_failure: true  # Don't block on E2E failures initially
```

**Benefits:**
- Skip unnecessary tests
- Faster feedback for developers
- E2E only on main branch or when requested

### Strategy 5: Optimize Artifacts

**Problem:** Large artifacts slow down pipeline

**Solution:**

```yaml
build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    exclude:
      - node_modules/  # Don't include dependencies
    expire_in: 1 week
    reports:
      # Only include test results, not full output
      junit: test-results.xml
```

**Benefits:**
- Smaller artifacts
- Faster artifact transfer
- Less storage used

### Strategy 6: Use Dedicated Runners

**Problem:** Shared runners are slow

**Solution:**

```yaml
build:
  stage: build
  tags:
    - docker
    - linux
  script:
    - npm ci
    - npm run build
```

**Benefits:**
- More resources available
- Less contention
- Faster execution

### Strategy 7: Split Large Pipelines

**Problem:** Monolithic pipeline for all services

**Solution:**

```yaml
# Per-service pipelines
# Each service has own .gitlab-ci.yml
# Triggered by changes to that service

build:service-a:
  stage: build
  script:
    - cd services/service-a
    - npm ci
    - npm run build
  only:
    changes:
      - services/service-a/**/*
```

**Benefits:**
- Only build changed services
- Parallel service builds
- Faster overall pipeline

### Strategy 8: Implement Test Result Caching

**Problem:** Running same tests repeatedly

**Solution:**

```yaml
test:
  stage: test
  script:
    - npm ci
    - |
      if [ -f test-results-cache.json ]; then
        echo "Using cached test results"
      else
        npm test -- --reporter json > test-results.json
      fi
  cache:
    key: test-results-${CI_COMMIT_REF_SLUG}
    paths:
      - test-results-cache.json
```

## Optimized Pipeline Example

```yaml
stages:
  - build
  - test
  - deploy

variables:
  NPM_CONFIG_CACHE: .npm
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

build:
  stage: build
  image: node:18
  script:
    - npm ci --cache .npm --prefer-offline
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/
    policy: pull-push

test:unit:
  stage: test
  image: node:18
  script:
    - npm ci --cache .npm --prefer-offline
    - npm run test:unit -- --reporter junit --output-file test-results.xml
  artifacts:
    reports:
      junit: test-results.xml
    expire_in: 1 week
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull
  parallel: 2  # Run 2 parallel jobs

test:integration:
  stage: test
  image: node:18
  script:
    - npm ci --cache .npm --prefer-offline
    - npm run test:integration
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request"

test:e2e:
  stage: test
  image: cypress/browsers:latest
  script:
    - npm ci
    - npm run test:e2e
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request" && $E2E_TESTS == "true"
  allow_failure: true
  when: manual  # Manual trigger for E2E

build:docker:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build
        --cache-from $CI_REGISTRY_IMAGE:latest
        --cache-from $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        -t $CI_REGISTRY_IMAGE:latest
        .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  dependencies:
    - build
  only:
    - main
    - tags

deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/web-app web-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/web-app
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context production
    - kubectl set image deployment/web-app web-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/web-app
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

## Performance Improvements

### Before Optimization:
- **Total Time**: 45 minutes
- **Build**: 10 minutes
- **Test**: 25 minutes
- **Deploy**: 10 minutes
- **Failure Rate**: 15%

### After Optimization:
- **Total Time**: 12 minutes (73% reduction)
- **Build**: 3 minutes (cache + Docker layer caching)
- **Test**: 6 minutes (parallelization)
- **Deploy**: 3 minutes (optimized)
- **Failure Rate**: 5% (better error handling)

## Monitoring and Metrics

### Track Pipeline Metrics:

```yaml
# Add to pipeline
pipeline:metrics:
  stage: .post
  script:
    - echo "Pipeline duration: $CI_PIPELINE_DURATION"
    - echo "Job duration: $CI_JOB_DURATION"
  when: always
```

### Key Metrics to Monitor:
- Pipeline duration
- Job duration per stage
- Cache hit rate
- Failure rate
- Resource usage

## Best Practices Summary

1. **Cache Everything Possible**
   - Dependencies (npm, pip, etc.)
   - Docker layers
   - Build artifacts

2. **Parallelize Jobs**
   - Run independent jobs in parallel
   - Use matrix builds
   - Split test suites

3. **Conditional Execution**
   - Skip unnecessary jobs
   - Use rules/only/except wisely
   - Manual triggers for expensive operations

4. **Optimize Artifacts**
   - Exclude unnecessary files
   - Compress if needed
   - Set expiration

5. **Use Appropriate Runners**
   - Dedicated runners for critical jobs
   - Right-sized runners
   - Tag runners appropriately

6. **Fail Fast**
   - Run quick checks first
   - Expensive tests later
   - Allow failures for non-critical tests

## Follow-up Questions

1. How do you handle cache invalidation?
2. What's your strategy for flaky tests?
3. How do you optimize for cost vs speed?
4. How do you handle pipeline failures?

## Key Takeaways

- Caching is the biggest win (dependencies, Docker layers)
- Parallelization significantly reduces time
- Conditional execution prevents wasted work
- Monitor metrics to identify bottlenecks
- Optimize artifacts to reduce transfer time
- Use appropriate infrastructure (runners, resources)
- Balance speed, cost, and reliability
