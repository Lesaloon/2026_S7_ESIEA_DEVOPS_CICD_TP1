# CI/CD Pipeline - WordPress on Kubernetes

A production-ready CI/CD pipeline that automates the deployment of WordPress and MySQL on Kubernetes with health probes, persistent storage, and comprehensive monitoring.

## 📋 Project Overview

This project implements a complete CI/CD pipeline that:

- Validates Docker Compose configuration
- Performs health checks on containerized services
- Converts services to Kubernetes manifests with enhanced features
- Packages and deploys to an FTP server
- Includes production-grade health probes and monitoring

## 🏗️ Architecture

### Deployment Configuration

| Service       | Image                         | Replicas | Storage | CPU       | Memory      |
| ------------- | ----------------------------- | -------- | ------- | --------- | ----------- |
| **WordPress** | wordpress:6.5.3-php8.2-apache | 4        | 5Gi     | 250m-500m | 256Mi-512Mi |
| **MySQL**     | mysql:5.6                     | 1        | 5Gi     | 250m-500m | 256Mi-512Mi |

### Network & Storage

- **Networking**: Internal bridge network with service discovery
- **Storage**: Persistent volumes for data durability
  - MySQL: 5Gi ReadWriteOnce (RWO)
  - WordPress: 5Gi ReadWriteMany (RWX)
- **Security**: Kubernetes Secrets with base64-encoded credentials

## 🔄 CI/CD Pipeline Workflow

The pipeline consists of 4 sequential jobs:

### Job 1: Validate Docker Compose

- **Description**: Validates syntax and structure of `docker-compose.yml`
- **Command**: `docker-compose config -q`
- **Trigger**: Every commit and pull request

### Job 2: Health Check

- **Description**: Starts containers and verifies they are healthy
- **Steps**:
  1. Creates secret files from GitHub Secrets
  2. Runs `docker-compose up -d`
  3. Performs health checks (container status verification)
  4. Displays logs on failure
  5. Cleans up containers
- **Timeout**: 60 seconds (max 30 attempts with 2-second intervals)

### Job 3: Prepare Kubernetes Manifests

- **Description**: Generates Kubernetes manifests from templates with health probes
- **Steps**:
  1. Copies custom K8s templates with health probes
  2. Generates Secret manifest with base64-encoded credentials
  3. Validates YAML syntax for all manifests
- **Features**:
  - Liveness probes for automatic restart on failure
  - Readiness probes for load balancing
  - Init containers for dependency management
  - Pod anti-affinity for high availability

### Job 4: Package and Upload to FTP

- **Description**: Packages manifests and uploads to FTP server
- **Steps**:
  1. Downloads generated manifests
  2. Creates folder structure: `CarvilleGuillian/`
  3. Creates zip: `CarvilleGuillianWordpress.zip`
  4. Uploads via LFTP with SSL verification disabled
- **Output**: Zip file in `RenduDevopsKube/` directory on FTP

## 🔐 GitHub Secrets Configuration

You must configure the following secrets in your GitHub repository settings:

### FTP Credentials (3 secrets)

```bash
# Replace with your FTP credentials
gh secret set FTP_HOST --body "ftp.cluster114.hosting.ovh.net"
gh secret set FTP_USER --body "your_ftp_username"
gh secret set FTP_PASSWORD --body "your_ftp_password"
```

### MySQL Credentials (4 secrets)

```bash
# Replace with secure values
gh secret set MYSQL_ROOT_PASSWORD --body "YourSecureRootPassword123!"
gh secret set MYSQL_DATABASE --body "wordpress"
gh secret set MYSQL_USER --body "wordpressuser"
gh secret set MYSQL_PASSWORD --body "YourSecureDBPassword456!"
```

### Verify Secrets

```bash
gh secret list
```

## 📦 Generated Kubernetes Manifests

The pipeline generates the following files in `CarvilleGuillianWordpress.zip`:

```
CarvilleGuillian/
├── 00-mysql-secret.yaml              # Base64-encoded secrets
├── mysql-deployment.yaml             # MySQL with liveness/readiness probes
├── mysql-service.yaml                # ClusterIP service for MySQL
├── persistent-volumes.yaml           # PVC for MySQL (5Gi) & WordPress (5Gi)
├── wordpress-deployment.yaml         # 4 replicas with anti-affinity & probes
├── wordpress-service.yaml            # LoadBalancer service
├── wordpress-configmap.yaml          # Configuration settings
├── ingress.yaml                      # Ingress rules (optional)
├── network-policy.yaml               # Network security policies
└── backup-cronjob.yaml               # Daily backup jobs
```

## 🚀 Deployment Instructions

### 1. Clone Repository

```bash
git clone <repository-url>
cd 2026_S7_ESIEA_DEVOPS_CICD_TP1
```

### 2. Configure GitHub Secrets

See [GitHub Secrets Configuration](#-github-secrets-configuration) above

### 3. Deploy to Kubernetes

```bash
# Download the generated zip from FTP server
# Extract manifests
unzip CarvilleGuillianWordpress.zip
cd CarvilleGuillian

# Create namespace (optional)
kubectl create namespace wordpress

# Apply manifests (order matters for secrets)
kubectl apply -f 00-mysql-secret.yaml
kubectl apply -f persistent-volumes.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f *.yaml

# Verify deployment
kubectl get pods
kubectl get svc
```

### 4. Verify Deployment

```bash
# Check pod status
kubectl get pods -o wide

# View WordPress service
kubectl get svc wordpress

# Check MySQL connectivity
kubectl logs deployment/mysql

# Check WordPress logs
kubectl logs deployment/wordpress
```

## 🔍 Health Probes Details

### MySQL Health Checks

- **Liveness Probe**: `mysqladmin ping` command
  - Initial delay: 30 seconds
  - Period: 10 seconds
  - Timeout: 5 seconds
  - Failure threshold: 3 consecutive failures

- **Readiness Probe**: MySQL query execution
  - Initial delay: 20 seconds
  - Period: 5 seconds
  - Timeout: 3 seconds
  - Failure threshold: 2 consecutive failures

### WordPress Health Checks

- **Liveness Probe**: HTTP GET to `/` (root)
  - Initial delay: 40 seconds
  - Period: 15 seconds
  - Timeout: 5 seconds
  - Failure threshold: 3 consecutive failures

- **Readiness Probe**: HTTP GET to `/wp-login.php`
  - Initial delay: 20 seconds
  - Period: 5 seconds
  - Timeout: 3 seconds
  - Failure threshold: 2 consecutive failures

## 📊 Docker Compose Secrets Feature

The pipeline uses Docker Compose secrets feature for secure credential handling:

```yaml
secrets:
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt
  mysql_database:
    file: ./secrets/mysql_database.txt
  mysql_user:
    file: ./secrets/mysql_user.txt
  mysql_password:
    file: ./secrets/mysql_password.txt
```

Secret files are created at runtime from GitHub Secrets and mounted at `/run/secrets/` inside containers.

## 🔄 How to Trigger the Pipeline

### Automatic Triggers

- Push to `main` branch
- Pull request to `main` branch

### Manual Trigger via Git

```bash
# Make changes to docker-compose.yml or other files
git add .
git commit -m "Update configuration"
git push origin main
```

### View Pipeline Status

```bash
# List recent workflow runs
gh run list --limit 5

# View specific run details
gh run view <run-id> --log

# Watch logs in real-time
gh run watch <run-id>
```

## 📋 Directory Structure

```
.
├── docker-compose.yml              # Docker Compose definition
├── .github/
│   └── workflows/
│       └── ci-cd.yml               # GitHub Actions workflow
├── k8s-templates/                  # Kubernetes manifest templates
│   ├── mysql-deployment.yaml
│   ├── wordpress-deployment.yaml
│   ├── persistent-volumes.yaml
│   ├── wordpress-configmap.yaml
│   ├── ingress.yaml
│   ├── network-policy.yaml
│   └── backup-cronjob.yaml
├── scripts/                        # Initialization scripts
│   ├── init-wordpress.sh           # WordPress setup with wp-cli
│   └── init-wordpress-sql.sh       # SQL-based initialization
└── README.md                       # This file
```

## 🛠️ Troubleshooting

### Pipeline Fails on Docker Compose Validation

- Check YAML syntax in `docker-compose.yml`
- Run locally: `docker-compose config -q`

### Health Check Fails

- Verify GitHub Secrets are set correctly
- Check container logs in workflow output
- Ensure Docker is available on the runner

### Manifest Generation Issues

- Verify YAML templates in `k8s-templates/` directory
- Check for missing required fields in manifests
- Validate with: `kubectl apply --dry-run=client -f manifest.yaml`

### FTP Upload Fails

- Verify FTP credentials in GitHub Secrets
- Check FTP host is reachable
- Ensure SSL verification is disabled for self-signed certificates
- Check LFTP is installed: `lftp --version`

## 📈 Monitoring & Maintenance

### View Artifact History

```bash
# List all artifacts for a run
gh run list --json artifacts

# Download latest artifacts
gh run download <run-id> -n k8s-manifests
```

### Database Backups

- Automatic daily backups via CronJob at 2 AM UTC
- Backups stored in 20Gi persistent volume
- Restore: `mysql < backup.sql`

### Logs

```bash
# View workflow logs
kubectl logs -f deployment/mysql
kubectl logs -f deployment/wordpress

# View previous pod logs (if crashed)
kubectl logs <pod-name> --previous
```

## 🔒 Security Best Practices

1. **GitHub Secrets**: Never commit credentials to git
2. **Base64 Encoding**: Kubernetes Secrets are encoded, not encrypted (use etcd encryption)
3. **Network Policies**: Restrict traffic between pods
4. **RBAC**: Configure role-based access control for your cluster
5. **SSL/TLS**: Enable ingress with TLS certificates

## 📝 Configuration

### Adjust Replica Count

Edit `k8s-templates/wordpress-deployment.yaml`:

```yaml
spec:
  replicas: 4 # Change this value
```

### Adjust Storage Size

Edit `k8s-templates/persistent-volumes.yaml`:

```yaml
resources:
  requests:
    storage: 5Gi # Change this value
```

### Adjust Resource Limits

Edit deployment manifests:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

## 🤝 Contributing

1. Create a feature branch
2. Make your changes
3. Push and create a pull request
4. Pipeline will automatically validate

## 📄 Technology Stack

- **Docker & Docker Compose**: Local testing and validation
- **GitHub Actions**: CI/CD orchestration
- **Kubernetes**: Container orchestration
- **MySQL 5.6**: Database
- **WordPress 6.5.3**: CMS
- **LFTP**: FTP client for deployment

## 📞 Support

For issues or questions:

- Check the troubleshooting section
- Review GitHub Actions logs
- Verify all GitHub Secrets are configured

## 📜 License

School project - ESIEA DevOps 2026

---

**Last Updated**: January 16, 2026  
**Pipeline Status**: ✅ Production Ready  
**Tested Configurations**: Ubuntu 22.04 LTS, Kubernetes 1.28+
