# K3s Control Plane Role

Ansible role for deploying and managing K3s Kubernetes control plane nodes in a homelab environment.

## Overview

This role handles the complete lifecycle of K3s control plane nodes:
- **Installation** - Downloads and installs K3s server components
- **Configuration** - Sets up control plane with SQLite datastore
- **Token Management** - Extracts and shares node tokens for worker nodes
- **Service Management** - Manages K3s systemd service
- **Cleanup** - Complete removal of K3s installation when needed

## Requirements

### System Requirements
- **OS**: Ubuntu 20.04+ or compatible Debian-based system
- **Privileges**: `become: true` (sudo access required)
- **Network**: Port 6443 accessible for Kubernetes API
- **Storage**: Sufficient disk space for container images and SQLite database

### Dependencies
- **Ansible Collections**: Standard `ansible.builtin` modules
- **Network Access**: Internet connectivity for downloading K3s installer
- **System Packages**: `curl`, `apt-transport-https` (installed automatically)

## Role Variables

### Required Variables
```yaml
# No required variables - role uses sensible defaults
```

### Optional Variables
```yaml
# K3s state management
k3s_state: present          # present|absent - default: present

# K3s configuration (defined in your inventory)
k3s_version: v1.33.1+k3s1   # K3s version to install
k3s_channel: stable         # Release channel
```

### Auto-Generated Variables
```yaml
k3s_token: "{{ node_token.stdout }}"  # Generated during installation
```

## Example Usage

### Basic Deployment
```yaml
- name: Deploy K3s Control Plane
  hosts: k3s_control_plane
  become: true
  tasks:
    - include_role:
        name: k3s_control_plane
      tags: [k3s, control_plane]
```

### With State Management
```yaml
# Install K3s
- name: Install K3s Control Plane
  hosts: k3s_control_plane
  become: true
  vars:
    k3s_state: present
  tasks:
    - include_role:
        name: k3s_control_plane

# Remove K3s
- name: Remove K3s Control Plane
  hosts: k3s_control_plane
  become: true
  vars:
    k3s_state: absent
  tasks:
    - include_role:
        name: k3s_control_plane
```

## Available Tags

- `k3s` - All K3s related tasks
- `control_plane` - Control plane specific tasks
- `always` - Debug and status messages
- `debug` - Detailed debugging information

## Features

### ✅ Installation Features
- **Automatic Download** - Fetches latest K3s installer script
- **Prerequisites** - Installs required system packages
- **Networking Tools** - Installs diagnostic utilities for troubleshooting
- **SQLite Datastore** - Uses embedded SQLite for single-node control plane
- **Service Management** - Configures systemd service with auto-start
- **Health Checks** - Waits for API server to be ready on port 6443

### ✅ Calico CNI Integration
- **Flannel Replacement** - Automatically disables built-in Flannel CNI
- **Tigera Operator** - Deploys Calico using the official Tigera operator
- **Intra-Pod Networking** - Enables proper localhost connectivity for multi-container pods
- **Advanced NetworkPolicies** - Supports enterprise-grade network security policies
- **BGP Routing** - Enables efficient Layer 3 networking where possible

### ✅ Token Management
- **Extraction** - Retrieves node token from control plane
- **Storage** - Makes token available to worker nodes via Ansible facts
- **Security** - Token is securely generated by K3s

### ✅ Cleanup Features
- **Complete Removal** - Removes all K3s components when `k3s_state: absent`
- **Service Cleanup** - Stops and disables systemd services
- **Directory Cleanup** - Removes configuration and data directories
- **Binary Removal** - Removes K3s executable

### ✅ Debugging
- **Comprehensive Logging** - Debug messages at each step
- **State Validation** - Shows how state variables are interpreted
- **Error Handling** - Graceful error handling with `ignore_errors`

### ✅ Firewall Configuration
- **UFW Integration** - Automatically configures firewall rules for K3s
- **Port Management** - Opens required ports (6443, 10250, Calico networking)
- **Network Security** - Allows pod and service network CIDRs
- **Node Communication** - Enables inter-node cluster communication

#### Required Ports
```bash
# Control Plane Ports
6443/tcp    # Kubernetes API server
10250/tcp   # Kubelet API
179/tcp     # Calico BGP routing
4789/udp    # Calico VXLAN overlay
5473/tcp    # Calico Typha (if enabled)
9100/tcp    # Node exporter (monitoring)

# Service Access
30000-32767/tcp  # NodePort range

# Internal Networks (Calico)
10.244.0.0/16    # Pod network CIDR
10.96.0.0/12     # Service network CIDR
```

#### Firewall Tasks
```yaml
# Automatic UFW configuration
- name: Configure UFW for K3s
  include_tasks: firewall.yml
  when: configure_ufw | default(true)
  tags: [k3s, firewall]
```

## Architecture

### K3s Configuration
```yaml
# Control plane setup
INSTALL_K3S_EXEC: --datastore-endpoint='sqlite:///var/lib/rancher/k3s/server.db'
```

### File Locations
```bash
# Binaries
/usr/local/bin/k3s                    # Main K3s binary
/usr/local/bin/k3s-uninstall.sh      # Uninstall script

# Configuration
/etc/rancher/k3s/                     # Configuration directory
/var/lib/rancher/k3s/                 # Data directory
/var/lib/rancher/k3s/server/node-token # Node token file

# Kubernetes data
/var/lib/kubelet/                     # Kubelet data
/var/log/pods/                        # Pod logs
/var/log/containers/                  # Container logs
```

### Network Ports
- **6443** - Kubernetes API server
- **10250** - Kubelet API (internal)
- **179** - Calico BGP routing protocol
- **4789** - Calico VXLAN overlay networking
- **5473** - Calico Typha scaling component

## Integration with Homelab

### Inventory Configuration
```ini
[k3s_control_plane]
k3s-control-1 ansible_host=192.168.1.10 ansible_user=user
k3s-control-2 ansible_host=192.168.1.11 ansible_user=user  # HA setup
```

### Variables Integration
```yaml
# From inventory/production/group_vars/all.yml
k3s_version: v1.33.1+k3s1
k3s_channel: stable
control_plane_host: "{{ groups['k3s_control_plane'][0] }}"
control_plane_ip: "{{ hostvars[groups['k3s_control_plane'][0]]['ansible_host'] }}"
```

## Command Line Usage

```bash
# Deploy control plane
ansible-playbook site.yml --tags="k3s,control_plane"

# Debug deployment
ansible-playbook site.yml --tags="k3s" -v

# Remove control plane
ansible-playbook site.yml -e "k3s_state=absent" --tags="k3s"

# Target specific control plane node
ansible-playbook site.yml --limit="k3s_control_plane" --tags="k3s"
```

## Troubleshooting

### Common Issues

**K3s fails to start:**
```bash
# Check service status
sudo systemctl status k3s

# Check logs
sudo journalctl -u k3s -f

# Verify port availability
sudo netstat -tlnp | grep 6443
```

**API server not accessible:**
```bash
# Test connectivity
curl -k https://localhost:6443/version

# Check firewall
sudo ufw status
```

**Token not found:**
```bash
# Manual token retrieval
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Debug Mode
```bash
# Run with maximum verbosity
ansible-playbook site.yml --tags="k3s,debug" -vvv
```

## Dependencies

### Before This Role
- Target hosts must be accessible via SSH
- User must have sudo privileges
- Basic networking must be configured

### After This Role
- Workers can join using the generated token
- Kubeconfig can be fetched for cluster access
- Additional Kubernetes components can be deployed

## Security Considerations

- ✅ **Token Security** - Node tokens are handled securely via Ansible facts
- ✅ **Service Hardening** - K3s runs with appropriate systemd security
- ✅ **Network Security** - Only necessary ports are used
- ⚠️ **SQLite** - Single point of failure (suitable for homelab, not production)

## License

MIT

## Author Information

K3s Homelab MLOps Platform - Infrastructure as Code
