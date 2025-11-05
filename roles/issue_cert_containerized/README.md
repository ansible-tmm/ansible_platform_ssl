# issue_cert_containerized

Automatically issue Let's Encrypt SSL certificates for containerized Ansible Automation Platform (AAP 2.x+) installations using nginx as a reverse proxy with SSL termination.

## Overview

This role installs nginx with a valid Let's Encrypt SSL certificate and configures it as a reverse proxy to your existing AAP installation. AAP remains on its current port (typically 443), while nginx provides SSL termination on a custom port (default: 8444) with a valid certificate.

## Features

- ✅ Automatic Let's Encrypt certificate issuance via certbot
- ✅ Nginx reverse proxy with SSL termination
- ✅ WebSocket support for Automation Controller
- ✅ HTTP to HTTPS redirection
- ✅ Works with existing AAP installations without modification
- ✅ Simple configuration - no AAP reconfiguration needed

## Requirements

- Ansible Automation Platform 2.x+ (containerized installation)
- RHEL 8/9
- DNS record pointing to your server
- Port 80 accessible for Let's Encrypt validation
- Custom port available for nginx (default: 8444)

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `dns_name` | FQDN for your AAP installation | `ansible.example.com` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_frontend_port` | `8444` | Port where nginx listens for HTTPS with valid cert |
| `aap_backend_port` | `443` | Port where AAP is currently running |
| `letsencrypt_email` | `ansible-network@redhat.com` | Email for Let's Encrypt notifications |
| `configure_selinux` | `true` | Configure SELinux for nginx proxy |
| `aap_user` | `ec2-user` | User running AAP (for podman installations) |

## Usage

### Standard Deployment (AAP on Port 443)

This is the default and simplest configuration:

```yaml
- hosts: localhost
  become: true
  roles:
    - role: issue_cert_containerized
      vars:
        dns_name: "ansible.example.com"
```

**Result:** Users access AAP at `https://ansible.example.com:8444` with a valid Let's Encrypt certificate.

### Custom Ports

If you installed AAP with custom ports, adjust the variables:

```yaml
- hosts: localhost
  become: true
  roles:
    - role: issue_cert_containerized
      vars:
        dns_name: "ansible.example.com"
        nginx_frontend_port: 9443      # Custom nginx port
        aap_backend_port: 8501         # If AAP was installed on custom port
```

## What This Role Does

1. **Installs** nginx and certbot packages
2. **Configures** SELinux for nginx proxy connections
3. **Obtains** Let's Encrypt SSL certificate via certbot
4. **Configures** nginx as reverse proxy to AAP
5. **Enables** HTTP → HTTPS redirection
6. **Verifies** SSL connectivity

## Architecture

```
[Internet] → https://hostname:8444 (nginx with Let's Encrypt cert)
                 ↓
              [nginx reverse proxy]
                 ↓
            https://localhost:443 (AAP)
```

## Advanced Configuration

### For Fresh AAP Installations

If you're installing AAP from scratch and want to use standard port 443 with a valid certificate, you can:

1. Install AAP with a custom port during setup:
   ```ini
   # In your AAP installer inventory
   envoy_https_port=8501
   ```

2. Then configure this role to match:
   ```yaml
   vars:
     nginx_frontend_port: 443       # Nginx takes standard HTTPS port
     aap_backend_port: 8501         # AAP on custom port
   ```

This gives you `https://hostname` (no port number) with a valid certificate!

## Example Playbook

See `update_cert_containerized.yml` in the project root for a complete example.

## Testing

```bash
ansible-playbook update_cert_containerized.yml
```

## Firewall Configuration

Make sure your firewall allows:
- Port 80 (for Let's Encrypt validation)
- Port 8444 (or your custom `nginx_frontend_port`)

Example for firewalld:
```bash
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=8444/tcp
firewall-cmd --reload
```

## Troubleshooting

### Certificate Validation Fails

Ensure port 80 is accessible from the internet for Let's Encrypt validation.

### Connection Refused on Custom Port

Check that the `nginx_frontend_port` is not already in use:
```bash
ss -tlnp | grep :8444
```

### SELinux Denials

If you encounter SELinux issues, check:
```bash
ausearch -m avc -ts recent | grep nginx
```

## License

MIT

## Author

IPvSean
