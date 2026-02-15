# Ansible Step-CA Docker

[![Ansible](https://img.shields.io/badge/Ansible-2.14+-blue.svg)](https://www.ansible.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Deploy [Smallstep step-ca](https://smallstep.com/docs/step-ca) Certificate Authority in Docker using Ansible.

## Description

This Ansible playbook automates the deployment of a private Certificate Authority (CA) using Smallstep's step-ca in a Docker container. It provides:

- **Automated CA initialization** with configurable DNS names and passwords
- **ACME protocol support** for automatic certificate issuance (like Let's Encrypt, but private)
- **Configurable certificate validity** durations
- **Secure defaults** with proper file permissions and dedicated system user
- **Docker Compose** based deployment for easy management

## Requirements

### Control Node
- Ansible 2.14+
- Python 3.8+

### Target Host
- Debian 11/12 or Ubuntu 20.04/22.04/24.04
- Docker Engine with Docker Compose v2
- Python 3.8+
- Root or sudo access

### Ansible Collections

Install required collections:

```bash
ansible-galaxy install -r requirements.yml
```

## Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/tegge/ansible-stepca.git
   cd ansible-stepca
   ```

2. **Install dependencies**
   ```bash
   ansible-galaxy install -r requirements.yml
   ```

3. **Configure inventory**
   ```bash
   cp inventories/example/inventory.yml inventories/production/inventory.yml
   # Edit with your server details
   ```

4. **Configure variables**
   ```bash
   cp group_vars/all/vars.example.yml group_vars/all/vars.yml
   # Edit with your settings
   ```

5. **Create vault for secrets**
   ```bash
   ansible-vault create group_vars/all/vault.yml
   # Add: vault_stepca_password: "your-secure-password-here"
   ```

## Usage

### Basic Deployment

```bash
ansible-playbook playbooks/site.yml
```

### With Custom Inventory

```bash
ansible-playbook playbooks/site.yml -i inventories/production/inventory.yml
```

### With Vault Password

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### Check Mode (Dry Run)

```bash
ansible-playbook playbooks/site.yml --check --diff
```

## Configuration Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `stepca_init_password` | CA initialization password (min 16 chars) | `{{ vault_stepca_password }}` |
| `stepca_init_dns` | DNS names for CA certificate | `["ca.example.com", "localhost"]` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `stepca_init_name` | `My Private CA` | CA display name |
| `stepca_data_dir` | `/data/step-ca/volume` | Data directory path |
| `stepca_compose_dir` | `/data/step-ca` | Docker Compose directory |
| `stepca_bind_port` | `8443` | Host port to expose |
| `stepca_host_user` | `stepca` | System user for container |
| `stepca_uid` | `1501` | User ID |
| `stepca_gid` | `1501` | Group ID |
| `stepca_docker_image` | `smallstep/step-ca:latest` | Docker image |
| `stepca_container_name` | `ca` | Container name |
| `stepca_restart_policy` | `unless-stopped` | Docker restart policy |
| `stepca_dns_servers` | `["1.1.1.1", "8.8.8.8"]` | DNS servers for container |

### Certificate Validity Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `stepca_cert_min_duration` | `5m` | Minimum certificate validity |
| `stepca_cert_default_duration` | `2160h` (90 days) | Default certificate validity |
| `stepca_cert_max_duration` | `8760h` (1 year) | Maximum certificate validity |

## Examples

### Minimal Configuration

```yaml
# group_vars/all/vars.yml
stepca_init_password: "{{ vault_stepca_password }}"
stepca_init_dns:
  - "ca.mycompany.com"
```

### Full Configuration

```yaml
# group_vars/all/vars.yml
stepca_init_name: "MyCompany Internal CA"
stepca_init_dns:
  - "ca.mycompany.com"
  - "ca.internal.lan"
  - "192.168.1.100"

stepca_data_dir: "/opt/step-ca/data"
stepca_compose_dir: "/opt/step-ca"
stepca_bind_port: 443

stepca_host_user: "ca-service"
stepca_uid: 2000
stepca_gid: 2000

stepca_cert_default_duration: "720h"   # 30 days
stepca_cert_max_duration: "2160h"      # 90 days

stepca_dns_servers:
  - "10.0.0.1"
  - "1.1.1.1"
```

### Using with Traefik

To use step-ca as an ACME provider for Traefik:

```yaml
# traefik configuration
certificatesResolvers:
  stepca:
    acme:
      email: admin@example.com
      storage: /letsencrypt/acme.json
      caServer: https://ca.example.com:8443/acme/acme/directory
      tlsChallenge: {}
```

## Post-Deployment

### Bootstrap a Client

After deployment, bootstrap clients to trust your CA:

```bash
# Get the root CA fingerprint
step certificate fingerprint /data/step-ca/volume/certs/root_ca.crt

# On client machines
step ca bootstrap \
  --ca-url https://ca.example.com:8443 \
  --fingerprint <fingerprint-from-above>
```

### Request a Certificate

```bash
step ca certificate myserver.example.com server.crt server.key
```

### View CA Status

```bash
docker logs ca
docker exec ca step ca health
```

## Directory Structure

```
.
├── README.md
├── LICENSE
├── ansible.cfg
├── requirements.yml
├── playbooks/
│   └── site.yml              # Main playbook
├── roles/
│   └── step_ca_docker/
│       ├── defaults/main.yml # Default variables
│       ├── tasks/main.yml    # Role tasks
│       ├── templates/        # Jinja2 templates
│       ├── handlers/main.yml # Handlers
│       └── meta/main.yml     # Role metadata
├── inventories/
│   └── example/
│       └── inventory.yml     # Example inventory
└── group_vars/
    └── all/
        ├── vars.example.yml  # Example variables
        └── vault.example.yml # Example vault
```

## Troubleshooting

### CA Container Won't Start

1. Check container logs:
   ```bash
   docker logs ca
   ```

2. Verify data directory permissions:
   ```bash
   ls -la /data/step-ca/volume
   ```

3. Ensure the password meets minimum length (16 chars)

### Certificate Request Fails

1. Verify CA is healthy:
   ```bash
   docker exec ca step ca health
   ```

2. Check ACME provisioner is enabled:
   ```bash
   docker exec ca step ca provisioner list
   ```

### Port Already in Use

Change `stepca_bind_port` to an available port, or stop the conflicting service.

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Run linting: `ansible-lint playbooks/ roles/`
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

### Development Setup

```bash
# Clone your fork
git clone https://github.com/tegge/ansible-stepca.git
cd ansible-stepca

# Install dev dependencies
pip install ansible-lint yamllint

# Run linting
ansible-lint playbooks/ roles/
yamllint .
```

## Security Considerations

- **Vault passwords**: Never commit unencrypted secrets
- **CA private key**: The CA private key at `stepca_data_dir/secrets/` is critical - protect and backup
- **Network access**: Consider firewall rules to restrict access to port 8443
- **Root CA**: Distribute the root certificate securely to clients

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Smallstep](https://smallstep.com/) for step-ca
- [Ansible Community](https://www.ansible.com/community)

## Related Projects

- [step-ca Documentation](https://smallstep.com/docs/step-ca)
- [step CLI](https://smallstep.com/docs/step-cli)
- [ACME Protocol](https://datatracker.ietf.org/doc/html/rfc8555)
