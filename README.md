# Ansible Role: Zot OCI Registry (`clawduino.zot_registry`)

[![Ansible Lint](https://img.shields.io/badge/ansible--lint-production-green.svg)](https://github.com/ansible/ansible-lint)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

An enterprise-ready Ansible role for deploying the high-performance [Zot OCI Registry](https://github.com/project-zot/zot) natively inside a Linux Container (LXC) or virtual machine, backed by **IDrive e2** S3-compatible cloud object storage.

---

## Features

- **Native Binary Execution**: Runs upstream `zot` as a hardened systemd service, avoiding nested container runtime overhead (e.g. Docker-in-LXC) and storage driver incompatibilities on ZFS/btrfs root filesystems.
- **IDrive e2 S3 Storage**: Native support for S3 object storage with path-style addressing (`forcepathstyle: true`), HTTPS transport, and content-addressable deduplication.
- **Lifecycle & Retention**: Configurable image retention policies (keeping latest *N* tags per repository) and automated daily background garbage collection (`gc: true`).
- **Security & Sandboxing**: Dedicated system user/group (`zot:zot`), isolated credentials in `/etc/zot/zot.env` (`0600`), and LXC-compatible systemd sandboxing.
- **Nginx Proxy Manager Ready**: Binds plain HTTP to port `8585` by default, designed for upstream SSL/TLS termination, custom domains, and access lists via Nginx Proxy Manager.
- **Access Control Matrix**: Anonymous image pulls (public read within private networks) coupled with bcrypt-hashed `htpasswd` credentials for authenticated pushes and deletions.
- **Full Extensions**: Includes embedded Web UI, Prometheus metrics endpoint (`/metrics`), GraphQL search, image scrub integrity verification, and optional pull-through mirror caching (e.g., Docker Hub, GHCR).

---

## Requirements

- **Ansible**: `>= 2.14`
- **Target OS**: Debian 12 (Bookworm), Debian 13 (Trixie), Ubuntu 22.04 (Jammy), Ubuntu 24.04 (Noble).
- **Target Architecture**: `amd64` (x86_64), `arm64` (aarch64).
- **Collections**:
  - `ansible.builtin`
  - `community.general` (for `community.general.htpasswd`)

---

## Role Variables

### Core & Distribution Settings

| Variable | Default | Description |
|---|---|---|
| `zot_version` | `"2.1.2"` | Upstream Zot release version |
| `zot_flavor` | `"full"` | Binary flavor: `"full"` (with UI, metrics, search, sync) or `"minimal"` |
| `zot_install_zli` | `true` | Install the official `zli` management CLI utility |
| `zot_user` | `"zot"` | System user for the Zot daemon |
| `zot_group` | `"zot"` | System group for the Zot daemon |
| `zot_bin_dir` | `"/usr/local/bin"` | Target directory for Zot executable binaries |
| `zot_config_dir` | `"/etc/zot"` | Directory for configuration and credentials |
| `zot_data_dir` | `"/var/lib/zot"` | Local metadata, BoltDB indexes, and upload staging cache |

### Network & Reverse Proxy

| Variable | Default | Description |
|---|---|---|
| `zot_listen_address` | `"0.0.0.0"` | Network interface address for HTTP binding |
| `zot_listen_port` | `8585` | HTTP listening port (ideal upstream target for Nginx Proxy Manager) |
| `zot_behind_reverse_proxy` | `true` | Prepares Zot for upstream proxy header handling |
| `zot_tls_enabled` | `false` | Enable native TLS inside Zot (keep `false` when using NPM) |

### IDrive e2 (S3) Storage Backend

| Variable | Default | Description |
|---|---|---|
| `zot_s3_enabled` | `true` | Enable S3 object storage driver |
| `zot_s3_endpoint` | `""` | IDrive e2 endpoint URL or hostname (e.g., `s3.us-east-1.idrivee2-20.com`) |
| `zot_s3_bucket` | `""` | Target IDrive e2 bucket name |
| `zot_s3_region` | `"us-east-1"` | S3 region |
| `zot_s3_access_key` | `""` | IDrive e2 Access Key ID |
| `zot_s3_secret_key` | `""` | IDrive e2 Secret Access Key |
| `zot_s3_secure` | `true` | Enforce HTTPS for S3 API communication |
| `zot_s3_skip_verify` | `false` | Skip SSL certificate verification (keep `false` in production) |
| `zot_s3_force_path_style` | `true` | Path-style addressing (required for IDrive e2) |
| `zot_storage_dedupe` | `true` | Enable content-addressable layer blob deduplication |
| `zot_storage_gc` | `true` | Enable automated garbage collection |
| `zot_storage_gc_interval` | `"24h"` | Garbage collection frequency |

### Retention & Policies

| Variable | Default | Description |
|---|---|---|
| `zot_retention_enabled` | `true` | Enable tag retention policies |
| `zot_retention_dry_run` | `false` | Log retention purges without deleting blobs |
| `zot_retention_keep_tags_count` | `5` | Number of most recent tags to retain per repository |
| `zot_retention_delete_untagged` | `true` | Automatically delete untagged image revisions |

### Authentication & Access Control

| Variable | Default | Description |
|---|---|---|
| `zot_auth_mode` | `"htpasswd"` | Authentication mechanism: `"htpasswd"` or `"none"` |
| `zot_allow_anonymous_read` | `true` | Permit anonymous pull access while requiring auth to push |
| `zot_users` | `[]` | List of user dictionaries: `[{ username: "admin", password: "...", state: "present" }]` |

### Extensions & Observability

| Variable | Default | Description |
|---|---|---|
| `zot_enable_ui` | `true` | Enable embedded web management UI |
| `zot_enable_metrics` | `true` | Enable Prometheus metrics endpoint at `/metrics` |
| `zot_enable_search` | `true` | Enable GraphQL metadata and image search engine |
| `zot_enable_sync` | `false` | Enable pull-through caching from upstream registries |
| `zot_sync_registries` | `[]` | List of upstream registries to proxy and mirror |

---

## Example Playbook

```yaml
---
- name: Deploy Zot OCI Registry with IDrive e2 Storage
  hosts: registry_nodes
  become: true
  roles:
    - role: clawduino.zot_registry
      vars:
        zot_listen_port: 8585
        zot_s3_enabled: true
        zot_s3_endpoint: "s3.us-east-1.idrivee2-20.com"
        zot_s3_bucket: "homelab-zot-registry"
        zot_s3_region: "us-east-1"
        zot_s3_access_key: "{{ vault_idrive_e2_access_key }}"
        zot_s3_secret_key: "{{ vault_idrive_e2_secret_key }}"
        zot_users:
          - username: "ci-builder"
            password: "{{ vault_ci_builder_password }}"
          - username: "admin"
            password: "{{ vault_admin_password }}"
        zot_enable_sync: true
        zot_sync_registries:
          - url: "https://registry-1.docker.io"
            prefix: "dockerhub"
            on_demand: true
            tls_verify: true
```

---

## Nginx Proxy Manager (NPM) Integration

1. In Nginx Proxy Manager, click **Add Proxy Host**.
2. **Details**:
   - **Domain Names**: `registry.example.com`
   - **Scheme**: `http`
   - **Forward Hostname / IP**: `<LXC_IP>`
   - **Forward Port**: `8585`
   - Enable: **Cache Assets**, **Block Common Exploits**, **Websockets Support**.
3. **Custom Locations** (Optional for large image pushes):
   - Under the **Advanced** tab, configure client body upload size:
     ```nginx
     client_max_body_size 0;
     proxy_request_buffering off;
     proxy_http_version 1.1;
     proxy_read_timeout 900s;
     ```
4. **SSL**:
   - Request or assign a Let's Encrypt certificate with **Force SSL** and **HTTP/2 Support** enabled.

---

## Pushing and Pulling Images

### 1. Authenticate with Registry
```bash
docker login registry.example.com -u ci-builder -p "<password>"
```

### 2. Tag and Push an Image
```bash
docker tag my-app:latest registry.example.com/production/my-app:1.0.0
docker push registry.example.com/production/my-app:1.0.0
```

### 3. Pulling Anonymous Images (Nodes without Credentials)
```bash
docker pull registry.example.com/production/my-app:1.0.0
```

### 4. Pull-Through Docker Hub Caching (if `zot_enable_sync` is active)
```bash
docker pull registry.example.com/dockerhub/library/alpine:latest
```

---

## License

MIT

## Author Information

Created by **Jason Parks** (clawduino).
