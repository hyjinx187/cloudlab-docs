# Docker Compose Setup Guide

Complete configuration guide for the containerized cloud lab environment.

## docker-compose.yml Configuration

### Full Service Stack

```yaml
services:
  # Reverse proxy for multiple services
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./configs/traefik.yml:/etc/traefik/traefik.yml
    networks:
      - lab-network

  # Git server (Gitea)
  gitea:
    image: gitea/gitea:latest
    environment:
      - USER_UID=1000
      - USER_GID=1000
    volumes:
      - ./data/gitea:/data
    ports:
      - "3000:3000"
    networks:
      - lab-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gitea.rule=Host(`git.lab.kylenorthcutt.com`)"

  # CI/CD (Jenkins)
  jenkins:
    image: jenkins/jenkins:lts
    volumes:
      - ./data/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8081:8080"
    networks:
      - lab-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`ci.lab.kylenorthcutt.com`)"
      - "traefik.http.services.jenkins.loadbalancer.server.port=8080"

  # Development database
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: [REDACTED - Generate secure password]
      POSTGRES_DB: devdb
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    networks:
      - lab-network

  # Semgrep Network Broker
  semgrep-broker:
    image: ghcr.io/semgrep/semgrep-network-broker:latest
    container_name: semgrep-network-broker
    volumes:
      - ../semgrep/config.yaml:/config.yaml:ro
    command: ["-c", "/config.yaml"]
    restart: unless-stopped
    networks:
      - lab-network
    ports:
      - "51820:51820/udp"
    cap_add:
      - NET_ADMIN
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    labels:
      - "traefik.enable=false"

networks:
  lab-network:
    driver: bridge
```

## Deployment Commands

### Resource-Efficient Startup Options

```bash
# Option 1: Minimal demo environment (Jenkins + Gitea only)
docker-compose up -d jenkins gitea

# Option 2: Development environment (with database)
docker-compose up -d jenkins gitea postgres

# Option 3: Full environment (all services)
docker-compose up -d

# Option 4: Semgrep testing environment
docker-compose up -d jenkins gitea semgrep-broker
```

### Management Commands

```bash
# View running services
docker-compose ps

# Check service logs
docker-compose logs -f jenkins
docker-compose logs -f gitea

# Stop specific services
docker-compose stop postgres semgrep-broker

# Stop all services (preserves data)
docker-compose down

# Stop all services and remove volumes (DESTRUCTIVE)
docker-compose down -v
```

## Service Configuration Details

### Traefik (Reverse Proxy)
- **Purpose**: Routes traffic, SSL termination, load balancing
- **Access**: http://your-server:8080 (dashboard)
- **Configuration**: `./configs/traefik.yml`
- **Domain Routing**: Configured for subdomain access

### Jenkins (CI/CD)
- **Purpose**: Build automation, Semgrep integration
- **Access**: http://your-server:8081
- **Data**: Persisted in `./data/jenkins/`
- **Docker Access**: Mounted socket for Docker-in-Docker builds
- **Key Features**: Pipeline automation, webhook integration

### Gitea (Git Server)
- **Purpose**: Self-hosted Git repositories
- **Access**: http://your-server:3000
- **Data**: Persisted in `./data/gitea/`
- **Features**: Webhook support, API access, user management

### PostgreSQL (Database)
- **Purpose**: Backend database for applications
- **Access**: Internal network only (no exposed ports)
- **Data**: Persisted in `./data/postgres/`
- **Credentials**: Set via environment variables

### Semgrep Network Broker
- **Purpose**: Secure connection to Semgrep Cloud
- **Configuration**: Requires `../semgrep/config.yaml`
- **Networking**: UDP port 51820 for VPN tunnel
- **Security**: Runs with elevated network privileges

## Directory Structure Setup

```bash
# Create required directories
mkdir -p cloud-lab/docker/configs
mkdir -p cloud-lab/docker/data/{jenkins,gitea,postgres}
mkdir -p cloud-lab/semgrep
mkdir -p cloud-lab/repos
```

## Environment Variables

Create a `.env` file for sensitive configuration:

```bash
# .env file (DO NOT COMMIT TO GIT)
POSTGRES_PASSWORD=[OBTAIN_FROM_SECURE_SOURCE]
SEMGREP_APP_TOKEN=[OBTAIN_FROM_SEMGREP_DASHBOARD]
DOMAIN_NAME=your-domain.com
```

## Security Considerations

### Secrets Management
- **PostgreSQL Password**: Generate using `openssl rand -base64 32`
- **Semgrep App Token**: Obtain from Semgrep Dashboard → Settings → Tokens
- **Jenkins Admin Password**: Found in `./data/jenkins/secrets/initialAdminPassword`

### Network Security
- Services communicate via internal `lab-network`
- Only necessary ports exposed to host
- Traefik handles SSL termination

### Data Persistence
- All service data persisted in `./data/` volumes
- Backup strategy recommended for production use
- Git repositories safely stored in Gitea volume

## Troubleshooting

### Common Issues

**Services won't start:**
```bash
# Check for port conflicts
netstat -tulpn | grep :8081
netstat -tulpn | grep :3000

# Check Docker daemon
systemctl status docker
```

**Gitea database connection issues:**
```bash
# Check postgres logs
docker-compose logs postgres

# Verify network connectivity
docker-compose exec gitea ping postgres
```

**Jenkins permission issues:**
```bash
# Fix volume permissions
sudo chown -R 1000:1000 ./data/jenkins
```

### Resource Monitoring

```bash
# Monitor resource usage
docker stats

# Check disk usage
du -sh ./data/*

# View system resources
free -h
df -h
```

## Scaling Considerations

### Vertical Scaling
- Increase DigitalOcean droplet size for more resources
- Adjust Docker memory limits if needed

### Horizontal Scaling
- Add Jenkins agents for distributed builds
- Consider external database for production

### Cost Optimization
- Use selective service startup for demos
- Schedule automatic shutdown during off-hours
- Monitor usage with DigitalOcean dashboard

---
*Next: [Jenkins Configuration Guide](./jenkins-setup.md)*