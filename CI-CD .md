# Multi-Client Docker Deployment using Azure DevOps, Ansible and Docker Compose

## Overview

This deployment architecture is designed to support multiple clients/tenants using a shared application codebase with client-specific configurations.

The solution uses:

* Microsoft Azure Azure DevOps for CI/CD
* Docker for containerization
* Azure Container Registry (ACR) for image storage
* Ansible for deployment automation
* Docker Compose for container orchestration

The deployment process is divided into:

1. CI Pipeline (Build & Push)
2. CD Stage-1 (Artifact Transfer)
3. CD Stage-2 (Remote Deployment using Ansible)

---

# Architecture Flow

```text
Developer Push
      ↓
Azure DevOps CI Pipeline
      ↓
Build Docker Images
      ↓
Push Images to ACR
      ↓
Publish Ansible + Docker Compose Artifacts
      ↓
CD Stage-1
Copy Artifacts to Target Server/Central server as temp path to execute ansible from there 
      ↓
CD Stage-2
Execute deploy.sh via SSH
      ↓
Ansible Playbook Execution
      ↓
Docker Compose Deployment
      ↓
Container Configuration & Restart
```

---

# Repository Structure

```text
ansible/
├── group_vars/
├── inventories/
│   ├── Romania/
│   ├── usa/
│   ├── kseb/
│   └── test/
├── playbooks/
│   ├── listservices.yml
│   └── mdmapplication.yml
├── roles/
│   ├── mdmapplication/
│   └── prerequisites/
└── shell/
    └── deploy.sh
```

---

# Multi-Client Inventory Structure

Each client has its own inventory folder.

Example:

```text
inventories/
 ├── Romania/
 │    ├── dc.yml
 │    ├── appsettingsdcapi.json
 │    ├── appsettingsdcui.json
 │    └── appsettingsaidc.json
 │
 ├── usa/
 └── kseb/
```

These folders contain:

* Environment inventory YAML
* API configuration
* UI configuration
* Authentication/SSO configuration

This enables tenant-specific deployments while using common Docker images.

---

# Ansible Playbook Structure

## Main Playbook

File:

```text
playbooks/mdmapplication.yml
```

Purpose:

* Executes deployment roles
* Runs deployment tasks on target servers

Example:

```yaml
- name: Deploy hesapplication
  hosts: mdmapplication_servers
  become: yes

  roles:
    - prerequisites
    - mdmapplication
```

---

# Deployment Role

## File

```text
roles/mdmapplication/tasks/main.yml
```

## Responsibilities

The role performs:

### 1. Create Deployment Directory

Creates deployment folder on target server.

```yaml
file:
  path: "{{ deploy_path }}"
```

---

### 2. Copy Deployment Files

Copies:

* docker-compose.yml
* UI appsettings
* API appsettings
* SSO appsettings

into deployment directory.

---

### 3. Docker Compose Operations

Deployment sequence:

```text
docker-compose down
docker-compose pull
docker-compose up -d
```

Purpose:

* Stop existing containers
* Pull latest images from ACR
* Start updated containers

---

### 4. Azure Container Registry Login

Logs into ACR before image pull.

```bash
docker login esyasoftacr.azurecr.io
```

---

### 5. Container Validation

Checks whether UI container is running successfully.

---

### 6. Runtime Configuration Injection

Copies configuration files into running containers using:

```bash
docker cp
```

This updates:

* API configuration
* UI configuration
* SSO configuration

---

### 7. Asset Deployment

If PNG assets exist:

* existing asset removed
* new asset copied into UI container

---

### 8. Container Restart

Restarts containers after configuration update.

---

# Shell Deployment Script

## File

```text
ansible/shell/deploy.sh
```

## Purpose

Acts as the deployment entry point.

Responsible for:

* validating arguments
* ACR authentication
* selecting inventory
* selecting playbook
* executing Ansible playbook

---

# Deployment Command

Example:

```bash
./deploy.sh Romania dc
```

Parameters:

| Parameter | Description             |
| --------- | ----------------------- |
| Romania   | Customer Name           |
| dc        | Environment/Tenant Code |

---

# Inventory Resolution

The script dynamically selects:

```bash
inventories/Romania/dc.yml
```

This enables customer-specific deployment.

---

# Dynamic Variables Passed to Ansible

The script passes:

```bash
tenant_code
Customer
ansible_owner
ansible_group
```

These variables are used inside deployment tasks.

---

# CI Pipeline Overview

The CI pipeline performs:

1. Source checkout
2. Docker image build
3. Docker image push to ACR
4. Packaging deployment artifacts
5. Publishing artifacts

---

# CI Pipeline Workflow

## Trigger

Pipeline triggers from:

```yaml
branches:
  include:
    - docker
```

---

# Docker Images Built

The pipeline builds:

| Service        | Repository              |
| -------------- | ----------------------- |
| MDM API        | esyasoft-mdm-api        |
| Authentication | esyasoft-authentication |
| MDM UI         | esyasoft-mdm-ui         |

---

# Docker Image Tags

Images are tagged with:

```text
latest
Build.BuildId
```

---

# Artifact Packaging

The pipeline packages:

```text
ansible/
docker-compose.yml
```

into build artifacts.

---

# Published Artifact Structure

```text
ansible/
 ├── docker/
 │    └── docker-compose.yml
 ├── inventories/
 ├── playbooks/
 ├── roles/
 └── shell/
```

---

# CD Stage-1

## Purpose

Copies deployment artifacts from Azure DevOps agent to deployment server.

## Target Path

```text
/home/mdmapplication/ansible
```

---

# CD Stage-2

## Purpose

Executes remote deployment through SSH.

## Steps Performed

1. Connect to remote server
2. Navigate to deployment script folder
3. Convert shell file to Unix format
4. Add execute permission
5. Execute deployment script

---

# Deployment Execution Example

```bash
./deploy.sh Romania dc
```

This triggers:

* Ansible playbook execution
* Docker deployment
* Container configuration update
* Service restart

---

# Multi-Client Deployment Design

The deployment supports multiple customers by:

* maintaining separate inventory folders
* maintaining separate appsettings files
* dynamically selecting deployment variables
* using shared Docker images
* using centralized deployment automation

---

# Technologies Used

| Component                | Technology                               |
| ------------------------ | ---------------------------------------- |
| CI/CD                    | Microsoft Azure Azure DevOps             |
| Containerization         | Docker Docker                            |
| Registry                 | Microsoft Azure Azure Container Registry |
| Configuration Management | Ansible                                  |
| Remote Execution         | SSH                                      |
| Orchestration            | Docker Compose                           |

---

# Current Deployment Characteristics

## Advantages

* Centralized deployment process
* Reusable deployment automation
* Tenant-specific configurations
* Shared Docker image architecture
* Automated CI/CD pipeline
* Environment-based deployment

---

# Current Deployment Limitations

* Hardcoded ACR credentials
* Runtime config injection using docker cp
* Full container downtime during deployment
* Shared container names
* Manual restart dependency
* Limited rollback capability

---

# Conclusion

This deployment architecture provides a centralized multi-client deployment solution using Azure DevOps, Docker, and Ansible.

The design enables:

* reusable deployment automation
* client-specific configurations
* automated container deployment
* centralized infrastructure management

The current implementation is suitable for small-to-medium scale multi-tenant deployments and provides a solid foundation for future DevOps and DevSecOps improvements.
