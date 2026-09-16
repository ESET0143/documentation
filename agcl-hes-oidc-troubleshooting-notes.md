# AGCL HES UI / OIDC Troubleshooting Notes

## Problem Summary

The AGCL HES UI was returning HTTP 500 errors.

Initial application error:

```text
System.Security.Cryptography.CryptographicException:
An error occurred while trying to encrypt the provided data.

Inner exception:
System.Net.Http.HttpRequestException:
Response status code does not indicate success: 401 (Unauthorized).

at Esyasoft.Identity.SSO.Middleware.Extensions.CustomDataProtectionKeyRepository.GetAllElements()
```

This showed that the HES UI was failing while loading Data Protection keys from the SSO/OIDC service.

---

## Environment

### HES Services

```yaml
agcl-hes-api:
  image: esyasoft/gasservices:esyasoft-hes-gas-api
  container_name: AGCL-HES-API
  network_mode: "host"
  environment:
    - ASPNETCORE_URLS=http://0.0.0.0:4001
    - Oidc_URL=http://localhost:4000

agcl-hes-ui:
  image: esyasoft/gasservices:esyasoft-hes-gas-ui
  container_name: AGCL-HES-UI
  network_mode: "host"
  environment:
    - ASPNETCORE_URLS=http://0.0.0.0:8001
    - API_URL=http://localhost:4001
    - Oidc_URL=http://localhost:4000
    - DOTNET_ENVIRONMENT=Production
```

### OIDC

Container:

```text
AGCL-OIDC
```

OIDC is listening on port:

```text
4000
```

The actual working listener is HTTP:

```text
http://localhost:4000
```

Test:

```bash
curl -I http://localhost:4000/
```

Result:

```text
HTTP/1.1 302 Found
Location: /mdm
```

HTTPS directly against port 4000 failed:

```text
SSL routines::wrong version number
```

So Nginx terminates HTTPS and proxies to OIDC over HTTP.

---

## Nginx Configuration

The SSO domain is:

```text
https://sso-agcl-tnd.etpltech.com
```

Nginx forwards it to OIDC:

```nginx
server_name sso-agcl-tnd.etpltech.com;

location / {
    proxy_pass http://127.0.0.1:4000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Nginx itself was working correctly.

---

## DNS Issue Found Earlier

At one stage the HES UI was trying to use:

```text
sso-agcl-tnd.esyasoft.com
```

This domain did not resolve:

```bash
nslookup sso-agcl-tnd.esyasoft.com
```

Result:

```text
NXDOMAIN
```

Correct domain:

```text
sso-agcl-tnd.etpltech.com
```

Correct DNS resolution:

```bash
getent hosts sso-agcl-tnd.etpltech.com
```

Result:

```text
20.231.125.191 sso-agcl-tnd.etpltech.com
```

The current HES `appsettings.json` was corrected to:

```json
"SSO.API": {
  "AuthURL": "https://sso-agcl-tnd.etpltech.com",
  "ClientId": "agcl-tnd",
  "ClientSecret": "App#4321",
  "Scopes": ["api", "resource_server", "offline_access"]
}
```

After recreating the container and copying the correct appsettings, the DNS failure stopped being the active issue.

---

## Current Active Error

The active error is now:

```text
Response status code does not indicate success: 401 (Unauthorized)

at Esyasoft.Identity.SSO.Middleware.Extensions.CustomDataProtectionKeyRepository.GetAllElements()
```

Nginx access logs confirmed:

```text
GET /api/DataProtectionKey HTTP/1.1" 401
```

At the same time:

```text
GET /index HTTP/1.1" 500
```

So the failure flow is:

```text
AGCL-HES-UI
    |
    v
CustomDataProtectionKeyRepository
    |
    v
GET https://sso-agcl-tnd.etpltech.com/api/DataProtectionKey
    |
    v
AGCL-OIDC
    |
    v
401 Unauthorized
    |
    v
Data Protection key ring cannot load
    |
    v
Session cookie cannot be encrypted
    |
    v
HES UI returns 500
```

---

## OIDC Authentication Log

OIDC showed exactly why the DataProtection request was rejected:

```text
DataProtectionKey Auth Check. Received ClientId='', Expected ClientId='agcl-tnd'
DataProtectionKey Auth Check. Received Secret Length='0', Expected Secret Length='12'
ClientIdMatch=False, ClientSecretMatch=False
```

This means the HES DataProtection request reaches OIDC, but the expected client credentials are not being sent in the format expected by the OIDC middleware.

---

## Important Comparison: MDM vs HES

The same AGCL OIDC is also used by MDM, and MDM is working.

Both MDM and HES have the same client configuration:

```json
"SSO.API": {
  "AuthURL": "https://sso-agcl-tnd.etpltech.com",
  "ClientId": "agcl-tnd",
  "ClientSecret": "App#4321"
}
```

So the configuration values themselves are not the main difference.

### Middleware Versions

MDM:

```text
Esyasoft.Identity.SSO.Middleware = 3.1.1
Assembly version = 3.1.1.0
Target = net6.0
```

HES:

```text
Esyasoft.Identity.SSO.Middleware = 3.1.5
Assembly version = 3.1.5.0
Target = net8.0
```

Hashes are different:

```text
MDM:
c20af655b97592287f3ce508b44a3e9ae842913b087bafb969f8bf36dac7e47c

HES:
a3595e140e982822dcba47052cec4f25e53fe145b1ab61b17d836aff460b2e20
```

Several Microsoft IdentityModel DLLs also differ between MDM and HES.

---

## Current Main Suspect

The main suspect is now the difference between:

```text
Esyasoft.Identity.SSO.Middleware 3.1.1
```

and:

```text
Esyasoft.Identity.SSO.Middleware 3.1.5
```

Possible causes:

- Middleware 3.1.5 changed how ClientId / ClientSecret are read.
- Middleware 3.1.5 changed the expected configuration contract.
- Middleware 3.1.5 changed the DataProtection request header format.
- Middleware 3.1.5 may contain a regression.
- HES startup/registration code may not match the expectations of middleware 3.1.5.

---

## Recommended Next Checks

### 1. Compare Middleware Registration Code

Search HES source:

```bash
grep -Rni "Esyasoft.Identity.SSO.Middleware" .
grep -RniE "AddDataProtection|DataProtectionKey|AddSSO|SSO.API|UseSSO" .
```

Compare with MDM source, especially:

```text
Program.cs
Startup.cs
```

---

### 2. Compare Package References

HES:

```text
Esyasoft.Identity.SSO.Middleware 3.1.5
```

MDM:

```text
Esyasoft.Identity.SSO.Middleware 3.1.1
```

Check `.csproj` files.

---

### 3. Test HES with Middleware 3.1.1

For a controlled test, rebuild HES using:

```xml
<PackageReference Include="Esyasoft.Identity.SSO.Middleware"
                  Version="3.1.1" />
```

Then build a new HES image and test.

Do not manually copy only the MDM middleware DLL into the HES container because IdentityModel dependency versions also differ.

---

## Container Configuration Note

For future deployments, it is better to mount appsettings directly instead of copying it after container startup.

Example:

```yaml
volumes:
  - ./appsettings.hesui.json:/app/appsettings.json:ro
  - ./agcl-hes-ui.Logs/Logs:/app/Logs
```

Then recreate:

```bash
sudo docker-compose up -d --force-recreate agcl-hes-ui
```

This avoids the application starting with stale configuration from the Docker image.

---

## Current Conclusion

The DNS issue was fixed.

The active blocker is:

```text
GET /api/DataProtectionKey
→ 401 Unauthorized
```

OIDC confirms that the HES request arrives with empty ClientId and ClientSecret.

Since MDM works against the same OIDC but uses SSO Middleware 3.1.1 while HES uses 3.1.5, the most likely remaining issue is a middleware-version/configuration-registration difference in HES.
