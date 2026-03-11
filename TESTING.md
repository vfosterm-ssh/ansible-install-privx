# PrivX Deployment Testing Guide

This guide provides instructions for testing the PrivX deployment playbooks in different environments.

## Test Environment Setup

### Minimum Test Environment
- 1 primary node (for basic functionality testing)
- 1 secondary node (for validation gate testing)

### Recommended Test Environment
- 1 primary node
- 2-3 secondary nodes (to test parallel installation)

## Pre-Test Checklist

### Infrastructure Requirements
- [ ] Target servers are accessible via SSH
- [ ] Servers have internet connectivity
- [ ] Servers have sufficient resources (CPU, RAM, disk)
- [ ] DNS resolution is working
- [ ] Firewall allows necessary traffic

### Ansible Requirements
- [ ] Ansible is installed (version 2.14+)
- [ ] SSH keys are configured for passwordless access
- [ ] Inventory file is properly configured
- [ ] Target user has sudo privileges

## Test Scenarios

### Scenario 1: Fresh Deployment
Test complete deployment on clean systems.

```bash
# Run complete deployment
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=test.example.com \
  -e database_password=test_password \
  -e postgres_address=localhost \
  -e superuser_password=test_admin_password
```

**Expected Results:**
- All repositories configured correctly
- PrivX installed on all nodes
- Services running and enabled
- No errors in playbook execution

### Scenario 2: Idempotency Test
Test that re-running playbooks doesn't cause issues.

```bash
# Run deployment twice
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=test.example.com \
  -e database_password=test_password \
  -e postgres_address=localhost \
  -e superuser_password=test_admin_password

ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=test.example.com \
  -e database_password=test_password \
  -e postgres_address=localhost \
  -e superuser_password=test_admin_password
```

**Expected Results:**
- Second run shows "no changes needed"
- No errors or warnings
- Services remain stable

### Scenario 3: Staged Deployment Test
Test deploying primary and additional nodes separately.

```bash
# Step 1: Install all nodes
ansible-playbook -i inventory deploy_privx.yml --tags install

# Step 2: Configure primary node
ansible-playbook -i inventory deploy_privx.yml \
  --tags configure_primary \
  -e privx_dns_name=test.example.com \
  -e database_password=test_password \
  -e postgres_address=localhost \
  -e superuser_password=test_admin_password

# Step 3: Configure additional nodes
ansible-playbook -i inventory deploy_privx.yml --tags configure_additional
```

**Expected Results:**
- Installation completes successfully
- Primary node configures successfully
- Additional nodes configure successfully
- All services running on all nodes

### Scenario 4: Extender Deployment Test
Test PrivX Extender deployment.

```bash
# Deploy extenders
ansible-playbook -i inventory deploy_extender.yml
```

**Expected Results:**
- Extender packages installed correctly
- FIPS mode detected and configured (if applicable)
- Configuration files deployed
- Services running and enabled

### Scenario 5: WAG Deployment Test
Test PrivX Web Access Gateway deployment.

```bash
# Deploy WAG components
ansible-playbook -i inventory deploy_wag.yml
```

**Expected Results:**
- Carrier and Web Proxy packages installed correctly
- Configuration files deployed
- Firewall rules configured for Web Proxy
- Services running and enabled

### Scenario 6: OS-Specific Testing
Test on different operating systems.

**Test Matrix:**
- Red Hat Enterprise Linux 9
- Rocky Linux 9
- Red Hat Enterprise Linux 8
- Rocky Linux 8
- Amazon Linux 2023

**Validation Points:**
- Correct repositories configured
- Appropriate packages installed
- OS-specific commands executed correctly

## Verification Commands

### Repository Verification
```bash
# Check repository configuration
ansible -i inventory all -m shell -a "ls -la /etc/yum.repos.d/ssh-products.repo"

# Verify GPG key import
ansible -i inventory all -m shell -a "rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | grep -i ssh"
```

### Package Verification
```bash
# Check PrivX installation
ansible -i inventory privx_primary -m shell -a "rpm -q PrivX"
ansible -i inventory privx_additional_nodes -m shell -a "rpm -q PrivX"

# Check Extender installation
ansible -i inventory privxextender -m shell -a "rpm -qa | grep PrivX-Extender"

# Check WAG installation
ansible -i inventory privxcarrier -m shell -a "rpm -q PrivX-Carrier"
ansible -i inventory privxwebproxy -m shell -a "rpm -q PrivX-Web-Proxy"
```

### Service Verification
```bash
# Check PrivX service status
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "systemctl is-active privx"
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "systemctl is-enabled privx"

# Check Extender service status
ansible -i inventory privxextender -m shell -a "systemctl is-active privx-extender"

# Check WAG service status
ansible -i inventory privxcarrier -m shell -a "systemctl is-active privx-carrier"
ansible -i inventory privxwebproxy -m shell -a "systemctl is-active privx-web-proxy"

# Check service logs
ansible -i inventory privx_primary -m shell -a "journalctl -u privx --no-pager -n 20"
```

## Test Data Collection

### Log Collection
```bash
# Collect Ansible logs
ansible-playbook -i inventory deploy_privx.yml --tags full_deployment -v > deployment.log 2>&1

# Collect system logs
ansible -i inventory all -m shell -a "journalctl --since '1 hour ago' --no-pager" > system_logs.txt
```

### System Information
```bash
# Collect OS information
ansible -i inventory all -m setup -a "filter=ansible_distribution*" > os_info.json

# Collect package information
ansible -i inventory all -m shell -a "rpm -qa | grep -E '(PrivX|postgresql|firewalld)'" > packages.txt
```

## Common Test Issues

### Repository Issues
- **Symptom**: Package not found errors
- **Check**: Internet connectivity, DNS resolution
- **Fix**: Verify repository URLs and GPG keys

### Permission Issues
- **Symptom**: Permission denied errors
- **Check**: SSH access, sudo privileges
- **Fix**: Verify REMOTE_USER and BECOME settings

### Service Issues
- **Symptom**: Service fails to start
- **Check**: System resources, port conflicts
- **Fix**: Review service logs, check firewall

## Test Environment Cleanup

### Remove PrivX Deployment
```bash
# Stop PrivX services
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "systemctl stop privx"

# Stop Extender services
ansible -i inventory privxextender -m shell -a "systemctl stop privx-extender"

# Stop WAG services
ansible -i inventory privxcarrier -m shell -a "systemctl stop privx-carrier"
ansible -i inventory privxwebproxy -m shell -a "systemctl stop privx-web-proxy"

# Remove packages
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "dnf remove -y PrivX"
ansible -i inventory privxextender -m shell -a "dnf remove -y PrivX-Extender*"
ansible -i inventory privxcarrier -m shell -a "dnf remove -y PrivX-Carrier"
ansible -i inventory privxwebproxy -m shell -a "dnf remove -y PrivX-Web-Proxy"
```

### Reset Repositories (Optional)
```bash
# Remove SSH.COM repository
ansible -i inventory all -m shell -a "rm -f /etc/yum.repos.d/ssh-products.repo"

# Remove GPG keys (if needed)
ansible -i inventory all -m shell -a "rpm -e gpg-pubkey-<keyid>"
```

## Automated Testing

### Basic Smoke Test Script
```bash
#!/bin/bash
# smoke_test.sh

echo "Running PrivX deployment smoke test..."

# Test connectivity
ansible -i inventory all -m ping || exit 1

# Run complete deployment
ansible-playbook -i inventory deploy_privx.yml \
  --tags full_deployment \
  -e privx_dns_name=test.example.com \
  -e database_password=test_password \
  -e postgres_address=localhost \
  -e superuser_password=test_admin_password || exit 1

# Verify installation
ansible -i inventory privx_primary:privx_additional_nodes -m shell -a "rpm -q PrivX" || exit 1

echo "Smoke test completed successfully!"
```

## Reporting Issues

When reporting issues, include:
- Operating system and version
- Ansible version
- Complete error messages
- Relevant log files
- Steps to reproduce

## Performance Testing

### Deployment Time Benchmarks
- Single node: ~5-10 minutes
- Multi-node (4 nodes): ~10-15 minutes
- Extender deployment: ~3-5 minutes per node
- WAG deployment: ~5-8 minutes per pair
- Network-dependent for package downloads

### Resource Usage
- Minimal during deployment
- Monitor disk space for package downloads
- Check memory usage during service startup