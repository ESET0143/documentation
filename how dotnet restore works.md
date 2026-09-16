# Reflinks:
- 1.https://dev.azure.com/esyasoft/Esyasoft-Gas-Services/_build/results?buildId=21756&view=results
- 2.https://dev.azure.com/esyasoft/Esyasoft-Gas-Services/_git/Esyasoft.CRM.GAS.Services?version=GBtestpkgs
````markdown
# NuGet Restore, Feeds, Azure Artifacts – Notes

## 1. What does `dotnet restore` do?

`dotnet restore` downloads all NuGet packages required by the project before the build.

It reads:

- Solution files (`.sln`)
- Project files (`.csproj`)
- `PackageReference` entries

Example:

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Serilog" Version="4.0.0" />
</ItemGroup>
````

NuGet sees these package references and downloads the required packages.

---

## 2. What is a NuGet Feed?

A **NuGet feed** is a package repository where NuGet packages are stored.

### Public Feed

Example:

```text
https://api.nuget.org/v3/index.json
```

This contains public packages such as:

* `Newtonsoft.Json`
* `Serilog`
* `AutoMapper`
* `Entity Framework Core`

### Private Feed – Azure Artifacts

Example:

```text
https://pkgs.dev.azure.com/<org>/<project>/_packaging/<feed>/nuget/v3/index.json
```

This may contain private company packages such as:

* `Esyasoft.Common`
* `Esyasoft.Security`
* `Esyasoft.Logging`

---

## 3. What is the purpose of `nuget.config`?

`nuget.config` does **not** contain package names.

It contains package sources, also called feeds.

Example:

```xml
<packageSources>
  <add key="Esyasoftpackages"
       value="https://pkgs.dev.azure.com/esyasoft/.../index.json" />

  <add key="NuGetOrg"
       value="https://api.nuget.org/v3/index.json" />
</packageSources>
```

Meaning:

```text
Search for packages in these repositories.
```

---

## 4. Where are package names stored?

Package names are usually stored inside `.csproj` files.

Example:

```xml
<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
```

`dotnet restore` reads these package references and tries to find the requested package and version in the configured NuGet feeds.

---

## 5. How restore works

Assume the project needs:

```text
Newtonsoft.Json
Serilog
Esyasoft.Common
```

Configured feeds:

```text
Feed 1: Azure Artifacts
Feed 2: NuGet.org
```

Restore process:

```text
Need Newtonsoft.Json
    |
    +--> Check Azure Artifacts
    |       |
    |       +--> Not Found
    |
    +--> Check NuGet.org
            |
            +--> Found
            |
            +--> Download
```

For a private package:

```text
Need Esyasoft.Common
    |
    +--> Check Azure Artifacts
            |
            +--> Found
            |
            +--> Download
```

---

## 6. Why doesn't NuGet directly check only NuGet.org?

Because some packages may exist only in a private feed such as Azure Artifacts.

Example:

```xml
<PackageReference Include="Esyasoft.Common" Version="1.0.0" />
```

If `Esyasoft.Common` exists only in Azure Artifacts, NuGet.org will not contain it.

If only NuGet.org is configured:

```text
Package not found
        |
        v
Restore fails
```

Therefore, NuGet searches the configured feeds.

---

## 7. Azure DevOps Restore Using `nuget.config`

Example:

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: restore
    projects: 'MySolution.sln'
    feedsToUse: config
    nugetConfigPath: 'nuget.config'
```

Meaning:

```text
Read feeds from nuget.config
        |
        v
Use those feeds during restore
```

---

## 8. Azure DevOps Restore Without Explicit `nuget.config`

Example:

```bash
dotnet restore MyProject.csproj
```

NuGet automatically searches for configuration files in locations such as:

* Project directory
* Parent directories
* Repository root
* User-level NuGet configuration
* Machine-level NuGet configuration

Because of this, restore may succeed even without explicitly specifying:

```yaml
nugetConfigPath
```

---

## 9. Purpose of `NuGetAuthenticate@1`

Example:

```yaml
- task: NuGetAuthenticate@1
```

This task:

* Authenticates to Azure Artifacts
* Creates temporary credentials
* Allows access to private Azure Artifacts feeds

It does **not** tell `dotnet restore` which feed URLs to use.

It only provides authentication credentials.

The responsibilities are different:

```text
nuget.config
    |
    +--> Tells NuGet WHERE to search

NuGetAuthenticate@1
    |
    +--> Provides credentials to ACCESS private feeds
```

---

## 10. How to check configured feeds

Run:

```bash
dotnet nuget list source
```

Example output:

```text
Registered Sources:

1. nuget.org
   https://api.nuget.org/v3/index.json

2. Esyasoftpackages
   https://pkgs.dev.azure.com/esyasoft/.../index.json
```

To check sources from a specific config file:

```bash
dotnet nuget list source --configfile ./nuget.config
```

---

## 11. How to find all `nuget.config` files

### Linux

```bash
find . -iname "nuget.config"
```

### Windows

```cmd
dir /s nuget.config
```

This is useful when restore is using a configuration file that is different from the one expected.

---

## 12. How to see which feed is actually used

Use detailed logging:

```bash
dotnet restore --verbosity detailed
```

Or:

```bash
dotnet restore --verbosity diagnostic
```

The logs may show requests such as:

```text
GET https://api.nuget.org/v3/index.json
```

and:

```text
GET https://pkgs.dev.azure.com/esyasoft/.../index.json
```

This helps identify which feeds NuGet is contacting during restore.

---

## 13. Where restored packages are stored

NuGet normally stores restored packages in the global package cache.

### Windows

```text
C:\Users\<user>\.nuget\packages
```

### Linux

```text
~/.nuget/packages
```

Packages stored in this cache can be reused during later restores.

If the cache is cleared, NuGet must download the packages again.

Example:

```bash
dotnet nuget locals all --clear
```

---

## 14. Alternative ways to restore packages

### Option 1: Azure DevOps Feed Selection

Azure DevOps can select a feed directly.

Example:

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: restore
    projects: 'MySolution.sln'
    feedsToUse: select
    vstsFeed: MyFeed
```

---

### Option 2: Command-Line Sources

Feeds can also be supplied directly to `dotnet restore`.

Example:

```bash
dotnet restore \
  --source https://api.nuget.org/v3/index.json \
  --source https://pkgs.dev.azure.com/.../index.json
```

---

### Option 3: User-Level `NuGet.Config`

#### Windows

```text
%AppData%\NuGet\NuGet.Config
```

#### Linux

```text
~/.nuget/NuGet/NuGet.Config
```

This configuration can provide package sources without requiring a project-level `nuget.config`.

---

## 15. Azure Artifacts Upstream Sources

Azure Artifacts feeds can be configured with upstream sources.

For example:

```text
Azure Artifacts Feed
        |
        +--> NuGet.org
```

In this setup, the restore flow may look like:

```text
dotnet restore
      |
      v
Azure Artifacts
      |
      +--> Package exists in feed
      |
      +--> Or retrieve/cache package from NuGet.org upstream
```

The application may therefore use only the Azure Artifacts feed URL while Azure Artifacts itself retrieves public packages from NuGet.org.

This can be useful in environments where direct access from the build agent to NuGet.org is restricted.

---

## 16. Overall NuGet Restore Flow

```text
.csproj / .sln
      |
      v
Read PackageReference entries
      |
      v
Find NuGet configuration
      |
      v
Read configured feeds
      |
      +-----------------------------+
      |                             |
      v                             v
Azure Artifacts                   NuGet.org
      |                             |
      v                             v
Private packages                 Public packages
      |
      v
Authenticate if required
      |
      v
Download package
      |
      v
Store in NuGet cache
      |
      v
Generate restore files
      |
      v
Build can continue
```

---

## 17. Important Difference: Feed Configuration vs Authentication

These two concepts should not be confused.

### Feed Configuration

Defines:

```text
Where should NuGet search?
```

Configured using:

```text
nuget.config
```

Example:

```xml
<add key="Esyasoftpackages"
     value="https://pkgs.dev.azure.com/.../index.json" />
```

### Authentication

Defines:

```text
Am I allowed to access this private feed?
```

In Azure DevOps, authentication can be provided using:

```yaml
- task: NuGetAuthenticate@1
```

The complete flow is:

```text
PackageReference
      |
      v
nuget.config
      |
      v
Find Azure Artifacts feed
      |
      v
NuGetAuthenticate credentials
      |
      v
Access feed
      |
      v
Find package
      |
      v
Download package
```

---

## 18. Quick Troubleshooting Commands

Check sources:

```bash
dotnet nuget list source
```

Check sources using a specific config:

```bash
dotnet nuget list source --configfile ./nuget.config
```

Find all NuGet configuration files:

```bash
find . -iname "nuget.config"
```

Run detailed restore:

```bash
dotnet restore --verbosity detailed
```

Run maximum diagnostic logging:

```bash
dotnet restore --verbosity diagnostic
```

Clear NuGet caches:

```bash
dotnet nuget locals all --clear
```

Restore using a specific configuration:

```bash
dotnet restore \
  --configfile ./nuget.config \
  --no-cache \
  --force
```

---

## 19. Key Points to Remember

* `.csproj` tells NuGet **which packages are required**.
* `nuget.config` tells NuGet **where to search for packages**.
* `NuGetAuthenticate@1` provides **credentials for private Azure Artifacts feeds**.
* NuGet.org normally contains public packages.
* Azure Artifacts normally contains private/internal packages.
* Multiple feeds can participate in one restore.
* `dotnet nuget list source` shows configured package sources.
* `dotnet restore --verbosity diagnostic` is useful when troubleshooting feed selection.
* Azure Artifacts upstream sources can proxy/cache packages from NuGet.org.
* Package source configuration and package source authentication are two separate concerns.

```
```

Azure Artifacts downloads and caches packages automatically.


````markdown
# Azure Artifacts NuGet Restore Inside Docker in Azure DevOps

## Purpose

This document explains how to restore private NuGet packages from **Azure Artifacts** during a Docker build in an Azure DevOps pipeline.

It also explains:

- How NuGet searches for packages
- How `nuget.config` is used
- Why `NuGetAuthenticate@1` alone is not enough for Docker builds
- How authentication works inside Docker
- How to pass Azure DevOps credentials into Docker
- Common errors such as `NU1101` and `NU1301`
- Recommended troubleshooting steps

---

# 1. Architecture Overview

The application uses:

- .NET 8
- Azure DevOps Pipelines
- Azure Artifacts
- Docker
- Azure Container Registry (ACR)
- Private NuGet package:
  `Esyasoft.Identity.SSO.Middleware`

The private NuGet feed is:

```text
Esyasoftpackages
````

Feed URL:

```text
https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json
```

---

# 2. Package Restore Flow

When this command runs:

```bash
dotnet restore Esyasoft.CRM.GAS.Web.csproj
```

NuGet performs the following process:

```text
Esyasoft.CRM.GAS.Web.csproj
        |
        v
Read PackageReference entries
        |
        v
Example:
Esyasoft.Identity.SSO.Middleware
        |
        v
Read NuGet package sources
        |
        v
nuget.config
        |
        +-----------------------------+
        |                             |
        v                             v
Azure Artifacts                   nuget.org
Esyasoftpackages
        |
        v
Authenticate
        |
        v
Find requested package/version
        |
        v
Download package
        |
        v
Resolve dependencies
        |
        v
Generate project.assets.json
```

---

# 3. NuGet Configuration

Create a `nuget.config` file in the project directory.

Recommended structure:

```text
Repository
|
+-- Esyasoft.CRM.GAS.Web
    |
    +-- Esyasoft.CRM.GAS.Web.csproj
    +-- Dockerfile
    +-- nuget.config
    +-- Pages
    +-- wwwroot
```

Example `nuget.config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />

    <add key="Esyasoftpackages"
         value="https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json" />

    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

The `<clear />` statement removes package sources inherited from other NuGet configuration files.

After `<clear />`, only the explicitly configured sources are used.

---

# 4. Verify NuGet Sources

To verify what sources NuGet sees:

```bash
dotnet nuget list source --configfile ./nuget.config
```

Expected output:

```text
Registered Sources:

1. Esyasoftpackages [Enabled]
   https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json

2. nuget.org [Enabled]
   https://api.nuget.org/v3/index.json
```

If only this appears:

```text
nuget.org
```

then Azure Artifacts is not being loaded from the expected `nuget.config`.

---

# 5. Azure DevOps Authentication

The pipeline uses:

```yaml
- task: NuGetAuthenticate@1
  displayName: Authenticate Azure Artifacts
```

This task authenticates the Azure DevOps build agent.

Example pipeline log:

```text
Setting up the credential provider to use the identity:

Esyasoft-Gas-Services Build Service (esyasoft)
```

The pipeline agent receives an access token through:

```text
VSS_NUGET_ACCESSTOKEN
```

Conceptually:

```text
NuGetAuthenticate@1
        |
        v
Azure DevOps Build Service identity
        |
        v
VSS_NUGET_ACCESSTOKEN
```

---

# 6. Important Docker Authentication Boundary

A Docker build runs inside a separate environment.

This means the following does **not** automatically work:

```text
Azure DevOps Agent
|
+-- NuGetAuthenticate@1
|      |
|      +-- credentials available here
|
+-- Docker build
       |
       +-- New container
              |
              +-- dotnet restore
```

The Docker container does not automatically inherit:

```text
/home/vsts/.nuget/plugins
```

or the Azure DevOps credential configuration from the host agent.

Therefore, authentication must also be configured inside the Docker build environment.

---

# 7. Azure Artifacts Credential Provider

Inside Docker, install the Azure Artifacts Credential Provider.

Example:

```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    curl -sL https://raw.githubusercontent.com/microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | bash && \
    rm -rf /var/lib/apt/lists/*
```

The credential provider allows `dotnet restore` to authenticate against Azure Artifacts.

---

# 8. Passing the Azure DevOps Token Into Docker

The token created by:

```yaml
NuGetAuthenticate@1
```

is available as:

```text
$(VSS_NUGET_ACCESSTOKEN)
```

Pass it to the Docker build:

```yaml
arguments: >
  --build-arg AZURE_ARTIFACTS_TOKEN=$(VSS_NUGET_ACCESSTOKEN)
```

The Dockerfile receives it using:

```dockerfile
ARG AZURE_ARTIFACTS_TOKEN
```

Flow:

```text
NuGetAuthenticate@1
        |
        v
VSS_NUGET_ACCESSTOKEN
        |
        v
docker build --build-arg
        |
        v
AZURE_ARTIFACTS_TOKEN
        |
        v
Docker build environment
```

---

# 9. Configure Azure Artifacts Authentication Inside Docker

Use:

```dockerfile
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="{\"endpointCredentials\":[{\"endpoint\":\"https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json\",\"username\":\"azuredevops\",\"password\":\"${AZURE_ARTIFACTS_TOKEN}\"}]}"
```

Important values:

```text
username = azuredevops
password = ${AZURE_ARTIFACTS_TOKEN}
```

The username is effectively a placeholder when token-based authentication is used.

The important credential is:

```text
AZURE_ARTIFACTS_TOKEN
```

Do not put a personal Azure DevOps password in the Dockerfile.

Do not hard-code a PAT directly into the Dockerfile.

---

# 10. Dockerfile Example

```dockerfile
# =========================
# Build Stage
# =========================

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

ARG AZURE_ARTIFACTS_TOKEN

WORKDIR /app

# Install Azure Artifacts Credential Provider
RUN apt-get update && \
    apt-get install -y curl && \
    curl -sL https://raw.githubusercontent.com/microsoft/artifacts-credprovider/master/helpers/installcredprovider.sh | bash && \
    rm -rf /var/lib/apt/lists/*

# Copy application source
COPY Esyasoft.CRM.GAS.Web/ /app/Esyasoft.CRM.GAS.Web/

WORKDIR /app/Esyasoft.CRM.GAS.Web

# Configure Azure Artifacts authentication
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="{\"endpointCredentials\":[{\"endpoint\":\"https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json\",\"username\":\"azuredevops\",\"password\":\"${AZURE_ARTIFACTS_TOKEN}\"}]}"

# Debug project files
RUN pwd
RUN ls -la

# Verify NuGet sources
RUN dotnet nuget list source --configfile ./nuget.config

# Clear NuGet cache
RUN dotnet nuget locals all --clear

# Restore packages
RUN dotnet restore "Esyasoft.CRM.GAS.Web.csproj" \
    --configfile ./nuget.config \
    --no-cache \
    --force

# Publish application
RUN dotnet publish "Esyasoft.CRM.GAS.Web.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore


# =========================
# Runtime Stage
# =========================

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

RUN apt-get update && \
    apt-get install -y tzdata && \
    ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime && \
    echo "Asia/Kolkata" > /etc/timezone && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=build /app/publish .

RUN mkdir -p /app/wwwroot/download

EXPOSE 7001

ENTRYPOINT ["dotnet", "Esyasoft.CRM.GAS.Web.dll"]
```

---

# 11. Azure DevOps Pipeline Example

Use separate Docker `build` and `push` tasks.

```yaml
trigger:
  branches:
    include:
      - docker

variables:
  acrLoginServer: 'esyasoftacr.azurecr.io'
  acrServiceConnection: 'EsyasoftACR'

stages:

  - stage: Build_and_Push_to_ACR
    displayName: Build and Push Docker Images to ACR

    jobs:

      - job: BuildAndPushToACR
        displayName: Build and Push Images to ACR

        pool:
          vmImage: 'ubuntu-22.04'

        steps:

          # =========================
          # Checkout Repository
          # =========================

          - checkout: self

          # =========================
          # Verify Repository Files
          # =========================

          - script: |
              echo "Listing repository files"
              ls -R $(Build.SourcesDirectory)
            displayName: List Repository Files

          # =========================
          # Authenticate Azure Artifacts
          # =========================

          - task: NuGetAuthenticate@1
            displayName: Authenticate Azure Artifacts

          # =========================
          # Login to ACR
          # =========================

          - task: Docker@2
            displayName: Login to ACR
            inputs:
              command: login
              containerRegistry: $(acrServiceConnection)

          # =========================
          # Build Docker Image
          # =========================

          - task: Docker@2
            displayName: Build CRM GAS Web Image
            inputs:
              command: build
              containerRegistry: $(acrServiceConnection)
              repository: esyasoft-crm-gas-web
              dockerfile: '$(Build.SourcesDirectory)/Esyasoft.CRM.GAS.Web/Dockerfile'
              buildContext: '$(Build.SourcesDirectory)'
              arguments: >
                --build-arg AZURE_ARTIFACTS_TOKEN=$(VSS_NUGET_ACCESSTOKEN)
              tags: |
                latest
                $(Build.BuildId)

          # =========================
          # Push Docker Image
          # =========================

          - task: Docker@2
            displayName: Push CRM GAS Web Image
            inputs:
              command: push
              containerRegistry: $(acrServiceConnection)
              repository: esyasoft-crm-gas-web
              tags: |
                latest
                $(Build.BuildId)
```

---

# 12. Why `buildAndPush` Was Split

Previously the pipeline used:

```yaml
command: buildAndPush
```

But we need to provide:

```yaml
arguments:
  --build-arg ...
```

Therefore it is better to use separate tasks:

```text
Docker build
     |
     v
Docker push
```

This also makes troubleshooting easier.

---

# 13. Complete Authentication Flow

```text
Azure DevOps Pipeline
        |
        v
NuGetAuthenticate@1
        |
        v
Build Service authenticated
        |
        v
VSS_NUGET_ACCESSTOKEN
        |
        v
Docker build argument
        |
        v
AZURE_ARTIFACTS_TOKEN
        |
        v
Docker SDK container
        |
        v
Azure Artifacts Credential Provider
        |
        v
VSS_NUGET_EXTERNAL_FEED_ENDPOINTS
        |
        v
nuget.config
        |
        v
Esyasoftpackages
        |
        v
Esyasoft.Identity.SSO.Middleware
        |
        v
Package downloaded
        |
        v
dotnet restore succeeds
```

---

# 14. Error: NU1101

Example:

```text
error NU1101:
Unable to find package Esyasoft.Identity.SSO.Middleware.

No packages exist with this id in source(s): nuget.org
```

This means NuGet only searched:

```text
nuget.org
```

and did not search the Azure Artifacts feed.

Possible causes:

* `nuget.config` does not exist
* Wrong `nuget.config` path
* File was not copied into Docker
* `.dockerignore` excluded the file
* `dotnet restore` did not use the expected config

Verify with:

```dockerfile
RUN ls -la
RUN cat ./nuget.config
RUN dotnet nuget list source --configfile ./nuget.config
```

---

# 15. Error: nuget.config Not Found

Example:

```text
Error: Not found nugetConfigPath:
/home/vsts/work/1/s/Esyasoft.CRM.GAS.Web/nuget.config
```

This means the pipeline cannot find the file.

Check its location:

```bash
find "$(Build.SourcesDirectory)" -iname "nuget.config" -print
```

Example Azure DevOps diagnostic step:

```yaml
- script: |
    echo "Searching for nuget.config"
    find "$(Build.SourcesDirectory)" -iname "nuget.config" -print
  displayName: Find NuGet Config
```

---

# 16. Error: NU1301

Example:

```text
error NU1301:
Unable to load the service index for source

https://pkgs.dev.azure.com/esyasoft/Esyasoft-Gas-Services/_packaging/Esyasoftpackages/nuget/v3/index.json
```

This is different from `NU1101`.

It means NuGet now knows about the Azure Artifacts feed.

The flow has reached:

```text
nuget.config
      |
      v
Azure Artifacts feed found
      |
      v
Authentication/access failed
```

Typical causes:

* Docker has no Azure Artifacts credentials
* Credential Provider is missing
* Token was not passed into Docker
* Build service does not have permission to read the feed
* Token environment variable was configured incorrectly

---

# 17. Understanding Errors Quickly

Use the following rule when troubleshooting.

### Error shows only nuget.org

```text
No packages exist with this id in source(s): nuget.org
```

Meaning:

```text
Azure Artifacts source is not being used.
```

Check:

```text
nuget.config
```

---

### Error shows Azure Artifacts URL with NU1301

```text
Unable to load the service index for source
https://pkgs.dev.azure.com/...
```

Meaning:

```text
Azure Artifacts source is configured,
but authentication/access is failing.
```

Check:

```text
Credential Provider
Token
Permissions
```

---

### Error shows both feeds but package is missing

Example:

```text
Unable to find package XYZ.

No packages exist with this id in source(s):
Esyasoftpackages, nuget.org
```

Meaning:

```text
Both feeds were searched,
but the package/version could not be found.
```

Check:

* Package ID
* Package version
* Correct Azure Artifacts feed
* Package published successfully

---

# 18. Troubleshooting Checklist

When NuGet restore fails inside Docker, troubleshoot in this order.

## Step 1 - Check nuget.config exists

```dockerfile
RUN ls -la
```

Expected:

```text
nuget.config
```

---

## Step 2 - Print NuGet config

```dockerfile
RUN cat ./nuget.config
```

Make sure the Azure Artifacts source exists.

---

## Step 3 - Check configured sources

```dockerfile
RUN dotnet nuget list source --configfile ./nuget.config
```

Expected:

```text
Esyasoftpackages
nuget.org
```

---

## Step 4 - Check Credential Provider

Make sure the Docker build stage installs the Azure Artifacts Credential Provider.

---

## Step 5 - Check token passing

Pipeline:

```yaml
arguments: >
  --build-arg AZURE_ARTIFACTS_TOKEN=$(VSS_NUGET_ACCESSTOKEN)
```

Docker:

```dockerfile
ARG AZURE_ARTIFACTS_TOKEN
```

---

## Step 6 - Check feed endpoint configuration

```dockerfile
ENV VSS_NUGET_EXTERNAL_FEED_ENDPOINTS="..."
```

The endpoint must exactly match the Azure Artifacts source URL.

---

## Step 7 - Check feed permissions

The build identity should have permission to read the feed.

Example identity:

```text
Esyasoft-Gas-Services Build Service (esyasoft)
```

It should have at least permission equivalent to:

```text
Feed Reader
```

---

## Step 8 - Run restore

```dockerfile
RUN dotnet restore "Esyasoft.CRM.GAS.Web.csproj" \
    --configfile ./nuget.config \
    --no-cache \
    --force
```

---

# 19. Recommended Temporary Debug Section

During troubleshooting, use:

```dockerfile
RUN echo "===== CURRENT DIRECTORY =====" && pwd

RUN echo "===== FILES =====" && ls -la

RUN echo "===== NUGET CONFIG =====" && cat ./nuget.config

RUN echo "===== NUGET SOURCES =====" && \
    dotnet nuget list source --configfile ./nuget.config
```

Remove unnecessary debug output after the pipeline is stable.

Never print the authentication token.

Do not run commands such as:

```bash
echo $AZURE_ARTIFACTS_TOKEN
```

---

# 20. Security Considerations

Do not store credentials directly in:

```text
Dockerfile
nuget.config
Git repository
pipeline YAML
```

Do not use:

```xml
<packageSourceCredentials>
```

with a hard-coded password or PAT committed to source control.

Prefer Azure DevOps-generated temporary credentials.

The authentication flow should be:

```text
NuGetAuthenticate@1
        |
        v
temporary Azure DevOps token
        |
        v
Docker build
        |
        v
Azure Artifacts
```

For stronger security, consider Docker BuildKit secrets instead of normal Docker build arguments.

---

# 21. Package Restore Decision Flow

```text
dotnet restore
     |
     v
Is nuget.config present?
     |
  NO +-----> Fix file location/COPY
     |
    YES
     |
     v
Does NuGet list Esyasoftpackages?
     |
  NO +-----> Fix nuget.config/path
     |
    YES
     |
     v
Can Azure Artifacts service index load?
     |
  NO +-----> Fix authentication/permissions
     |
    YES
     |
     v
Does requested package/version exist?
     |
  NO +-----> Check package ID/version/feed
     |
    YES
     |
     v
Restore succeeds
```

---

# 22. Final Expected Flow

The final working architecture is:

```text
Azure DevOps
    |
    +-- Checkout repository
    |
    +-- NuGetAuthenticate@1
    |       |
    |       +-- VSS_NUGET_ACCESSTOKEN
    |
    +-- Login to ACR
    |
    +-- Docker build
    |       |
    |       +-- .NET SDK 8 container
    |       |
    |       +-- Install Azure Artifacts Credential Provider
    |       |
    |       +-- Receive AZURE_ARTIFACTS_TOKEN
    |       |
    |       +-- Read nuget.config
    |       |
    |       +-- Authenticate to Esyasoftpackages
    |       |
    |       +-- Restore private packages
    |       |
    |       +-- Restore public packages
    |       |
    |       +-- dotnet publish
    |
    +-- Create ASP.NET runtime image
    |
    +-- Push image to ACR
```

---

# 23. Key Lessons

1. `nuget.config` tells NuGet **where to search**.

2. `NuGetAuthenticate@1` tells Azure DevOps **who the pipeline is**.

3. A Docker build is a separate environment from the Azure DevOps agent.

4. Authentication configured on the Azure DevOps agent is not automatically available inside Docker.

5. The Azure Artifacts Credential Provider handles authentication inside the Docker build.

6. `VSS_NUGET_ACCESSTOKEN` can be passed from the pipeline into Docker.

7. `NU1101` usually indicates a package-source/package lookup problem.

8. `NU1301` against `pkgs.dev.azure.com` usually means the Azure Artifacts source is known, but authentication or access is failing.

9. Never hard-code PATs or passwords in the Dockerfile or `nuget.config`.

10. Always verify package sources using:

```bash
dotnet nuget list source --configfile ./nuget.config
```

```
```

