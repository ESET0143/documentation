# Notes: Publishing a Private NuGet Package to Azure Artifacts

These notes summarize the complete process we followed to upload a company package (`EsyaSecurity.1.4.0`) to an Azure Artifacts feed.

---

# Architecture

```text
Developer
      │
      │ nuget push
      ▼
Azure Artifacts Feed
(private_packages)
      │
      │ dotnet restore / nuget restore
      ▼
Projects
```

Instead of storing `.nupkg` files inside each repository, the package is stored once in Azure Artifacts and consumed from there.

---

# Prerequisites

* Azure DevOps Project
* Azure Artifacts Feed
* `nuget.exe` (or .NET SDK)
* Personal Access Token (PAT) with Packaging permissions
* `.nupkg` package file

---

# Step 1 - Create an Azure Artifacts Feed

Azure DevOps

```
Project
    ↓
Artifacts
    ↓
Create Feed
```

Example

```
Feed Name:
private_packages
```

Feed URL

```
https://pkgs.dev.azure.com/esyasoft/Esyasoft-MDM/_packaging/private_packages/nuget/v3/index.json
```

---

# Step 2 - Download NuGet.exe

Download

```
https://dist.nuget.org/win-x86-commandline/latest/nuget.exe
```

Verify

```cmd
nuget.exe help
```

Expected

```
NuGet Version: x.x.x
```

---

# Step 3 - Create Personal Access Token (PAT)

Azure DevOps

```
Profile
    ↓
Personal Access Tokens
    ↓
New Token
```

Scopes

```
Packaging
    Read
    Write
```

Copy the PAT immediately because Azure DevOps shows it only once.

---

# Step 4 - Add the Azure Artifacts Feed

```cmd
nuget.exe sources Add ^
-Name esya_packages ^
-Source "https://pkgs.dev.azure.com/esyasoft/Esyasoft-MDM/_packaging/private_packages/nuget/v3/index.json"
```

If authentication is required

```
Username:
Melam.Surendra@esyasoft.com

Password:
<Personal Access Token>
```

Verify

```cmd
nuget.exe sources List
```

---

# Step 5 - Locate the Correct Package

There are two items after downloading/extracting:

```
esyasecurity.1.4.0.nupkg      
esyasecurity.1.4.0\           
```

Never push the extracted folder.

Push only

```
EsyaSecurity.1.4.0.nupkg
```

---

# Step 6 - Push the Package

```cmd
nuget.exe push "C:\Users\MelamSurendra\Downloads\esyasecurity.1.4.0.nupkg" ^
-Source esya_packages ^
-ApiKey az
```

Azure Artifacts ignores the actual API key value because authentication is done using the PAT.

---

# Step 7 - Verify

Azure DevOps

```
Artifacts
    ↓
private_packages
```

Expected

```
EsyaSecurity

Version

1.4.0
```

This confirms the package is successfully published.

---

# Consuming the Package

Create `nuget.config`

```xml
<?xml version="1.0" encoding="utf-8"?>

<configuration>

  <packageSources>

    <clear/>

    <add key="nuget.org"
         value="https://api.nuget.org/v3/index.json" />

    <add key="private_packages"
         value="https://pkgs.dev.azure.com/esyasoft/Esyasoft-MDM/_packaging/private_packages/nuget/v3/index.json"/>

  </packageSources>

</configuration>
```

Project file

```xml
<ItemGroup>

    <PackageReference Include="EsyaSecurity"
                      Version="1.4.0"/>

</ItemGroup>
```

Restore

```bash
dotnet restore
```

or

```bash
nuget restore
```

---

# Authentication Methods

## Option 1 (Recommended)

Azure Artifacts Credential Provider

Advantages

* Automatic authentication
* No PAT stored in `NuGet.config`
* Best for developers

---

## Option 2

Personal Access Token (PAT)

Advantages

* Simple
* Works with `nuget.exe`
* Good for manual package publishing

---

# Common Errors

### No .NET SDKs were found

Reason

Only .NET Runtime installed.

Solution

Install .NET SDK or use `nuget.exe`.

---

### The source specified has already been added

Reason

Feed already exists.

Solution

```cmd
nuget.exe sources List
```

or

```cmd
nuget.exe sources Update
```

---

### 401 Unauthorized

Reason

* Wrong PAT
* Expired PAT
* Missing Packaging permissions

Solution

Create a new PAT with **Packaging (Read & Write)**.

---

### Pushing an Extracted Folder

Wrong

```
EsyaSecurity\
    lib\
    package\
```

Correct

```
EsyaSecurity.1.4.0.nupkg
```

---

# Useful Commands

List feeds

```cmd
nuget.exe sources List
```

Add feed

```cmd
nuget.exe sources Add
```

Update credentials

```cmd
nuget.exe sources Update
```

Remove feed

```cmd
nuget.exe sources Remove -Name esya_packages
```

Push package

```cmd
nuget.exe push package.nupkg
```

---

# Best Practices

* Keep internal packages in Azure Artifacts rather than in Git repositories.
* Version packages properly (e.g., `1.4.0`, `1.4.1`, `1.5.0`).
* Use `PackageReference` instead of committing `.nupkg` files.
* Grant access to the feed only to authorized users or projects.
* Rotate PATs regularly and avoid committing them to source control.

---

# Outcome

You successfully:

* Created an Azure Artifacts feed named `private_packages`.
* Configured `nuget.exe` to connect to the feed.
* Authenticated using your Azure DevOps account and a PAT.
* Published `EsyaSecurity` version `1.4.0` to the feed.
* Verified that the package appears in Azure Artifacts and is ready to be consumed by other projects.
