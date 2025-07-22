# Controller

## Step-by-Step Setup Instructions (Development)

1. **Generate and Trust a Dev Certificate**

```bash
dotnet dev-certs https -ep certs/ArquebuseDev.pfx -p "DevCertPassword" -t
```

2. **Verify the Certificate Is Trusted**

- On Windows: run `certmgr.msc` and look under **Trusted Root Certification Authorities**.  
- On macOS/Linux: import into your system keychain or trust store.

3. **Set Up a Mock LDAP Directory**

   - **Option A: AD LDS** on Windows – install AD LDS, create an instance, add OUs/users/groups.  
   - **Option B: OpenLDAP Container**

```bash
docker run -p 389:389 \
   --name ldap \
   -e LDAP_DOMAIN="example.com" \
   -e LDAP_ORGANISATION="Example Corp" \
   -e LDAP_ADMIN_PASSWORD="adminpass" \
   osixia/openldap:1.5.0
```

Then use Apache Directory Studio (or similar) to add entries.

4. **Database Migrations and Seeding**

```bash
dotnet build
dotnet ef database update
```

Optionally seed subscriptions via the API or directly via EF Core.

5. **Run the Controller-Core Service**

- **Direct**:

```bash
dotnet run --project Arquebuse.ControllerCore
```

- **Docker Compose**:

```bash
docker-compose up
```

6. **Test Registration and Notification**

```powershell
# Register
$body = @{ user="jdoe"; channelUri="https://wns.notify.windows.com/?token=abcd" }
$cred = Get-Credential -Message "AD credentials"
Invoke-RestMethod -Uri https://localhost:5001/register `
                  -Method POST `
                  -Body ($body | ConvertTo-Json) `
                  -ContentType "application/json" `
                  -Credential $cred
# Notify
$notif = @{ filters=@{ User="jdoe" }; message="Test alert via CLI" }
Invoke-RestMethod -Uri https://localhost:5001/notify `
                  -Method POST `
                  -Body ($notif | ConvertTo-Json) `
                  -ContentType "application/json" `
                  -Credential $cred
```

7. **Trusting Certificates for CLI/Clients**

- **Dev-only workaround** (PowerShell):

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
```

- **Recommended**: install the dev certificate into the OS trust store so normal validation passes.