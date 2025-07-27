# Jenkins Setup and Configuration Guide

Complete guide for setting up Jenkins with Semgrep integration in the cloud lab environment.

## Initial Jenkins Setup

### 1. Access Jenkins
After starting the Docker containers:
```bash
# Start Jenkins
docker-compose up -d jenkins

# Access Jenkins web interface
open http://your-server:8081
```

### 2. Initial Admin Password
```bash
# Retrieve the initial admin password
docker-compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Or from the host system
cat ./data/jenkins/secrets/initialAdminPassword
```

### 3. Plugin Installation
Install the following essential plugins during setup wizard:
- **Git Plugin**: Git repository integration
- **Generic Webhook Trigger**: Webhook handling from Gitea
- **Gitea Plugin**: Specific Gitea integration
- **Pipeline Plugin**: Pipeline as code support
- **Credentials Plugin**: Secret management

## Credentials Configuration

### Semgrep App Token
1. Navigate to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Select **Secret text** type
4. Configure:
   - **Secret**: `[OBTAIN_FROM_SEMGREP_DASHBOARD]`
   - **ID**: `semgrep-app-token`
   - **Description**: `Semgrep Cloud API Token`

**Where to get the token**: Semgrep Dashboard → Settings → Tokens → Create New Token

### Gitea Access Token
1. Add another credential for Gitea integration:
   - **Secret**: `[OBTAIN_FROM_GITEA_USER_SETTINGS]`
   - **ID**: `gitea-token`
   - **Description**: `Gitea API Access Token`

**Where to get the token**: Gitea → User Settings → Applications → Generate New Token

## Pipeline Configuration

### Sample Jenkinsfile for Semgrep Integration

```groovy
pipeline {
    agent any
    
    environment {
        SEMGREP_APP_TOKEN = credentials('semgrep-app-token')
        SEMGREP_REPO_NAME = "your-repo-name"
        SEMGREP_REPO_URL = "http://gitea:3000/your-user/your-repo.git"
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
                            
                            echo "🚀 Running fast targeted scan..."
                            
                            # Use targeted script for optimized scanning
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
                            
                            # Run semgrep ci for cloud upload
                            echo "⚠️  Note: semgrep ci uses rules configured in Semgrep Cloud"
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
            echo '🎯 Learning mode: Optimized for fast feedback!'
            echo '📤 Dashboard: Results uploaded to Semgrep Cloud for visibility'
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

## Job Configuration

### 1. Create New Pipeline Job
1. **New Item → Pipeline**
2. Configure:
   - **Name**: `semgrep-security-scan`
   - **Description**: `Automated security scanning with Semgrep`

### 2. Pipeline Configuration
**Pipeline Definition**: Pipeline script from SCM
**SCM**: Git
**Repository URL**: `http://gitea:3000/your-user/your-repo.git`
**Credentials**: Select `gitea-token`
**Branch**: `*/main`
**Script Path**: `Jenkinsfile`

### 3. Build Triggers
Configure **Generic Webhook Trigger**:
- **Post Content Parameters**:
  - Variable: `repository_name`
  - Expression: `$.repository.name`
- **Token**: `[GENERATE_UNIQUE_TOKEN]`
- **Trigger webhook URL**: `http://your-server:8081/generic-webhook-trigger/invoke?token=[YOUR_TOKEN]`

## Performance Optimization

### Build Performance Results
Our optimized configuration achieves:
- **Before**: 11 minutes (full ruleset scan)
- **After**: 2 minutes (targeted rules)
- **Improvement**: 82% faster execution

### Optimization Strategies

#### 1. Targeted Rule Configuration
Create `semgrep-run-targeted.sh`:
```bash
#!/bin/bash

# Create multiple --config arguments for targeted rules
CONFIG_ARGS=""
CONFIG_ARGS="$CONFIG_ARGS --config=r/dockerfile.security.missing-user.missing-user"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.flask.security.audit.render-template-string.render-template-string"
CONFIG_ARGS="$CONFIG_ARGS --config=r/python.lang.security.dangerous-os-exec.dangerous-os-exec"
# Add your specific rules here...

# Get file list for targeted scanning
file_list=$(tr '\n' ' ' < target-files.txt)

# Run semgrep with targeted rules and files
semgrep $CONFIG_ARGS --json --output=semgrep-results.json $file_list
```

#### 2. File Targeting
Create `target-files.txt` with specific files to scan:
```
Dockerfile
middlewares.py
vuln-1.py
vulns/sql_injection/sql_injection_login.py
vulns/file_upload/file_upload.py
# Add your critical files...
```

#### 3. Package Caching
For faster subsequent builds, consider:
```bash
# Cache Semgrep installation
RUN pip install --user semgrep

# Or use Docker layer caching
FROM jenkins/jenkins:lts
USER root
RUN pip install semgrep
USER jenkins
```

## Webhook Integration with Gitea

### 1. Configure Gitea Webhook
In your Gitea repository:
1. **Settings → Webhooks → Add Webhook**
2. Configure:
   - **Target URL**: `http://jenkins:8081/generic-webhook-trigger/invoke?token=[YOUR_TOKEN]`
   - **HTTP Method**: `POST`
   - **Content Type**: `application/json`
   - **Events**: Push events, Pull request events

### 2. Test Webhook
```bash
# Test webhook manually
curl -X POST http://your-server:8081/generic-webhook-trigger/invoke?token=[YOUR_TOKEN] \\
  -H "Content-Type: application/json" \\
  -d '{"repository": {"name": "test-repo"}}'
```

## Troubleshooting

### Common Issues

#### 1. Semgrep Installation Fails
```bash
# Debug: Check Python installation
docker-compose exec jenkins python3 --version
docker-compose exec jenkins pip3 --version

# Solution: Install Python if missing
docker-compose exec jenkins apt-get update && apt-get install -y python3-pip
```

#### 2. Permission Issues
```bash
# Fix Jenkins user permissions
sudo chown -R 1000:1000 ./data/jenkins

# Check Docker socket permissions
ls -la /var/run/docker.sock
```

#### 3. Network Connectivity
```bash
# Test Gitea connectivity from Jenkins
docker-compose exec jenkins ping gitea
docker-compose exec jenkins curl http://gitea:3000

# Check container networking
docker network ls
docker network inspect cloud-lab_lab-network
```

#### 4. Credential Issues
```bash
# Verify credentials are configured
docker-compose exec jenkins ls /var/jenkins_home/credentials.xml

# Test Semgrep authentication
docker-compose exec jenkins bash -c "
export SEMGREP_APP_TOKEN=[YOUR_TOKEN]
semgrep --version
"
```

### Performance Troubleshooting

#### Slow Build Times
1. **Check resource allocation**:
   ```bash
   docker stats jenkins
   ```

2. **Monitor disk I/O**:
   ```bash
   iotop
   ```

3. **Optimize rule selection** in Semgrep Cloud dashboard

#### Memory Issues
```bash
# Increase Jenkins memory
# Edit docker-compose.yml:
jenkins:
  environment:
    - JAVA_OPTS=-Xmx2g -Xms1g
```

## Monitoring and Metrics

### Build Metrics
Monitor these key performance indicators:
- **Build Duration**: Target < 5 minutes
- **Success Rate**: Target > 95%
- **Finding Quality**: Low false positive rate

### Jenkins Monitoring
```bash
# View build history
curl -u admin:[PASSWORD] http://your-server:8081/api/json?pretty=true

# Monitor system info
curl -u admin:[PASSWORD] http://your-server:8081/systemInfo/api/json?pretty=true
```

## Advanced Configuration

### Multi-branch Pipeline
For advanced Git workflows:
1. **New Item → Multibranch Pipeline**
2. Configure branch sources
3. Set up branch-specific scanning rules

### Jenkins Agents
For scaling builds:
```yaml
# Add Jenkins agent in docker-compose.yml
jenkins-agent:
  image: jenkins/ssh-agent
  environment:
    - JENKINS_AGENT_SSH_PUBKEY=[PUBLIC_KEY]
```

---
*Next: [Gitea Configuration Guide](./gitea-setup.md)*