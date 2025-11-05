# Ansible Automation Platform SSL Certificate Management

Automated SSL certificate issuance and management for Ansible Automation Platform using Let's Encrypt.

## Table of Contents

- [Ansible Automation Platform SSL Certificate Management](#ansible-automation-platform-ssl-certificate-management)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [How It Works](#how-it-works)
  - [Roles](#roles)
    - [1. `issue_cert` - RPM-Based AAP Installations](#1-issue_cert---rpm-based-aap-installations)
    - [2. `issue_cert_containerized` - Containerized AAP Installations](#2-issue_cert_containerized---containerized-aap-installations)
  - [Quick Start](#quick-start)
    - [Prerequisites](#prerequisites)
    - [For RPM-Based AAP](#for-rpm-based-aap)
    - [For Containerized AAP](#for-containerized-aap)
  - [Architecture](#architecture)
    - [RPM-Based Deployment](#rpm-based-deployment)
    - [Containerized Deployment](#containerized-deployment)
  - [Important Notes](#important-notes)
    - [Certificate Validation](#certificate-validation)
    - [Chrome "Not Secure" Warning (Containerized Role)](#chrome-not-secure-warning-containerized-role)
  - [Requirements](#requirements)
  - [Support Matrix](#support-matrix)
  - [Firewall Configuration](#firewall-configuration)
    - [For RPM-Based Role](#for-rpm-based-role)
    - [For Containerized Role](#for-containerized-role)
  - [Certificate Renewal](#certificate-renewal)
  - [Troubleshooting](#troubleshooting)
    - [Certificate Validation Fails](#certificate-validation-fails)
    - [SELinux Issues](#selinux-issues)
  - [Examples](#examples)
    - [RPM-Based Example](#rpm-based-example)
    - [Containerized Example](#containerized-example)
  - [Contributing](#contributing)
  - [License](#license)
  - [Contact](#contact)
  - [Acknowledgments](#acknowledgments)

## Overview

This repository provides Ansible roles to automatically obtain and configure valid SSL certificates from Let's Encrypt for Ansible Automation Platform (AAP) installations. Two different roles are provided to support different AAP deployment types.

## How It Works

**Important:** These roles **do not modify your AAP installation directly**. Instead, they:

1. **Install nginx** as a separate web server on your system
2. **Obtain Let's Encrypt certificates** using certbot
3. **Configure nginx as a reverse proxy** with SSL termination
4. **Forward requests** from nginx to your existing AAP installation

This approach means:
- ✅ **No AAP modification required** - Your AAP installation remains unchanged
- ✅ **No installer re-run needed** - Works with existing AAP deployments
- ✅ **Non-invasive** - nginx handles SSL, AAP continues working as-is
- ✅ **Easy to remove** - Simply stop nginx to revert to original AAP access

The nginx proxy sits in front of AAP, terminates SSL connections with valid certificates, and forwards traffic to AAP's existing ports.

## Roles

### 1. `issue_cert` - RPM-Based AAP Installations

For traditional **RPM-based AAP installations** on RHEL where AAP components are installed via RPM packages.

**Use Case:**
- AAP installed via RPM packages
- System-wide installation
- AAP services managed by systemd directly

**Documentation:** [roles/issue_cert/README.md](roles/issue_cert/README.md) *(if available)*

**Example Usage:**
```bash
ansible-playbook update_cert.yml
```

See [update_cert.yml](update_cert.yml) for configuration.

---

### 2. `issue_cert_containerized` - Containerized AAP Installations

For **containerized AAP installations** on RHEL using podman.

**Use Case:**
- AAP installed via containerized deployment
- User-based podman installations (common on AWS, Azure, etc.)
- AAP 2.x+ with gateway proxy architecture

**Documentation:** [roles/issue_cert_containerized/README.md](roles/issue_cert_containerized/README.md)

**Example Usage:**
```bash
ansible-playbook update_cert_containerized.yml
```

See [update_cert_containerized.yml](update_cert_containerized.yml) for configuration.

---

## Quick Start

### Prerequisites

Install required Ansible collections:
```bash
ansible-galaxy collection install ansible.posix
```

### For RPM-Based AAP

1. Edit `update_cert.yml` with your DNS name
2. Run the playbook:
   ```bash
   ansible-playbook update_cert.yml
   ```

### For Containerized AAP

1. Edit `update_cert_containerized.yml` with your DNS name
2. Run the playbook:
   ```bash
   ansible-playbook update_cert_containerized.yml
   ```

## Architecture

### RPM-Based Deployment
```
Internet → Nginx Proxy (with Let's Encrypt cert) → AAP Components
```

**What happens:**
1. Nginx installed on the system (separate from AAP)
2. Let's Encrypt certificate obtained and configured in nginx
3. Nginx acts as reverse proxy to existing AAP installation
4. AAP components remain unchanged

### Containerized Deployment
```
Internet → Nginx (Port 8444) with Let's Encrypt → AAP Gateway Proxy (Port 443)
```

**What happens:**
1. Nginx installed on the system (separate from AAP containers)
2. Let's Encrypt certificate obtained and configured in nginx
3. Nginx listens on port 8444 with valid SSL certificate
4. Nginx proxies requests to AAP gateway on port 443
5. AAP containers remain unchanged

## Important Notes

### Certificate Validation

Both roles use Let's Encrypt for certificate issuance, which requires:
- Port 80 accessible from the internet (for HTTP-01 challenge)
- Valid DNS record pointing to your server
- Server must be reachable at the specified DNS name

### Chrome "Not Secure" Warning (Containerized Role)

When using the containerized role with the default port 8444, Chrome may display a "Not Secure" warning even though the certificate is valid and the connection is encrypted. This is a Chrome security policy for non-standard HTTPS ports.

**What this means:**
- ✅ Your connection IS secure and encrypted
- ✅ Certificate IS valid (verified by Let's Encrypt)
- ⚠️ Chrome flags non-standard ports as "Not Secure"
- ✅ Firefox and other browsers show it as secure

**Options:**
1. **Accept the warning** - Connection is secure, just Chrome being cautious (Recommended for existing installations)
2. **Use port 443** - Requires AAP reinstallation with custom ports

See the [containerized role README](roles/issue_cert_containerized/README.md) for details on configuring standard ports.

## Requirements

- Ansible 2.9+
- RHEL 8 or 9
- Ansible Automation Platform 2.x+
- DNS configured and pointing to your server
- Internet access for Let's Encrypt validation
- Root/sudo access

## Support Matrix

| AAP Version | RPM Role | Containerized Role |
|-------------|----------|-------------------|
| AAP 2.5     | ✅       | ✅                |
| AAP 2.4     | ✅       | ✅                |
| AAP 2.3     | ✅       | ✅                |

## Firewall Configuration

### For RPM-Based Role
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### For Containerized Role
```bash
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8444/tcp
firewall-cmd --reload
```

## Certificate Renewal

Let's Encrypt certificates are valid for 90 days. Certbot automatically sets up a renewal cron job/timer.

To manually renew:
```bash
sudo certbot renew
```

## Troubleshooting

### Certificate Validation Fails

Ensure:
- Port 80 is accessible from the internet
- DNS is properly configured
- No firewall blocking Let's Encrypt validation

### SELinux Issues

The roles handle SELinux configuration automatically. If you encounter issues:
```bash
# Check for SELinux denials
sudo ausearch -m avc -ts recent | grep nginx

# View SELinux context
sudo ls -lZ /etc/nginx/ssl/
```

## Examples

### RPM-Based Example
```yaml
---
- name: Update SSL cert
  hosts: localhost
  connection: local
  become: true
  gather_facts: false
  tasks:
    - name: Update the cert
      ansible.builtin.include_role:
        name: issue_cert
      vars:
        issue_cert_dns_name: "ansible.example.com"
```

### Containerized Example
```yaml
---
- name: Issue SSL certificate for containerized AAP
  hosts: localhost
  connection: local
  become: true
  gather_facts: true
  tasks:
    - name: Issue Let's Encrypt certificate for AAP
      ansible.builtin.include_role:
        name: issue_cert_containerized
      vars:
        dns_name: "ansible.example.com"
        # Optional: nginx_frontend_port: 8444
        # Optional: aap_backend_port: 443
```

## Contributing

Issues and pull requests are welcome! Please ensure:
- Code follows existing patterns
- Documentation is updated
- Testing is performed on RHEL 8/9

## License

MIT

## Contact

**Owner:** IPvSean
**Email:** ansible-tmm@redhat.com
**Team:** Ansible Technical Marketing Team

## Acknowledgments

This project is created and maintained by the Red Hat Ansible Technical Marketing Team to help customers easily deploy secure AAP installations with valid SSL certificates.

---

**Need help?** Open an issue or contact ansible-tmm@redhat.com
