# Docker TAR Deployment — Short Notes

## What is TAR in Docker?

TAR is an archive file used to package Docker images for transfer between servers.

---

# Why TAR is used?

Used when:

* Production server has no internet
* Registry access is blocked
* Secure/offline deployment is required
* Company uses internal deployment process

---

# Main Flow

```text
Build Image
   ↓
docker save
   ↓
image.tar
   ↓
Copy to Server
   ↓
docker load
   ↓
Run Containers
```

---

# Important Commands

## Save Docker Image

```bash
docker save -o app.tar app:latest
```

Creates TAR file from Docker image.

---

## Load Docker Image

```bash
docker load -i app.tar
```

Restores image into Docker engine.

---

# What TAR Contains

* Image layers
* Metadata
* Tags
* Docker manifests

---

# Advantages

✅ Works without internet
✅ Secure deployment
✅ Easy artifact transfer
✅ Useful for air-gapped servers
✅ Good for enterprise environments

---

# docker save vs docker export

## docker save

* Saves complete image
* Keeps layers and tags
* Used in CI/CD

## docker export

* Saves only container filesystem
* Loses image metadata
* Rarely used for deployments

---

# Real-World Usage

Common in:

* Banking
* Government
* Healthcare
* Internal enterprise deployments

---

# Your Project Flow

```text
Azure DevOps
   ↓
Build Docker Image
   ↓
Save as TAR
   ↓
Ansible copies TAR
   ↓
Server loads image
   ↓
Docker Compose starts containers
```
