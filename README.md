# Pipeline Deployment

Ansible automation for deploying Docker services and systemd services across multiple environments.

## Overview

This repository contains Ansible playbooks and roles for:
- **Docker Services**: Deploy containerized applications using Docker Compose
- **System Services**: Deploy and manage systemd services
- **Docker Setup**: Automated Docker and Docker Compose installation

## Features

### Docker Service Deployment (`deploy-services.yml`)
- Multi-environment support (Dev/QA/Prod)
- Docker Compose orchestration
- Automatic image pulling
- Service health verification
- Container log inspection
- Automatic cleanup of old images
- Template-based configuration

### System Service Deployment (`deploy-system-service.yml`)
- Systemd service management
- Service installation and configuration
- Automatic service startup
- Health monitoring
- Log inspection with journalctl
- Configuration templating

### Docker Setup Role (`roles/setup_docker`)
- Cross-platform Docker installation (Ubuntu/Debian, RHEL/CentOS)
- Docker Compose installation
- Docker daemon configuration
- User permission management
- Service verification

## Prerequisites

### Control Node (Where Ansible runs)
- Ansible 2.9 or higher
- Python 3.6 or higher
- SSH access to target hosts

### Target Hosts
- Ubuntu 20.04+, RHEL 8+, or CentOS 8+
- Python 3.6 or higher
- SSH server running
- Sudo privileges for deployment user

## Directory Structure

```
pipeline-deployment/
├── deploy-services.yml           # Docker service deployment playbook
├── deploy-system-service.yml     # Systemd service deployment playbook
├── ansible.cfg                   # Ansible configuration
├── roles/
│   └── setup_docker/
│       └── tasks/
│           └── main.yml         # Docker installation tasks
└── README.md                    # This file
```

## Usage

### 1. Deploy Docker Service

Deploy a containerized service to an environment:

```bash
ansible-playbook deploy-services.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=dev service_name=user-api"
```

With custom variables:

```bash
ansible-playbook deploy-services.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=qa service_name=product-frontend" \
  -e "docker_registry=my-registry.example.com"
```

### 2. Deploy System Service

Deploy a systemd service:

```bash
ansible-playbook deploy-system-service.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=prod service_name=data-processor"
```

### 3. Setup Docker on New Hosts

Install Docker on all hosts in an environment:

```bash
ansible-playbook -i ../deployment-config/inventory.ini \
  -m include_role \
  -a name=setup_docker \
  -l dev
```

Or as part of service deployment (automatically included):

```bash
ansible-playbook deploy-services.yml \
  -i ../deployment-config/inventory.ini \
  -e "env=dev service_name=user-api"
```

## Configuration

### Ansible Configuration (`ansible.cfg`)

Key settings:
- **Inventory**: Points to `../deployment-config/inventory.ini`
- **Remote User**: `ansible` (default)
- **Privilege Escalation**: Enabled with sudo
- **Forks**: 5 parallel processes
- **Logging**: `/var/log/ansible/ansible.log`

Customize by editing `ansible.cfg`:

```ini
[defaults]
remote_user = your_user
private_key_file = ~/.ssh/your_key
forks = 10
```

### Environment Variables

Pass these variables using `-e` flag:

#### Required
- `env`: Target environment (dev/qa/prod)
- `service_name`: Name of the service to deploy

#### Optional
- `docker_registry`: Docker registry URL (default: localhost:5000)
- `services_base_path`: Installation path (default: /opt/services)

## Service Configuration

Services are configured in the `deployment-config` repository:

### Docker Services
Configuration template: `deployment-config/services/<service-name>/docker-compose.yaml.j2`

Example:
```yaml
services:
  user-api:
    image: "{{ docker_registry }}/user-api:{{ version }}"
    container_name: user-api
    ports:
      - "8080:8080"
    environment:
      - DB_HOST={{ db_host }}
```

### System Services
Configuration directory: `deployment-config/system-services/<service-name>/`

Required files:
- `<service-name>.service.j2`: Systemd unit file
- Additional config files (copied to `/opt/services/<service-name>/`)

Example systemd service:
```ini
[Unit]
Description=Data Processor Service
After=network.target

[Service]
Type=simple
User=serviceuser
WorkingDirectory=/opt/services/data-processor
ExecStart=/opt/services/data-processor/processor
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Deployment Workflow

### Docker Service Deployment Flow

1. **Validation**: Check required variables and service config
2. **Preparation**: Create service directories
3. **Configuration**: Template docker-compose files
4. **Shutdown**: Stop existing containers
5. **Update**: Pull latest Docker images
6. **Startup**: Start containers with docker-compose
7. **Verification**: Check container health
8. **Cleanup**: Remove dangling images

### System Service Deployment Flow

1. **Validation**: Check required variables and service config
2. **Shutdown**: Stop existing service
3. **Preparation**: Create service directories
4. **Transfer**: Copy service files to target
5. **Installation**: Install systemd unit file
6. **Startup**: Enable and start service
7. **Verification**: Check service status and logs

## Monitoring and Troubleshooting

### Check Service Status

Docker service:
```bash
ssh user@host
cd /opt/services/user-api
docker compose ps
docker compose logs
```

System service:
```bash
ssh user@host
sudo systemctl status data-processor
sudo journalctl -u data-processor -f
```

### Common Issues

#### Docker not found
The `setup_docker` role will automatically install Docker. If it fails:
```bash
ansible-playbook -i inventory.ini setup_docker.yml -l <host>
```

#### Permission denied
Ensure deployment user has sudo privileges:
```bash
sudo usermod -aG docker ansible
sudo usermod -aG sudo ansible
```

#### Service won't start
Check logs:
```bash
# Docker service
docker compose -f /opt/services/<service>/docker-compose.yml logs

# System service
sudo journalctl -u <service> -n 50
```

## Integration with Jenkins

These playbooks are typically called from Jenkins pipelines defined in `deployment-config/jenkinsfile`:

```groovy
stage('Deploy Service') {
    steps {
        script {
            sh """
                ansible-playbook deploy-services.yml \\
                  -i ../deployment-config/inventory.ini \\
                  -e "env=${env.ENVIRONMENT}" \\
                  -e "service_name=${env.SERVICE_NAME}"
            """
        }
    }
}
```

## Security Considerations

1. **SSH Keys**: Use SSH key authentication, not passwords
2. **Ansible Vault**: Store sensitive variables in encrypted vault files
3. **Sudo Password**: Use vault or `--ask-become-pass` for sudo password
4. **Docker Registry**: Use private registries with authentication
5. **File Permissions**: Ensure config files have appropriate permissions

## Best Practices

1. **Test in Dev First**: Always test deployments in dev environment
2. **Use Tags**: Version your Docker images with tags
3. **Health Checks**: Define health checks in docker-compose files
4. **Logging**: Configure centralized logging for all services
5. **Backups**: Backup service data before deployments
6. **Rollback Plan**: Keep previous versions for quick rollback

## Requirements

### Ansible Collections
```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker
```

### Python Packages
```bash
pip install ansible docker docker-compose
```

## Contributing

To add new deployment types:
1. Create new playbook in root directory
2. Add corresponding role in `roles/` if needed
3. Update this README with usage instructions
4. Test in dev environment first

## License

Internal use only - proprietary
