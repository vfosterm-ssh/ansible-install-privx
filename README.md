# PrivX Installation and Configuration Automation – Ansible

[![Ansible Tests](https://img.shields.io/github/actions/workflow/status/SSHcom/ansible-install-privx/ansible-tests.yml?style=for-the-badge&label=Ansible%20Tests)](https://github.com/SSHcom/ansible-install-privx/actions)
[![Last Commit](https://img.shields.io/github/last-commit/SSHcom/ansible-install-privx.svg?style=for-the-badge)](https://github.com/SSHcom/ansible-install-privx/commits)

This repository contains Ansible roles and playbooks for automated PrivX and additional components installation and configuration on multiple Linux distributions.

## Overview

**Unified Deployment** with role-based architecture:
- **PrivX Core**: Installation and configuration on primary and additional nodes
- **PrivX Extender**: Extender installation and configuration with non-FIPS or FIPS
- **PrivX WAG**: Web Access Gateway (Carrier and Web Proxy) installation and configuration
- **PrivX Core PoC Single Node**: Installation and configuration on primary using local PostgreSQL


## Supported Operating Systems

- Red Hat Enterprise Linux 8/9
- Rocky Linux 8/9
- Amazon Linux 2023 (**PrivX Core Only**)

## Prerequisites

- Ansible 2.14+
- SSH access
- sudo privileges

**Note:- When deploying on Rocky Linux 8 or Red Hat Enterprise Linux 8, use Ansible versions 2.14 to 2.16, and ensure a compatible Python version is available on the target hosts to avoid Python and dnf module compatibility issues.**

## System Requirements

Refer to [PrivX Documentation](https://privx.docs.ssh.com/docs/deployment/preparing-for-deployment#system-requirements)

## Repository Structure

```text
.
├── deploy_privx.yml               # Main PrivX deployment
├── deploy_privx_poc.yml           # PrivX PoC single-node deployment
├── deploy_extender.yml            # Extender deployment
├── deploy_wag.yml                 # WAG deployment
├── uninstall.yml                  # PrivX uninstall playbook
├── inventory/hosts.ini            # Inventory configuration
├── configuration_files/           # Component configuration files
└── roles/                         # Ansible roles
    ├── install_privx/             # PrivX installation
    ├── configure_privx/           # PrivX configuration
    ├── configure_additional_nodes/ # Additional nodes setup
    ├── deploy_extender/           # Extender deployment
    └── deploy_wag/                # WAG deployment
```

## Inventory Configuration

Edit `inventory/hosts.ini` to configure your PrivX nodes and default values:

```ini
[privx_primary]
privx-node1.example.com

[privx_additional_nodes]
privx-node2.example.com
privx-node3.example.com

[all:vars]
# Optional configuration variables with defaults (customize as needed):
privx_ips=""
privx_superuser=admin
email_domain=example.com
database_name=privx
database_username=privx
database_port=5432
# Database-connection SSL mode: disable, require, verify-ca, verify-full
# verify-full recommended for production use
database_sslmode=require

# PostgreSQL database certificate or CA certificate in PEM format when database_sslmode is verify-ca or verify-full
postgres_ssl_cert_file=./privx-db-trust-anchor.pem

# Ansible connection settings
ansible_user=rocky
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

You can customize the default values directly in the inventory file, or override them via command line using `-e` parameters.

## Repository Control

The playbooks can skip managing external repositories and module streams when your hosts already have access to the required packages through internal mirrors or preconfigured repositories. Configure these in `inventory/hosts.ini` or override them with `-e`. If management is disabled and the required package is not available from the host's existing package sources, the package installation will fail normally.

Variables:

- `manage_ssh_product_repo=true`
- `manage_epel_repo=true`
- `manage_docker_ce_repo=true`
- `manage_postgresql_module=true`
- `custom_ssh_product_repo_url=""`
- `custom_ssh_product_repo_file=""`
- `ssh_product_repo_gpg_key_url="https://product-repository.ssh.com/info.fi-ssh.com-pubkey.asc"`

Examples:

```bash
ansible-playbook -i inventory deploy_privx.yml --tags install \
  -e manage_ssh_product_repo=false
```

```bash
ansible-playbook -i inventory deploy_privx.yml --tags install \
  -e manage_ssh_product_repo=false \
  -e manage_epel_repo=false \
  -e manage_postgresql_module=false
```

```bash
ansible-playbook -i inventory deploy_wag.yml \
  -e manage_ssh_product_repo=false \
  -e manage_epel_repo=false \
  -e manage_docker_ce_repo=false
```

## PrivX Core Deployment Options

### Unified Deployment (Main Approach)

Use the main deployment playbook with tags for flexible execution:

#### Complete Deployment
```bash
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

#### Installation Only
```bash
ansible-playbook -i inventory deploy_privx.yml --tags install
```

#### Configuration Only
```bash
ansible-playbook -i inventory deploy_privx.yml \
  --tags configure \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

#### Primary Node Only
```bash
ansible-playbook -i inventory deploy_privx.yml \
  --tags configure_primary \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

#### Additional Nodes Only
```bash
ansible-playbook -i inventory deploy_privx.yml --tags configure_additional
```

## PrivX Extender Installation and Configuration

The unified deployment supports PrivX Extender installation with automatic FIPS detection and configuration deployment.

### Prerequisites for Extender Deployment

1. **Download extender configuration files** from PrivX web interface:
   - Go to Administration → Deployment → Deploy PrivX VPC/VPN Extenders
   - Download configuration file for each extender
   - Save as `./configuration_files/<inventory_hostname>-extender-config.toml`

2. **Configure inventory** with extender nodes in `[privxextender]` group:
   ```ini
   [privxextender]
   extender-node1.example.com
   extender-node2.example.com
   extender-node3.example.com
   ```

3. **Set FIPS mode** (optional) for Red Hat/Rocky Linux 9:
   ```ini
   extender_fips_enabled=true
   ```

### Deploy Extenders

```bash
ansible-playbook -i inventory deploy_extender.yml
```

### Extender Features

- **Multi-OS Support**: Red Hat/Rocky Linux 8/9
- **FIPS Support**: Automatic FIPS package selection for RHEL/Rocky 9
- **Service Management**: Automatic service startup and enablement

## PrivX Web Access Gateway (WAG) Installation and Configuration

The automation supports PrivX Web Access Gateway installation, which consists of PrivX Carrier and Web Proxy components working as pairs.

### Prerequisites for WAG Deployment

1. **Download WAG configuration files** from PrivX web interface:
   - Go to Settings → Deployment → Deploy PrivX web-access gateways
   - Create or download a web-access-gateway configuration
   - Download Carrier Config and save as `./configuration_files/<inventory_hostname>-carrier-config.toml`
   - Download Proxy Config and save as `./configuration_files/<inventory_hostname>-web-proxy-config.toml`

2. **Configure inventory** with WAG nodes:
   ```ini
   [privxcarrier]
   # ⚠️ CRITICAL: List nodes in pairing sequence!
   carrier-node1.example.com
   carrier-node2.example.com

   [privxwebproxy]
   # ⚠️ CRITICAL: Must match Carrier node order!
   webproxy-node1.example.com  # Pairs with carrier-node1
   webproxy-node2.example.com  # Pairs with carrier-node2
   ```

   **Important**: Carrier and Web Proxy nodes work as pairs. List them in the same order in both groups - the first Carrier pairs with the first Web Proxy, second with second, etc.

### Deploy WAG Components

```bash
ansible-playbook -i inventory deploy_wag.yml
```

### WAG Features

- **Paired Components**: Carrier and Web Proxy work together for HTTP/HTTPS access
- **Multi-OS Support**: Red Hat/Rocky Linux 8/9
- **Firewall Configuration**: Automatic firewall setup for Web Proxy (ports 18080, 18443, 18444)
- **Service Management**: Automatic service startup and enablement

### WAG Network Requirements

- **Web Proxy Firewall Ports**: 18080, 18443, 18444 (automatically configured)
- **Carrier-Web Proxy Communication**: Ensure Carrier nodes can access Web Proxy nodes on required ports
- **Target Access**: Configure target HTTP/HTTPS hosts for access via WAG

## PrivX Core PoC Single Node Deployment (Evaluation Purpose Only)

The unified deployment supports PrivX Core Installation and Configuration for PoC/evaluation purposes only on single node/server with local PostgreSQL on same server.

### Default values used 
```ini
#Auto calculated
privx_dns_name: {{ ansible_facts['fqdn'] }} 
privx_ips: {{ ansible_facts['default_ipv4']['address'] }}

privx_superuser: admin
email_domain: example.com
database_name: privx
database_username: privx
```

### Unified Deployment

Configure node in `[privx_primary]` group for PoC single node deployment

#### Complete Deployment
```bash
ansible-playbook -i inventory deploy_privx_poc.yml \
  --tags full_deployment \
  -e database_password=your_password \
  -e superuser_password=your_admin_password
```
#### Installation Only
```bash
ansible-playbook -i inventory deploy_privx_poc.yml --tags install
```

#### Configuration Only
```bash
ansible-playbook -i inventory deploy_privx_poc.yml \
  --tags configure \
  -e database_password=your_password \
  -e superuser_password=your_admin_password
```

## PrivX Core Deployment Tag Reference

| Tag | Description | Use Case |
|-----|-------------|----------|
| `full_deployment` | Complete deployment | Production deployments |
| `install` | Installation only | Prepare nodes |
| `configure` | All configuration | Configure installed nodes |
| `configure_primary` | Primary node only | Single node or staged deployment |
| `configure_additional` | Additional nodes only | Add nodes to existing deployment |
| `deploy_ca_chain` | CA chain certificate deployment only | Deploy/update CA certificates |
| `summary` | Deployment summary | Status check |

For detailed deployment scenarios, see [DEPLOYMENT.md](DEPLOYMENT.md).

## OS-Specific Package Installation

The playbook automatically detects the operating system and installs appropriate packages for each supported distribution.

## Configuration Variables

### Required Variables

- `privx_dns_name`: DNS name for PrivX (e.g., privx.example.com)
- `database_password`: Password for the database user
- `postgres_address`: PostgreSQL server address
- `superuser_password`: Password for PrivX superuser account

### Optional Variables (with defaults in inventory)

These variables have default values set in `inventory/hosts.ini` and can be customized there or overridden via command line:

- `privx_ips`: IP addresses for PrivX nodes (default: "")
- `privx_superuser`: Superuser name (default: "admin")
- `email_domain`: Email domain (default: "example.com")
- `database_name`: Database name (default: "privx")
- `database_username`: Database username (default: "privx")
- `database_port`: Database port (default: "5432")
- `database_sslmode`: Database-connection SSL mode (default: "require")
- `postgres_ssl_cert_file`: PostgreSQL server certificate or CA certificate in PEM format 

### Auto-Calculated Variables

- `privx_lb`: Load balancer setting (0 for single node, 1 for multiple nodes)

### Connection Variables

- `ansible_user` (default: `root`)  
  SSH user for connecting to target servers

### Debug Variables

- `show_script_output` (default: `false`)  
  Display detailed script output during configuration and backup operations

### Version Variables

- `privx_version` (optional)  
  Install a specific PrivX and component version. Supports either a version prefix such as `42.2` or an exact package version such as `42.2-57`.  
  If not defined, installs latest available version  
  **Important**: When adding nodes/components to existing deployments, version consistency is automatically validated

## Safety Features

### Version Consistency Validation
- **Automatic version checking** when adding nodes or components to existing deployments
- **Prevents version mismatches** that could cause compatibility issues
- **Clear error messages** with exact commands to resolve version conflicts
- **Supports mixed scenarios** where some components exist and others are being added

### Staged Installation
- Primary node installed first, then secondary nodes
- Idempotent operations prevent duplicate installations

### Service Management
- Automatic service startup and enablement
- Service status verification on all nodes

### Idempotency
- Safe to re-run playbooks multiple times
- Checks existing installations before proceeding
- No duplicate installations or configuration changes

## Execution Examples

### Complete Deployment (Recommended)
```bash
# Deploy everything with unified playbook
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

### Complete Deployment with Debug Output
```bash
# Deploy with detailed script output for troubleshooting
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e show_script_output=true \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

### Complete Deployment with Specific Version
```bash
# Deploy specific PrivX version
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_version=42.2 \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

### Complete Deployment with Exact Package Version
```bash
# Deploy an exact PrivX package version
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_version=42.2-57 \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com \
  -e superuser_password=your_admin_password
```

### Staged Deployment
```bash
# Stage 1: Install all nodes
ansible-playbook -i inventory deploy_privx.yml --tags install

# Stage 2: Configure primary node
ansible-playbook -i inventory deploy_privx.yml \
  --tags configure_primary \
  -e privx_dns_name=privx.example.com \
  -e database_password=your_password \
  -e postgres_address=db.example.com

# Stage 3: Configure additional nodes
ansible-playbook -i inventory deploy_privx.yml --tags configure_additional
```

### Component-Specific Deployment
```bash
# Install extenders
ansible-playbook -i inventory deploy_extender.yml

# Install WAG components
ansible-playbook -i inventory deploy_wag.yml
```

## Uninstall

Use `uninstall.yml` to remove PrivX packages and related components from target hosts.

```bash
ansible-playbook -i inventory uninstall.yml
```

Review the playbook and inventory before running it, especially in shared or existing environments where hosts may contain components you want to preserve.


## Troubleshooting

### Common Issues
- **Repository/Package failures**: Check internet connectivity and DNS resolution
- **Service startup issues**: Verify system resources and firewall settings  
- **Configuration file not found**: Ensure config files are downloaded and placed in `./configuration_files/`
- **Version mismatch errors**: Use matching versions when adding components to existing deployments

### Quick Debug Commands
```bash
# Test connectivity
ansible -i inventory all -m ping

# Check installed versions
ansible -i inventory privx_primary -m shell -a "rpm -q PrivX"
ansible -i inventory privxextender -m shell -a "rpm -qa | grep PrivX-Extender"

# Enable debug output
ansible-playbook -i inventory deploy_privx.yml -e show_script_output=true [other options]
```

## Check if PostgreSQL connection is using SSL

When `database_sslmode` is set to `verify-ca` or `verify-full`, the `postgres_ssl_cert_file` variable must be configured with the path to a file containing the PostgreSQL server certificate or CA certificate in PEM format, which is required for SSL handshake validation.

### Prerequisites for PostgreSQL SSL handshake validation

1. **Place PostgreSQL server certificate or CA certificate in PEM format** in the file:
   - File name: privx-db-trust-anchor.pem (example)
   - Locaiton: Any (Same directory as the playbook files is prefered)

2. **Update postgres_ssl_cert_file variable**
   - For example : `postgres_ssl_cert_file=./privx-db-trust-anchor.pem`


## CA Chain Certificate Deployment (Optional)

The deployment supports optional CA chain certificate deployment for custom SSL certificates.

### Prerequisites for CA Chain Certificate

1. **Place CA chain certificate** in the playbook directory:
   - File name: `ca_chain.crt`
   - Location: Same directory as the playbook files

2. **Certificate requirements**:
   - **Single-server deployments**: Provide the CA chain of the PrivX-server certificate
   - **HA deployments**: Provide the CA chain of the load-balancer certificate

### CA Chain Certificate Deployment Process

The deployment automatically:
1. **Checks** for `./ca_chain.crt` file existence
2. **Creates** `/etc/nginx/ssl/` directory if needed
3. **Uploads** certificate to `/etc/nginx/ssl/ca_chain.crt`
4. **Updates** trust anchor using `/opt/privx/scripts/init_nginx.sh update-trust`
5. **Displays** deployment status and results

### Usage Examples

#### Deploy CA Chain Certificate Only
```bash
# Place your CA chain certificate
cp your-ca-chain.crt ./ca_chain.crt

# Deploy only CA chain certificate (after installation)
ansible-playbook -i inventory deploy_privx.yml --tags deploy_ca_chain
```

The CA chain certificate deployment is completely optional and will be skipped if the file doesn't exist.

## Design Principles

- **OS-agnostic**: Supports multiple Linux distributions
- **Safe for production**: Staged installation with validation gates
- **Idempotent**: Safe to re-run without side effects
- **Clear feedback**: Detailed status messages and error handling

## License

[![See LICENSE](https://img.shields.io/github/license/SSHcom/ansible-install-privx.svg?style=for-the-badge)](LICENSE)

## Support & Commercial Services
This is a public open-source project licensed under Apache 2.0 and not covered by standard support SLA. Community feedback and contributions are welcome. Support is provided on a best-effort basis only. For dedicated support, customisations, or enterprise assistance, please raise a ticket via our support portal ( https://care.ssh.com) or via your local support partner. Any requests would be assigned to your account manager.
