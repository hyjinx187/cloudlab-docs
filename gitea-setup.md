# Gitea Setup and Configuration Guide

Complete guide for configuring Gitea as your self-hosted Git server with Jenkins webhook integration.

## Initial Gitea Setup

### 1. Access Gitea
After starting the Docker containers:
```bash
# Start Gitea
docker-compose up -d gitea

# Access Gitea web interface
open http://your-server:3000
```

### 2. Initial Configuration
On first access, you'll see the Gitea installation page:

#### Database Configuration
- **Database Type**: SQLite3 (default, simplest for lab use)
- **Path**: `/data/gitea/gitea.db` (automatically configured in container)

#### Basic Settings
- **Site Title**: `Cloud Lab Git Server`
- **Repository Root Path**: `/data/git/repositories`
- **Git LFS Root Path**: `/data/git/lfs`
- **Log Path**: `/data/gitea/log`

#### Server and Third-Party Service Settings
- **SSH Server Domain**: `your-server-ip`
- **SSH Port**: `22` (or `2222` if customized)
- **HTTP Port**: `3000`
- **Application URL**: `http://your-server:3000/`

#### Administrator Account
Create your admin account:
- **Username**: `admin` (recommended for lab consistency)
- **Password**: `[GENERATE_SECURE_PASSWORD]`
- **Email**: `admin@lab.local`

**Password Security**: Use `openssl rand -base64 16` to generate a secure password

### 3. Post-Installation Configuration

#### Email Settings (Optional)
For notifications and user management:
1. **Admin Panel → Configuration → Mailer**
2. Configure SMTP settings or use a service like SendGrid

#### Authentication Settings
For production use, consider:
- LDAP integration
- OAuth providers (GitHub, GitLab, etc.)
- Two-factor authentication

## Repository Management

### Creating Organizations
For better project organization:
1. **+ → New Organization**
2. Configure:
   - **Organization Name**: `semgrep-demos`
   - **Visibility**: Public or Private
   - **Description**: `Demo repositories for Semgrep integration`

### Creating Repositories

#### 1. Demo Repository Setup
Create a repository for Semgrep testing:
```bash
# Repository settings
Name: bad-python-app-kyle-managed
Description: Intentionally vulnerable Python app for Semgrep testing
Visibility: Public (for demo purposes)
Initialize: ✓ Add .gitignore (Python)
License: MIT
```

#### 2. Repository Structure
Your demo repository should include:
```
bad-python-app-kyle-managed/
├── Jenkinsfile                    # CI/CD pipeline definition
├── semgrep-run-targeted.sh        # Targeted scanning script
├── target-files.txt               # Files to scan
├── requirements.txt               # Python dependencies
├── app.py                         # Main application
├── vulns/                         # Intentional vulnerabilities
│   ├── sql_injection/
│   ├── file_upload/
│   └── xss/
└── templates/                     # Web templates
```

### Repository Configuration

#### Branch Protection
For production-like workflows:
1. **Repository → Settings → Branches**
2. Configure protection rules:
   - **Branch name pattern**: `main`
   - **Protect this branch**: ✓
   - **Require pull request reviews**: ✓
   - **Require status checks**: ✓ (Jenkins build)

#### Webhooks Configuration
Essential for Jenkins integration:
1. **Repository → Settings → Webhooks**
2. **Add Webhook → Gitea**
3. Configure:
   - **Target URL**: `http://jenkins:8081/generic-webhook-trigger/invoke?token=[WEBHOOK_TOKEN]`
   - **HTTP Method**: `POST`
   - **Content Type**: `application/json`
   - **Secret**: `[OPTIONAL_WEBHOOK_SECRET]`

**Webhook Token**: Generate using `uuidgen` or online UUID generator

#### Webhook Events
Select appropriate triggers:
- ✓ **Push**: Trigger builds on code changes
- ✓ **Pull Request**: Trigger builds on PR creation/updates
- ✓ **Wiki**: If documentation changes should trigger builds
- ✗ **Repository**: Avoid unnecessary triggers

### User Management

#### Creating Service Accounts
For Jenkins integration:
1. **Admin Panel → User Accounts → Create User Account**
2. Configure:
   - **Username**: `jenkins-service`
   - **Email**: `jenkins@lab.local`
   - **Password**: `[GENERATE_SERVICE_PASSWORD]`
   - **Account Type**: Regular User

#### API Token Generation
For automated access:
1. **User Settings → Applications**
2. **Generate New Token**
3. Configure:
   - **Token Name**: `Jenkins Integration`
   - **Scopes**: 
     - ✓ **repo**: Repository access
     - ✓ **write:repo_hook**: Webhook management
     - ✗ **admin**: Avoid unless necessary

**Store Token Securely**: Copy the token immediately - it won't be shown again

## Git Configuration

### Setting Up Git Remote
For local development:
```bash
# Clone repository
git clone http://your-server:3000/admin/bad-python-app-kyle-managed.git

# Configure user
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Add remote for pushing
git remote set-url origin http://jenkins-service:[TOKEN]@your-server:3000/admin/bad-python-app-kyle-managed.git
```

### SSH Key Setup (Alternative)
For SSH-based access:
```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "jenkins@lab.local"

# Add public key to Gitea
# User Settings → SSH/GPG Keys → Add Key
cat ~/.ssh/id_ed25519.pub

# Test SSH connection
ssh -T git@your-server
```

## Webhook Integration Testing

### Manual Webhook Testing
```bash
# Test webhook endpoint
curl -X POST http://your-server:8081/generic-webhook-trigger/invoke?token=[YOUR_TOKEN] \
  -H "Content-Type: application/json" \
  -H "X-Gitea-Event: push" \
  -d '{
    "repository": {
      "name": "bad-python-app-kyle-managed",
      "full_name": "admin/bad-python-app-kyle-managed",
      "clone_url": "http://your-server:3000/admin/bad-python-app-kyle-managed.git"
    },
    "pusher": {
      "login": "admin"
    },
    "ref": "refs/heads/main"
  }'
```

### Webhook Debugging
Check webhook delivery:
1. **Repository → Settings → Webhooks**
2. Click on your webhook
3. **Recent Deliveries** tab shows request/response details

Common webhook issues:
- **DNS resolution**: Use container names (`jenkins` not `localhost`)
- **Firewall**: Ensure internal network communication
- **Authentication**: Verify webhook tokens match

## File Management

### Large File Support (Git LFS)
For binary assets in demos:
```bash
# Install Git LFS
git lfs install

# Track binary files
git lfs track "*.jpg"
git lfs track "*.png"
git lfs track "*.pdf"

# Commit tracking file
git add .gitattributes
git commit -m "Add Git LFS tracking"
```

### Repository Templates
Create templates for consistent demo repos:
1. **Repository → Settings → Advanced**
2. **Template Repository**: ✓
3. Clone template for new demos:
   ```bash
   # Use template via API
   curl -X POST http://your-server:3000/api/v1/repos/admin/template-repo/generate \
     -H "Authorization: token [API_TOKEN]" \
     -H "Content-Type: application/json" \
     -d '{"name": "new-demo-repo", "owner": "admin"}'
   ```

## Backup and Maintenance

### Data Backup
Critical data locations:
```bash
# Gitea data directory
./data/gitea/
├── conf/           # Configuration files
├── gitea.db        # Database (if using SQLite)
├── git/            # Git repositories
└── log/            # Application logs
```

### Backup Script
```bash
#!/bin/bash
# backup-gitea.sh

BACKUP_DIR="/backups/gitea/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Stop Gitea temporarily
docker-compose stop gitea

# Copy data
cp -r ./data/gitea/ "$BACKUP_DIR/"

# Restart Gitea
docker-compose start gitea

echo "Backup completed: $BACKUP_DIR"
```

### Database Maintenance
For SQLite (default):
```bash
# Optimize database
docker-compose exec gitea sqlite3 /data/gitea/gitea.db 'VACUUM;'

# Check database integrity
docker-compose exec gitea sqlite3 /data/gitea/gitea.db 'PRAGMA integrity_check;'
```

## Security Configuration

### Access Control
Production security settings:
1. **Admin Panel → Configuration → Security**
2. Configure:
   - **Disable Git Hooks**: ✓ (unless needed)
   - **Disable Webhooks**: ✗ (needed for Jenkins)
   - **Enable CSRF Protection**: ✓
   - **Require Sign-in to View**: ✓ (for private deployments)

### Rate Limiting
Prevent abuse:
1. **Admin Panel → Configuration → API**
2. Set reasonable rate limits:
   - **API Rate Limit**: 1000/hour
   - **API Rate Limit per User**: 100/hour

### Security Headers
Configure reverse proxy (Traefik) for security headers:
```yaml
# traefik.yml
http:
  middlewares:
    security-headers:
      headers:
        customRequestHeaders:
          X-Forwarded-Proto: "https"
        customResponseHeaders:
          X-Frame-Options: "SAMEORIGIN"
          X-Content-Type-Options: "nosniff"
          Referrer-Policy: "strict-origin-when-cross-origin"
```

## Monitoring and Logs

### Log Management
```bash
# View Gitea logs
docker-compose logs -f gitea

# Check specific log files
tail -f ./data/gitea/log/gitea.log

# Rotate logs (configure in app.ini)
[log]
LEVEL = Info
ROOT_PATH = /data/gitea/log
MAX_SIZE_SHIFT = 28
DAILY_ROTATE = true
MAX_DAYS = 7
```

### Performance Monitoring
```bash
# Monitor resource usage
docker stats gitea

# Check disk usage
du -sh ./data/gitea/

# Database size monitoring
ls -lh ./data/gitea/gitea.db
```

## Troubleshooting

### Common Issues

#### 1. Repository Access Issues
```bash
# Check permissions
ls -la ./data/gitea/git/repositories/

# Fix ownership
sudo chown -R 1000:1000 ./data/gitea/
```

#### 2. Webhook Failures
```bash
# Test network connectivity
docker-compose exec gitea ping jenkins
docker-compose exec gitea curl http://jenkins:8081

# Check webhook logs
docker-compose logs gitea | grep webhook
```

#### 3. Database Corruption
```bash
# Backup current database
cp ./data/gitea/gitea.db ./data/gitea/gitea.db.backup

# Repair SQLite database
docker-compose exec gitea sqlite3 /data/gitea/gitea.db '.recovery'
```

#### 4. Performance Issues
```bash
# Check Git repository integrity
docker-compose exec gitea bash -c "
cd /data/git/repositories/admin/bad-python-app-kyle-managed.git
git fsck --full
"

# Optimize repositories
docker-compose exec gitea bash -c "
cd /data/git/repositories/admin/bad-python-app-kyle-managed.git
git gc --aggressive
"
```

### Network Troubleshooting
```bash
# Check container networking
docker network inspect cloud-lab_lab-network

# Verify service discovery
docker-compose exec jenkins nslookup gitea
docker-compose exec gitea nslookup jenkins
```

## Integration Testing

### End-to-End Workflow Test
1. **Push code to Gitea**:
   ```bash
   echo "Test change" >> README.md
   git add README.md
   git commit -m "Test webhook trigger"
   git push origin main
   ```

2. **Verify webhook delivery** in Gitea UI

3. **Check Jenkins build** was triggered

4. **Confirm Semgrep scan** completed successfully

### API Integration Test
```bash
# Test Gitea API
curl -H "Authorization: token [API_TOKEN]" \
  http://your-server:3000/api/v1/user/repos

# Test repository API
curl -H "Authorization: token [API_TOKEN]" \
  http://your-server:3000/api/v1/repos/admin/bad-python-app-kyle-managed
```

---
*Next: [Semgrep Integration Guide](./semgrep-integration.md)*