# Configuration Files Directory

This directory contains configuration files for PrivX components.

## Purpose

Configuration files for PrivX components (Extenders, Web Access Gateway) are stored here following established naming conventions.

## Current Usage

### PrivX Extender Configuration Files
- **Naming convention**: `<inventory_hostname>-extender-config.toml`
- **Example**: `extender-node1.example.com-extender-config.toml`
- **Source**: Downloaded from PrivX UI (Administration → Deployment → Deploy PrivX VPC/VPN Extenders)
- **Required**: Before running extender installation

### PrivX Web Access Gateway Configuration Files

#### Carrier Configuration Files
- **Naming convention**: `<inventory_hostname>-carrier-config.toml`
- **Example**: `carrier-node1.example.com-carrier-config.toml`
- **Source**: Downloaded from PrivX UI (Settings → Deployment → Deploy PrivX web-access gateways → Download Carrier Config)
- **Required**: Before running WAG installation for Carrier nodes

#### Web Proxy Configuration Files
- **Naming convention**: `<inventory_hostname>-web-proxy-config.toml`
- **Example**: `webproxy-node1.example.com-web-proxy-config.toml`
- **Source**: Downloaded from PrivX UI (Settings → Deployment → Deploy PrivX web-access gateways → Download Proxy Config)
- **Required**: Before running WAG installation for Web Proxy nodes

## Configuration File Sources

### PrivX Extender
1. Access PrivX web interface
2. Go to Administration → Deployment → Deploy PrivX VPC/VPN Extenders
3. Create or download extender configuration
4. Save as `./configuration_files/<hostname>-extender-config.toml`

### PrivX Web Access Gateway
1. Access PrivX web interface
2. Go to Settings → Deployment → Deploy PrivX web-access gateways
3. Create or download web-access-gateway configuration
4. Download Carrier Config and save as `./configuration_files/<hostname>-carrier-config.toml`
5. Download Proxy Config and save as `./configuration_files/<hostname>-web-proxy-config.toml`

**Important**: For WAG pairs, use the same web-access-gateway configuration from PrivX UI for both the Carrier and Web Proxy components that will work together.

## File Structure Example

```
configuration_files/
├── README.md
├── extender-node1.example.com-extender-config.toml
├── extender-node2.example.com-extender-config.toml
├── carrier-node1.example.com-carrier-config.toml      # WAG Pair 1
├── webproxy-node1.example.com-web-proxy-config.toml   # WAG Pair 1
├── carrier-node2.example.com-carrier-config.toml      # WAG Pair 2
└── webproxy-node2.example.com-web-proxy-config.toml   # WAG Pair 2
```

**WAG Pairing Example:**
- `carrier-node1` + `webproxy-node1` = WAG Pair 1 (use same PrivX web-access-gateway config)
- `carrier-node2` + `webproxy-node2` = WAG Pair 2 (use same PrivX web-access-gateway config)

## Important Notes

- Configuration files must be downloaded from PrivX UI after core PrivX installation
- File names must match the inventory hostnames exactly
- Configuration files are validated before deployment
- WAG components (Carrier and Web Proxy) work as pairs for HTTP/HTTPS access
