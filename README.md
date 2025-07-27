# Cloud Lab Documentation

A containerized development environment for Semgrep Solutions Engineering featuring Jenkins CI/CD, Gitea SCM, and integrated security scanning workflows.

## Overview

This lab environment provides a complete DevSecOps pipeline for demonstrating Semgrep integration patterns, troubleshooting customer scenarios, and building reference architectures. Perfect for Solutions Engineers who need hands-on experience with real integration challenges.

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     Traefik     │    │     Jenkins     │    │      Gitea      │
│ (Reverse Proxy) │    │    (CI/CD)      │    │   (Git Server)  │
│    Port 80/443  │    │    Port 8081    │    │    Port 3000    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
         ┌─────────────────┐    ┌─────────────────┐
         │   PostgreSQL    │    │ Semgrep Network │
         │   (Database)    │    │     Broker      │
         │      Hidden     │    │   Port 51820    │
         └─────────────────┘    └─────────────────┘
```

## Services

- **Traefik**: Routes traffic and provides SSL termination
- **Jenkins**: CI/CD pipeline automation with Semgrep integration
- **Gitea**: Self-hosted Git service for repository management
- **PostgreSQL**: Database backend for applications
- **Semgrep Network Broker**: Secure connection to Semgrep Cloud

## Quick Start

### Prerequisites
- Docker and Docker Compose installed
- DigitalOcean droplet (or similar) with at least 4GB RAM
- Domain name configured (optional, for traefik routing)

### 1. Clone and Setup
```bash
git clone https://github.com/hyjinx187/cloudlab-docs.git
cd cloudlab-docs
```

### 2. Start Services (Resource-Efficient)
```bash
# Start only essential services
docker-compose up -d jenkins gitea

# Or start everything
docker-compose up -d
```

### 3. Access Services
- **Jenkins**: http://your-server:8081
- **Gitea**: http://your-server:3000
- **Traefik Dashboard**: http://your-server:8080

## Resource Management

For resource-constrained environments:

```bash
# Development mode (minimal resources)
docker-compose up -d gitea postgres

# Demo mode (Jenkins + Git)
docker-compose up -d jenkins gitea

# Full environment
docker-compose up -d

# Stop when not needed
docker-compose down
```

## Directory Structure

```
cloud-lab/
├── docker/
│   ├── docker-compose.yml          # Main service definitions
│   ├── configs/
│   │   └── traefik.yml            # Reverse proxy configuration
│   └── data/                      # Persistent data volumes
│       ├── jenkins/               # Jenkins home directory
│       ├── gitea/                 # Gitea data and repositories
│       ├── postgres/              # Database storage
│       └── logs/                  # Application logs
├── repos/                         # Local repository checkouts
└── semgrep/                       # Semgrep configuration files
```

## Use Cases

### For Solutions Engineers
- **Customer POC Setup**: Replicate customer environments quickly
- **Integration Testing**: Test Semgrep with various CI/CD tools
- **Troubleshooting**: Debug real integration issues
- **Demo Preparation**: Consistent, reliable demo environment

### Technical Scenarios
- Jenkins pipeline optimization (11min → 2min scan times)
- Rule configuration and tuning
- Platform integration troubleshooting
- Performance analysis and optimization

## Documentation Index

- [Docker Compose Setup](./docker-compose-setup.md) - Service configuration and deployment
- [Jenkins Configuration](./jenkins-setup.md) - CI/CD pipeline setup and Semgrep integration
- [Gitea Setup](./gitea-setup.md) - Git server configuration and webhook setup
- [Semgrep Integration](./semgrep-integration.md) - Security scanning configuration and troubleshooting
- [Resource Management](./resource-management.md) - Optimizing for cost and performance
- [Troubleshooting Guide](./troubleshooting.md) - Common issues and solutions

## Security Notes

⚠️ **Important**: This is a development/demo environment. Do not use in production without proper security hardening.

- All secrets are redacted in documentation
- Obtain actual credentials from your Semgrep organization
- Use environment variables for sensitive configuration
- Enable proper authentication in production deployments

## Contributing

This documentation is maintained for Semgrep Solutions Engineering use. Update as you discover new integration patterns or troubleshooting scenarios.

---
*Built for Semgrep Solutions Engineers by Kyle Northcutt*