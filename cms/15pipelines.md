Yes. Below is the consolidated information I remember from our work. I’ve separated confirmed details from items still needed.

## 1. Azure DevOps

| Item              | Value                                       |
| ----------------- | ------------------------------------------- |
| Organization      | `esyasoft-mobility`                         |
| Organization URL  | `https://dev.azure.com/esyasoft-mobility`   |
| Project           | `Esyasoft.CHRGUP.CMS`                       |
| Project ID        | `a5bf1d55-203a-4fe1-99e5-a4d7e068eca9`      |
| Build agent image | `ubuntu-22.04`                              |
| Deployment method | Classic Release pipelines                   |
| Deployment type   | Offline Docker TAR deployment using Ansible |

## 2. Repositories and branches

| Repository             | Demo branch | AEML branch | BLive branch |
| ---------------------- | ----------- | ----------- | ------------ |
| `BACKEND`              | `main`      | `AEML-main` | `BLive-main` |
| `FRONTEND`             | `main`      | `AEML-main` | `blive-main` |
| `Esyasoft.AzureDevOps` | `main-prod` | `main-prod` | `main-prod`  |

Branch casing is important:

* Backend BLive: `BLive-main`
* Frontend BLive: `blive-main`

## 3. Projects and services

There are three projects:

* `demo`
* `aeml`
* `blive`

Each project has five independently deployable services:

| Service      | Internal service argument                       |
| ------------ | ----------------------------------------------- |
| Frontend/UI  | `frontend`                                      |
| API          | `api`                                           |
| RMQ Consumer | `rmqconsumer` or possibly the shell alias `rmq` |
| OCPI         | `ocpi`                                          |
| OCPP         | `ocpp`                                          |

The RMQ argument needs one final verification: some release commands used `rmq`, while the Ansible service map uses `rmqconsumer`.

## 4. Server distribution

| Server       | Services                                              |
| ------------ | ----------------------------------------------------- |
| `10.149.1.4` | Frontend, API, RMQ Consumer, OCPI, Ansible controller |
| `10.149.2.4` | OCPP                                                  |
| `10.149.2.5` | OCPP                                                  |

The Classic Release pipeline connects to `10.149.1.4`, including for OCPP deployment. That is intentional: Ansible starts on the controller and then targets the OCPP inventory group.

## 5. CI/build pipeline names

### Demo

* `cms-demo-api`
* `cms-demo-ocpi`
* `cms-demo-ocpp`
* `cms-demo-rmq`
* `cms-demo-ui`

### AEML

* `cms-aeml-api`
* `cms-aeml-ocpi`
* `cms-aeml-ocpp`
* `cms-aeml-rmq`
* `cms-aeml-ui`

### BLive

* `cms-blive-api`
* `cms-blive-ocpi`
* `cms-blive-ocpp`
* `cms-blive-rmq`
* `cms-blive-ui`

The frontend build pipelines use `-ui`, while the release definitions currently use `-frontend`.

## 6. Backend pipeline YAML files

Inside the `BACKEND` repository:

```text
pipelines/
├── api-pipeline.yml
├── ocpi-pipeline.yml
├── ocpp-pipeline.yml
└── rmqconsumer-pipeline.yml
```

Frontend uses:

```text
docker-pipeline.yml
```

## 7. Classic Release pipelines

| ID | Release definition   | Folder       |
| -: | -------------------- | ------------ |
| 24 | `cms-aeml-api`       | `\cms-aeml`  |
| 14 | `cms-aeml-frontend`  | `\cms-aeml`  |
| 16 | `cms-aeml-ocpi`      | `\cms-aeml`  |
| 17 | `cms-aeml-ocpp`      | `\cms-aeml`  |
| 18 | `cms-aeml-rmq`       | `\cms-aeml`  |
| 15 | `cms-blive-api`      | `\cms-blive` |
| 19 | `cms-blive-frontend` | `\cms-blive` |
| 21 | `cms-blive-ocpi`     | `\cms-blive` |
| 22 | `cms-blive-ocpp`     | `\cms-blive` |
| 23 | `cms-blive-rmq`      | `\cms-blive` |
|  8 | `cms-demo-api`       | `\cms-demo`  |
| 13 | `cms-demo-frontend`  | `\cms-demo`  |
| 11 | `cms-demo-ocpi`      | `\cms-demo`  |
| 10 | `cms-demo-ocpp`      | `\cms-demo`  |
| 12 | `cms-demo-rmq`       | `\cms-demo`  |

All 15 release definitions were intended to have:

* Continuous deployment artifact trigger enabled
* Pull-request trigger disabled
* Stage 1 overwrite enabled
* Stage 1 clean target disabled
* Independent service deployment

## 8. Release stages

### Stage 1 — Copy artifact

Destination:

```text
/mnt/data/chrgup/ansible/
```

Expected settings:

```text
Contents: **
OverWrite: true
CleanTargetFolder: false
```

Typical source format:

```text
$(System.DefaultWorkingDirectory)/_<build-pipeline>/<artifact-name>
```

### Stage 2 — Execute deployment

```bash
cd /mnt/data/chrgup/ansible/shell
sed -i 's/\r$//' deploy.sh
chmod +x deploy.sh
./deploy.sh <project> dc <service>
```

Examples:

```bash
./deploy.sh demo dc api
./deploy.sh aeml dc ocpi
./deploy.sh blive dc ocpp
```

## 9. Image, container and TAR naming

Standard naming without ACR:

```text
Image:     chrgup-cms-<project>-<service>:v1.1
Container: chrgup-<project>-<service>
TAR:       chrgup-cms-<project>-<service>.tar
```

Examples:

```text
chrgup-cms-demo-api:v1.1
chrgup-demo-api
chrgup-cms-demo-api.tar
```

```text
chrgup-cms-aeml-ocpp:v1.1
chrgup-aeml-ocpp
chrgup-cms-aeml-ocpp.tar
```

```text
chrgup-cms-blive-rmqconsumer:v1.1
chrgup-blive-rmqconsumer
chrgup-cms-blive-rmqconsumer.tar
```

Old `esyasoftacr.azurecr.io/...` images are legacy and are not part of the current offline deployment.

## 10. Artifact naming

The intended project-specific pattern appears to be:

```text
ansible-<project>-api
ansible-<project>-frontend
ansible-<project>-ocpi
ansible-<project>-ocpp
ansible-<project>-rmqconsumer
```

Confirmed example:

```text
ansible-aeml-rmqconsumer
```

However, some earlier pipelines used generic names such as:

```text
ansible-api
ansible-frontend
ansible-ocpi
ansible-ocpp
ansible-rmqconsumer
```

The exact artifact names currently produced by all 15 CI pipelines still need to be collected.

## 11. Ansible repository structure

Repository: `Esyasoft.AzureDevOps`

Branch: `main-prod`

```text
ansible/
├── docker/
│   ├── demo/
│   │   ├── docker-compose-main.yml
│   │   └── docker-compose-ocpp.yml
│   ├── aeml/
│   │   ├── docker-compose-main.yml
│   │   └── docker-compose-ocpp.yml
│   └── blive/
│       ├── docker-compose-main.yml
│       └── docker-compose-ocpp.yml
├── group_vars/
│   ├── all.yml
│   ├── demo.yml
│   ├── aeml.yml
│   └── blive.yml
├── inventories/
│   ├── demo/
│   ├── aeml/
│   └── blive/
├── playbooks/
│   └── chrgupcms.yml
├── roles/
│   ├── chrgupcms/
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── chrgupcms_main/
│   ├── chrgupcms_ocpp/
│   └── prerequisites/
└── shell/
    └── deploy.sh
```

Each project inventory contains:

```text
appsettingsdcapi.json
appsettingsdcocpi.json
appsettingsdcocpp.json
appsettingsdcrmq.json
dc.yml
```

The active playbook uses the `chrgupcms` role. The `chrgupcms_main` and `chrgupcms_ocpp` roles appear to be legacy unless referenced elsewhere.

## 12. Ansible service mapping

```yaml
chrgup_services:
  frontend:
    tar_name: "chrgup-cms-{{ project_name }}-frontend.tar"
    compose_file: docker-compose-main.yml
    compose_service: frontend
    container_name: "chrgup-{{ project_name }}-frontend"
    appsettings_suffix: ""

  api:
    tar_name: "chrgup-cms-{{ project_name }}-api.tar"
    compose_file: docker-compose-main.yml
    compose_service: api
    container_name: "chrgup-{{ project_name }}-api"
    appsettings_suffix: api

  rmqconsumer:
    tar_name: "chrgup-cms-{{ project_name }}-rmqconsumer.tar"
    compose_file: docker-compose-main.yml
    compose_service: rmqconsumer
    container_name: "chrgup-{{ project_name }}-rmqconsumer"
    appsettings_suffix: rmq

  ocpi:
    tar_name: "chrgup-cms-{{ project_name }}-ocpi.tar"
    compose_file: docker-compose-main.yml
    compose_service: ocpi
    container_name: "chrgup-{{ project_name }}-ocpi"
    appsettings_suffix: ocpi

  ocpp:
    tar_name: "chrgup-cms-{{ project_name }}-ocpp.tar"
    compose_file: docker-compose-ocpp.yml
    compose_service: ocpp
    container_name: "chrgup-{{ project_name }}-ocpp"
    appsettings_suffix: ocpp
```

## 13. Playbook target selection

Expected inventory groups:

```yaml
chrgup_main_servers
chrgup_ocpp_servers
```

Selection logic:

* `frontend`, `api`, `rmqconsumer`, `ocpi` → `chrgup_main_servers`
* `ocpp` → `chrgup_ocpp_servers`

Server selection is handled by the playbook/inventory, not by the shell script.

## 14. Deployment paths

Ansible controller:

```text
/mnt/data/chrgup/ansible
```

Application Compose paths:

```text
/mnt/data/docker/cms/demo
/mnt/data/docker/cms/aeml
/mnt/data/docker/cms/blive
```

Old paths no longer intended for active deployment:

```text
/home/chargup/ansible
/home/test/code/docker
/mnt/data/ansible
```

Important unresolved point: on the checked OCPP server, `/mnt/data` was not mounted. It had:

```text
/mnt     -> /dev/sdb1
/dev/sdc -> unmounted 256 GB disk
/dev/sdd -> unmounted 128 GB disk
```

Therefore, `/mnt/data/docker/cms/...` must not be assumed valid on both OCPP servers until their disk mount configuration is finalized.

## 15. Docker Compose files

For every project:

```text
docker/<project>/docker-compose-main.yml
docker/<project>/docker-compose-ocpp.yml
```

Main Compose service keys:

```text
frontend
api
rmqconsumer
ocpi
```

OCPP Compose service key:

```text
ocpp
```

Compose project names:

```text
chrgup-demo
chrgup-aeml
chrgup-blive
```

Independent deployment command:

```bash
docker compose \
  -p "chrgup-<project>" \
  -f docker-compose-main.yml \
  up -d --no-deps --force-recreate <service>
```

On older servers without Compose v2, use:

```bash
docker-compose \
  -p "chrgup-<project>" \
  -f docker-compose-main.yml \
  up -d --no-deps --force-recreate <service>
```

The earlier error `unknown shorthand flag: 'p'` showed that `docker compose` was unavailable on at least one server.

## 16. Known ports

Confirmed from earlier Compose examples:

| Project |           API |          OCPI |          OCPP |
| ------- | ------------: | ------------: | ------------: |
| Demo    |        `5004` |        `5005` |        `5193` |
| AEML    |        `7004` |        `7005` |        `7193` |
| BLive   | Not confirmed | Not confirmed | Not confirmed |

All services currently use host networking:

```yaml
network_mode: host
```

## 17. Current server/storage observations

On the checked OCPP server:

* Docker reported root: `/var/lib/docker`
* Most image storage was actually under `/var/lib/containerd`
* Three current OCPP images were in use:

  * `chrgup-cms-demo-ocpp:v1.1`
  * `chrgup-cms-aeml-ocpp:v1.1`
  * `chrgup-cms-blive-ocpp:v1.1`
* All three OCPP containers were restarting with exit code `139`
* Exit `139` indicates an application/native crash and remains unresolved
* Old ACR images were still present but unused

## 18. Information still needed

Please provide these details for complete documentation:

1. Exact Azure DevOps build definition IDs for all 15 CI pipelines.
2. Exact artifact name produced by each CI pipeline.
3. Repository IDs for:

   * `BACKEND`
   * `FRONTEND`
   * `Esyasoft.AzureDevOps`
4. Final inventory contents for:

   * `inventories/demo/dc.yml`
   * `inventories/aeml/dc.yml`
   * `inventories/blive/dc.yml`
5. SSH usernames and ports for all three servers.
6. Azure DevOps deployment-group names and agent names.
7. BLive API, OCPI and OCPP port numbers.
8. Frontend ports or reverse-proxy URLs for each project.
9. Whether RMQ Stage 2 must use `rmq` or `rmqconsumer`.
10. Final secure-file mapping:

    * Demo: `.env.prod`
    * AEML: likely `.env.prodAEML`
    * BLive: likely `.env.prodblive`
11. Whether `/mnt/data` will be mounted on both OCPP servers.
12. Final Docker application path on OCPP servers.
13. Which disk should be used on each OCPP server.
14. Docker Compose version installed on each server.
15. Current health/log cause for the three OCPP exit-139 failures.
16. Whether `10.149.2.4` and `10.149.2.5` both run all three project OCPP containers or are split between projects.
17. Database, RabbitMQ, Redis and external endpoint details—preferably names and ports only, with secrets excluded.
18. Backup, rollback and artifact-retention requirements.
19. Production health-check URLs for each service.
20. Final owner/contact and approval process for production releases.

This is the complete technical state I currently remember; the main uncertainty is exact CI artifact/build IDs, final OCPP storage mounts, BLive ports, and the RMQ command alias.
