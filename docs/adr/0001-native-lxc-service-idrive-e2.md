# 1. Native Binary Process in LXC with IDrive e2 S3 Storage

## Context
Deploying an enterprise OCI container registry in a homelab Proxmox environment typically encounters overhead and storage friction when nesting Docker or Podman engines inside unprivileged LXC containers (specifically overlayfs on ZFS/btrfs). Furthermore, hosting multi-gigabyte container image blobs directly on local node disks consumes high-cost hypervisor storage and risks quota exhaustion during large builds.

## Decision
We deploy the upstream Go binary `/usr/local/bin/zot` as a native systemd service directly inside a Debian/Ubuntu LXC container, backed by IDrive e2 S3-compatible cloud object storage. Sensitive S3 credentials are decoupled from `/etc/zot/config.json` into `/etc/zot/zot.env` (`0600`), ingress TLS is terminated at Nginx Proxy Manager forwarding to HTTP port `8585`, and the registry enforces anonymous pulls alongside htpasswd-authenticated pushes.

## Considered Options
- **Docker-in-LXC / Podman Quadlet**: Rejected due to nested container virtualization overhead, storage driver instability on ZFS root filesystems, and complex nested cgroup delegation.
- **Local ZFS / NFS Blob Storage**: Rejected to prevent local hypervisor storage saturation and preserve local I/O for compute workloads.
- **In-Config Plaintext Credentials**: Rejected in favor of systemd `EnvironmentFile` to enforce strict POSIX permissions and eliminate credential disclosure risks.

## Consequences
- The role requires curl/tar on the target host for direct binary release installation from GitHub.
- Systemd sandboxing inside the LXC container must omit kernel-level restrictions (such as `ProtectKernelTunables` or `ProtectClock`) that fail in unprivileged user namespaces.
- S3 latency depends on external WAN connectivity to IDrive e2; local metadata and upload staging remain fast on `/var/lib/zot`.
