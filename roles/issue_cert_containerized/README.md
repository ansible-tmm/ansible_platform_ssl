# issue_cert_containerized

Automatically issue Let's Encrypt SSL certificates for containerized Ansible Automation Platform (AAP 2.x+) installations using nginx as a reverse proxy.

## Features

- ✅ Automatic Let's Encrypt certificate issuance via certbot
- ✅ Nginx reverse proxy with SSL termination
- ✅ WebSocket support for Automation Controller
- ✅ Optional envoy port modification for brownfield environments
- ✅ HTTP to HTTPS redirection
- ✅ Automatic backup of original configurations

## Requirements

- Ansible Automation Platform 2.x+ (containerized installation)
- RHEL 8/9
- DNS record pointing to your server
- Port 80 accessible for Let's Encrypt validation

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `dns_name` | FQDN for your AAP installation | `ansible.example.com` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `modify_envoy` | `false` | Set to `true` to move AAP to `aap_backend_port` |
| `nginx_frontend_port` | `443` | Port where nginx listens for HTTPS |
| `aap_backend_port` | `8501` | Backend port for AAP (if `modify_envoy: true`) |
| `letsencrypt_email` | `ansible-network@redhat.com` | Email for Let's Encrypt notifications |
| `configure_selinux` | `true` | Configure SELinux for nginx proxy |

## Usage Scenarios

### Scenario 1: Brownfield AAP on Port 443 (Recommended)

Move AAP to port 8501, nginx takes over 443:

```yaml
- hosts: localhost
  become: true
  roles:
    - role: issue_cert_containerized
      vars:
        dns_name: "ansible.example.com"
        modify_envoy: true
```

### Scenario 2: Keep AAP on Current Port

AAP stays on 443, nginx also configured for 443:

```yaml
- hosts: localhost
  become: true
  roles:
    - role: issue_cert_containerized
      vars:
        dns_name: "ansible.example.com"
        modify_envoy: false
        aap_backend_port: 443
```

## What This Role Does

1. **Detects** current AAP configuration
2. **Optionally modifies** envoy port (if `modify_envoy: true`)
3. **Installs** nginx and certbot
4. **Obtains** Let's Encrypt certificate
5. **Configures** nginx as reverse proxy
6. **Verifies** SSL connectivity

## Example Playbook

See `update_cert_containerized.yml` in the project root for a complete example.

## Testing

```bash
ansible-playbook update_cert_containerized.yml
```

## License

MIT

## Author

IPvSean
