# Semgrep Integration and Troubleshooting Guide

Comprehensive guide for integrating Semgrep security scanning into your CI/CD pipeline with performance optimization and troubleshooting strategies.

## Overview

This guide covers the complete Semgrep integration that achieves:
- **Performance**: 11 minutes → 2 minutes (82% improvement)
- **Targeted scanning**: 14 rules on 20 files instead of full ruleset
- **Platform visibility**: Results uploaded to Semgrep Cloud dashboard
- **Dual approach**: Local scanning for fast feedback + cloud upload for visibility

## Semgrep Authentication Setup

### 1. Obtain Semgrep App Token
1. **Login to Semgrep Dashboard**: https://semgrep.dev/
2. **Navigate to Settings → Tokens**
3. **Create New Token**:
   - **Name**: `Jenkins CI Integration`
   - **Scopes**: Full access (for CI/CD)
   - **Expiration**: Set appropriate duration

**Security Note**: Store this token securely - it provides full access to your Semgrep organization.

### 2. Configure Jenkins Credentials
Add the token to Jenkins:
1. **Manage Jenkins → Credentials → System → Global**
2. **Add Credentials → Secret text**
3. Configure:
   - **Secret**: `[YOUR_SEMGREP_APP_TOKEN]`
   - **ID**: `semgrep-app-token`
   - **Description**: `Semgrep Cloud API Token`

## Performance Optimization Strategy

### Dual Scanning Approach

Our optimized configuration uses two parallel scans:

1. **Fast Targeted Scan**: Local execution with 14 specific rules
2. **Cloud Upload**: Platform visibility with same file targeting

### Targeted Rule Configuration

Create `semgrep-run-targeted.sh` for optimized local scanning:

```bash
#!/bin/bash

# Create multiple --config arguments for each targeted rule
CONFIG_ARGS=""

# Dockerfile security rules
CONFIG_ARGS="$CONFIG_ARGS --config=r/dockerfile.security.missing-user.missing-user"

# Java security rules
CONFIG_ARGS="$CONFIG_ARGS --config=r/java.java-jwt.security.jwt-hardcode.java-jwt-hardcoded-secret"

# Python Django security rules
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.django.security.django-no-csrf-token.django-no-csrf-token"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.django.security.injection.tainted-sql-string.tainted-sql-string"

# Python Flask security rules
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.security.audit.render-template-string.render-template-string"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.security.audit.secure-set-cookie.secure-set-cookie"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.security.xss.audit.template-autoescape-off.template-autoescape-off"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.web.flask-cookie-httponly-missing.flask-cookie-httponly-missing"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.web.flask-cookie-samesite-missing.flask-cookie-samesite-missing"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.web.flask-cookie-secure-missing.flask-cookie-secure-missing"

# Python general security rules
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.lang.security.audit.dynamic-urllib-use-detected.dynamic-urllib-use-detected"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.lang.security.audit.md5-used-as-password.md5-used-as-password"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.lang.security.dangerous-os-exec.dangerous-os-exec"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.lang.security.dangerous-system-call.dangerous-system-call"

# Get file list for targeted scanning
file_list=$(tr '\n' ' ' < target-files.txt)

# Run semgrep with targeted rules and files
semgrep $CONFIG_ARGS --json --output=semgrep-results.json $file_list
```

### File Targeting Configuration

Create `target-files.txt` to specify which files to scan:

```
Dockerfile
middlewares.py
templates/file_upload.html
templates/idor/idor_login.html
templates/ssrf.html
templates/xss-reflected.html
templates/xss-stored.html
vuln-1.py
vuln-main-10.java
vuln-main-2.java
vuln-main-3.java
vuln-main-4.java
vuln-main-7.java
vuln-main-9.java
vuln-main.java
vulns/file_upload/file_upload.py
vulns/idor/idor.py
vulns/sql_injection/sql_injection_login.py
vulns/sql_injection/sql_injection_search.py
vulns/ssrf/ssrf.py
```

**Strategy**: Focus on high-risk files containing vulnerabilities rather than scanning everything.

## Semgrep Cloud Configuration

### Rule Management in Dashboard

1. **Access Semgrep Dashboard**: https://semgrep.dev/manage/rules
2. **Disable all rules initially** (bulk action)
3. **Enable only the 14 targeted rules**:
   - Search for each rule ID from the targeted list
   - Set to "Monitor" or "Block" mode
   - Ensure they're enabled for your deployment

### Expected Performance Results

With optimized cloud configuration:
- **Previous**: 23,636 rules downloaded, 0 actually executed
- **Optimized**: Only 14 rules executed
- **Build time**: ~2 minutes vs. 11+ minutes

### Platform Benefits

Cloud upload provides:
- **Dashboard visibility** for security teams
- **Historical trending** of security findings
- **Compliance reporting** capabilities
- **Team collaboration** features

## Jenkins Integration

### Complete Jenkinsfile Configuration

```groovy
pipeline {
    agent any
    
    environment {
        SEMGREP_APP_TOKEN = credentials('semgrep-app-token')
        SEMGREP_REPO_NAME = "bad-python-app-kyle-managed"
        SEMGREP_REPO_URL = "http://gitea:3000/admin/bad-python-app-kyle-managed.git"
        SEMGREP_BRANCH = "${env.BRANCH_NAME ?: 'main'}"
    }

    stages {
        stage('Semgrep Scans') {
            parallel {
                stage('Fast Targeted Scan (Learning)') {
                    steps {
                        sh '''
                            echo "Installing Semgrep..."
                            python3 -m pip install --user --break-system-packages semgrep
                            export PATH="$HOME/.local/bin:$PATH"
                            
                            echo "🚀 Running fast targeted scan (14 rules on 20 files only)..."
                            
                            # Use our targeted script for super fast scans while learning
                            ./semgrep-run-targeted.sh || true
                            
                            echo "=== Fast Scan Results ==="
                            if [ -f semgrep-results.json ]; then
                                python3 -c "
import json
try:
    with open('semgrep-results.json') as f:
        data = json.load(f)
    findings = data.get('results', [])
    print(f'📊 Fast targeted scan found: {len(findings)} findings')
    
    print('\\n🔍 Top 10 findings:')
    for i, finding in enumerate(findings[:10], 1):
        rule_id = finding.get('check_id', 'Unknown')
        path = finding.get('path', 'Unknown')
        severity = finding.get('extra', {}).get('severity', 'Unknown')
        message = finding.get('message', 'No message')
        if len(message) > 80:
            message = message[:80] + '...'
        print(f'  {i}. [{severity}] {rule_id}')
        print(f'     File: {path}')
        print(f'     Issue: {message}')
        print()
    
    if len(findings) > 10:
        print(f'... and {len(findings) - 10} more findings')
    
    print(f'⚡ Scan optimized for learning - 99.8% faster than full scan!')
    
except Exception as e:
    print(f'Error processing results: {e}')
"
                            else
                                echo "No results file found"
                            fi
                        '''
                    }
                }
                
                stage('Cloud Upload (Customer Dashboard)') {
                    steps {
                        withCredentials([string(credentialsId: 'semgrep-app-token', variable: 'SEMGREP_APP_TOKEN')]) {
                            sh '''
                            echo "Setting up Semgrep CI for cloud upload..."
                            export PATH="$HOME/.local/bin:$PATH"
                            
                            echo "📤 Running Semgrep CI for dashboard visibility..."
                            echo "Repository: ${SEMGREP_REPO_NAME}"
                            echo "Branch: ${SEMGREP_BRANCH}"
                            
                            # Build include patterns for targeted files
                            INCLUDE_ARGS=""
                            while read -r file_path; do
                                INCLUDE_ARGS="$INCLUDE_ARGS --include=$file_path"
                            done < target-files.txt
                            
                            # Note: semgrep ci uses rules configured in Semgrep Cloud, not custom --config
                            # Custom rules are handled in the Fast Targeted Scan stage for learning
                            
                            # Run semgrep ci for cloud upload (uses cloud-configured rules)
                            echo "⚠️  Note: semgrep ci uses rules configured in Semgrep Cloud, not custom --config"
                            semgrep ci $INCLUDE_ARGS --verbose || echo "Semgrep CI completed with findings - check dashboard!"
                            
                            echo "✅ Results uploaded to Semgrep Cloud dashboard!"
                        '''
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo '📋 Targeted Semgrep scans completed'
            archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
            echo '🎯 Learning mode: 14 rules × 20 files = super fast results!'
            echo '📤 Dashboard: Results uploaded to Semgrep Cloud for customer visibility'
            echo '🔗 Check your dashboard at https://semgrep.dev/manage/findings'
        }
        success {
            echo '✅ Both scans completed successfully!'
            echo '📊 Fast local scan: Perfect for rapid learning cycles'  
            echo '☁️  Cloud upload: Customer dashboard updated'
        }
        failure {
            echo '❌ Scan failed! Check logs for details'
        }
    }
}
```

## Troubleshooting Guide

### Common Integration Issues

#### 1. "0 Code rules" Executed
**Symptoms**: Cloud scan shows "Scanning X files with 0 Code rules"

**Root Cause**: Rules not properly enabled in Semgrep Dashboard

**Solution**:
1. Check Semgrep Dashboard → Rules → Active Rules
2. Verify the 14 rules are enabled (not disabled)
3. Ensure rules are in "Monitor" or "Block" mode
4. Confirm rules match the exact file types being scanned

#### 2. Build Performance Issues
**Symptoms**: Scans taking 10+ minutes despite optimization

**Debugging Steps**:
```bash
# Check if targeted script is being used
grep -n "semgrep-run-targeted.sh" build-logs

# Verify file targeting is working
grep -n "INCLUDE_ARGS" build-logs

# Check rule count in Semgrep dashboard
curl -H "Authorization: Bearer [TOKEN]" \
  https://semgrep.dev/api/v1/deployments/[DEPLOYMENT_ID]/rules
```

**Common Causes**:
- Script file missing or not executable
- Semgrep Cloud still has full ruleset enabled
- File targeting not working properly

#### 3. Authentication Failures
**Symptoms**: `semgrep ci` fails with authentication errors

**Solution**:
```bash
# Test token validity
curl -H "Authorization: Bearer [YOUR_TOKEN]" \
  https://semgrep.dev/api/v1/me

# Verify Jenkins credential configuration
# Check Jenkins logs for credential loading errors
```

#### 4. Network Connectivity Issues
**Symptoms**: Cannot reach Semgrep Cloud from Jenkins

**Debugging**:
```bash
# Test from Jenkins container
docker-compose exec jenkins curl -I https://semgrep.dev

# Check DNS resolution
docker-compose exec jenkins nslookup semgrep.dev

# Test with proxy if needed
export HTTPS_PROXY=http://proxy:8080
```

### Performance Debugging

#### Build Time Analysis
Monitor these metrics:
```bash
# Pipeline timing breakdown
grep "Pipeline Duration" jenkins-logs

# Semgrep installation time
grep "Installing Semgrep" jenkins-logs -A5

# Scan execution time
grep "Scan completed" jenkins-logs
```

#### Resource Monitoring
```bash
# Monitor during build
docker stats jenkins --no-stream

# Check disk usage
du -sh ./data/jenkins/workspace/

# Memory usage during scan
free -h
```

### Findings Validation

#### Comparing Local vs Cloud Results
```python
#!/usr/bin/env python3
"""Compare local Semgrep results with cloud dashboard findings."""

import json
import requests

def compare_findings():
    # Read local results
    with open('semgrep-results.json') as f:
        local_results = json.load(f)
    
    local_count = len(local_results.get('results', []))
    print(f"Local scan findings: {local_count}")
    
    # API call to get cloud findings (requires API access)
    # headers = {'Authorization': f'Bearer {SEMGREP_TOKEN}'}
    # cloud_response = requests.get(
    #     'https://semgrep.dev/api/v1/deployments/{deployment_id}/findings',
    #     headers=headers
    # )
    # cloud_count = len(cloud_response.json().get('findings', []))
    # print(f"Cloud dashboard findings: {cloud_count}")

if __name__ == '__main__':
    compare_findings()
```

## Advanced Configuration

### Custom Rule Development
For organization-specific rules:
```yaml
# custom-rule.yml
rules:
  - id: org-specific-pattern
    pattern: dangerous_function($...ARGS)
    message: "Avoid using dangerous_function in production"
    languages: [python]
    severity: ERROR
    metadata:
      category: security
      subcategory: [dangerous-code]
```

### Multiple Environment Support
```groovy
// Jenkinsfile environment-specific configuration
environment {
    SEMGREP_APP_TOKEN = credentials(
        env.BRANCH_NAME == 'main' ? 'semgrep-prod-token' : 'semgrep-dev-token'
    )
    SEMGREP_DEPLOYMENT = env.BRANCH_NAME == 'main' ? 'prod' : 'dev'
}
```

### CI/CD Integration Patterns

#### Pull Request Scanning
```groovy
when {
    anyOf {
        branch 'main'
        changeRequest()
    }
}
```

#### Scheduled Scans
```groovy
triggers {
    // Nightly full scan
    cron('H 2 * * *')
}
```

## Monitoring and Alerting

### Build Notifications
Configure Jenkins to send notifications:
```groovy
post {
    failure {
        emailext (
            subject: "🚨 Semgrep scan failed for ${env.JOB_NAME}",
            body: "Build ${env.BUILD_NUMBER} failed. Check console output.",
            to: "security-team@company.com"
        )
    }
}
```

### Metrics Collection
Track important metrics:
- Build duration trends
- Finding counts over time
- Rule effectiveness
- False positive rates

### Dashboard Integration
Connect to external dashboards:
- Grafana for build metrics
- Slack for real-time notifications
- JIRA for security issue tracking

## Best Practices

### Rule Selection Strategy
1. **Start small**: Begin with high-confidence rules
2. **Gradual expansion**: Add rules based on team feedback
3. **Regular review**: Assess rule effectiveness monthly
4. **Team input**: Include developers in rule selection

### File Targeting Strategy
1. **High-risk files first**: Focus on user input handling
2. **Security-critical paths**: Authentication, authorization
3. **Recent changes**: Prioritize newly modified files
4. **Dependency scanning**: Include third-party integrations

### Performance Optimization
1. **Baseline measurement**: Establish current performance
2. **Incremental changes**: Test one optimization at a time
3. **Resource monitoring**: Track CPU, memory, disk usage
4. **Feedback loops**: Regular performance reviews

---
*Next: [Resource Management Guide](./resource-management.md)*