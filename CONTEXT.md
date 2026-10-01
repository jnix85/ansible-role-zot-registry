# Zot Registry Management

Manages the deployment, lifecycle, and configuration of the Zot OCI image registry running inside a Linux Container (LXC) with cloud-backed object storage on IDrive e2.

## Language

**Registry Host**:
The target Linux Container (Debian or Ubuntu LXC guest) in inventory executing the native Zot daemon.
_Avoid_: Registry VM, Docker host, Container server

**Object Storage**:
IDrive e2 S3-compatible cloud storage destination storing all container image layers and manifest blobs.
_Avoid_: Local disk, NAS share, S3 bucket (generic without IDrive e2 context)

**Root Directory**:
The local filesystem hierarchy on the Registry Host (`/var/lib/zot`) housing internal BoltDB/BadgerDB indexes, search caches, and transient chunk upload staging.
_Avoid_: Image store, data volume, blob storage

**Storage Driver**:
The Zot storage driver mechanism configured with S3-compatible parameters (`regionendpoint`, `bucket`, `forcepathstyle`, `secure`) targeting IDrive e2.
_Avoid_: Filesystem driver, storage backend

**Pull-Through Mirror**:
An on-demand upstream proxy caching feature that transparently pulls, caches, and serves images from external registries like Docker Hub or GitHub Container Registry.
_Avoid_: Proxy registry, remote cache, registry mirror

**Access Control Matrix**:
The security policy enforcing unauthenticated public read (anonymous pull) paired with bcrypt-hashed htpasswd credentials for write operations (push and delete).
_Avoid_: Basic auth, login configuration

**Ingress Edge**:
The external reverse proxy (Nginx Proxy Manager) terminating TLS certificates and forwarding plain HTTP traffic to port 8585 on the Registry Host.
_Avoid_: Ingress gateway, web server, internal TLS
