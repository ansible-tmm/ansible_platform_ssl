# Ansible Automation Platform SSL Certificate Management

Automated SSL certificate issuance and management for Ansible Automation Platform using Let's Encrypt.

## Overview

This repository provides Ansible roles to automatically obtain and configure valid SSL certificates from Let's Encrypt for Ansible Automation Platform (AAP) installations. Two different roles are provided to support different AAP deployment types.

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
Internet → Let's Encrypt Certificate → AAP (Port 443)
```

The role obtains a certificate and installs it directly into AAP's nginx configuration.

### Containerized Deployment
```
Internet → Nginx (Port 8444) with Let's Encrypt → AAP Gateway Proxy (Port 443)
```

The role installs a separate nginx instance with SSL termination that proxies to the AAP gateway.

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
