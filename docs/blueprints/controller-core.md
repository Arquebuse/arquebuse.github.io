# Controller-Core

## 1. High-Level Architecture Overview

### Client Registration
- The Desktop-App (Windows service) auto-registers with Windows Push Notification Service (WNS) to obtain a unique channel URI.
- On startup (and periodically, since channel URIs expire after ~30 days), the client requests a new WNS channel, then calls the Controller-Core `POST /register` endpoint over HTTPS, sending its channel URI and identity (e.g. AD username or device ID).
- The Controller-Core authenticates via an LDAP bind to Active Directory, stores the subscription (user/device + channel URI) in a SQLite database, and uses a read-only AD service account for least-privilege queries.

### Notification Dispatch (CLI Trigger)
- An administrator runs the Controller-CLI PowerShell module (e.g. `Send-Notification -Filter @{Group='IT'} -Message '…'`), which invokes `POST /notify` on the Controller-Core, including filter criteria (AD group, specific user, or computer names).
- The Controller-Core authenticates the caller via LDAP, queries AD to expand filters into target user/computer names, looks up their channel URIs in the SQLite store, and proceeds to delivery.

### WNS Push Delivery
- The Controller-Core obtains an OAuth2 access token from WNS using the app’s Package SID and secret (one global app identity).
- For each target channel URI, it issues an HTTPS `POST` with the toast payload and required headers to the WNS endpoint.
- WNS routes the notification to the appropriate device. The Desktop-App receives the push and displays it as a native Windows toast, logging the event locally in JSON.

### Unregistration & Cleanup
- The Desktop-App calls `DELETE /unregister` (e.g. on shutdown) to remove its subscription; the Controller-Core deletes the record.
- If WNS returns “410 Gone” for a channel, the Controller-Core purges that subscription.
- All significant events (registrations, notifications, delivery results, errors) are appended as JSON lines in a server audit log.

---

## 2. .NET 8 Minimal API Project Structure

- **Controllers/**  
  Minimal API endpoint handlers (e.g. `RegistrationController`, `NotificationController`) defining `/register`, `/unregister`, and `/notify`.

- **Services/**  
  Business logic and integrations:
  - `LdapAuthService.cs` (LDAP auth & queries)  
  - `AdQueryService.cs` (AD group/user resolution)  
  - `WnsClient.cs` (WNS token acquisition & push sending)

- **Data/**  
  Persistence layer:
  - `SubscriptionDbContext.cs` (EF Core context for SQLite)  
  - `Subscription.cs` (model class)  
  - `Migrations/` (EF Core migrations)

- **Logging/**  
  Configuration and helpers for structured JSON logs (e.g. Serilog).

- **Database/**  
  Contains SQLite file (`subscriptions.db`) or initialization scripts.

- **Program.cs**  
  Entry point configuring `WebApplication`, Kestrel HTTPS, DI for services and DbContext, endpoint mappings, and security middleware.

- **appsettings.json**  
  Holds configuration for LDAP, database, WNS credentials, and Kestrel.

- **Dockerfile**  
  Builds and publishes the .NET 8 service on a Windows Server Core image, exposes port 5001.

- **docker-compose.yml**  
  Defines the service for local development, mounts volumes for SQLite and certificates, and sets environment variables for secrets.

---

## 3. Key Classes & Functions (Pseudo-code Sketches)

### LdapAuthService
```csharp
public bool Authenticate(string username, string password) {
    using var entry = new DirectoryEntry("LDAP://ldap.example.com/DC=example,DC=com",
                                         username, password) {
        AuthenticationType = AuthenticationTypes.Secure
    };
    try {
        _ = entry.NativeObject;
        return true;
    }
    catch {
        return false;
    }
}

public List<string> QueryGroupMembers(string groupName) {
    using var searcher = new DirectorySearcher {
        Filter = $"(&(objectClass=user)(memberOf=CN={groupName},OU=Groups,DC=example,DC=com))",
        PropertiesToLoad = { "sAMAccountName" },
        PageSize = 1000
    };
    var results = searcher.FindAll();
    return results
        .Cast<SearchResult>()
        .Select(r => (string)r.Properties["sAMAccountName"][0])
        .ToList();
}
```

### SubscriptionRepository
```csharp
public void AddSubscription(string userId, string channelUri) {
    var sub = new Subscription {
        UserOrDeviceId = userId,
        ChannelUri      = channelUri,
        RegisteredAt    = DateTime.UtcNow
    };
    _dbContext.Subscriptions.Add(sub);
    _dbContext.SaveChanges();
}

public void RemoveSubscription(string userId, string channelUri) {
    var sub = _dbContext.Subscriptions
                .FirstOrDefault(s => s.UserOrDeviceId == userId
                                     && s.ChannelUri == channelUri);
    if (sub != null) {
        _dbContext.Subscriptions.Remove(sub);
        _dbContext.SaveChanges();
    }
}

public IEnumerable<Subscription> GetByUserIds(IEnumerable<string> userIds) {
    return _dbContext.Subscriptions
                     .Where(s => userIds.Contains(s.UserOrDeviceId))
                     .ToList();
}
```

### NotificationController
```csharp
[HttpPost("/register")]
public IActionResult Register([FromBody] ClientInfo info, HttpContext ctx) {
    if (!_ldap.Authenticate(info.User, info.Password))
        return Unauthorized();
    _repo.AddSubscription(info.User, info.ChannelUri);
    return Ok();
}

[HttpDelete("/unregister")]
public IActionResult Unregister([FromBody] ClientInfo info) {
    _repo.RemoveSubscription(info.User, info.ChannelUri);
    return NoContent();
}

[HttpPost("/notify")]
public async Task<IActionResult> Notify([FromBody] NotifyRequest req) {
    if (!_ldap.Authenticate(req.Caller, req.Credential))
        return Unauthorized();
    var targets = _adQuery.QueryGroupMembers(req.Filters.Group);
    var subs    = _repo.GetByUserIds(targets);
    foreach (var sub in subs) {
        await _wns.SendToastAsync(sub.ChannelUri, req.Message);
    }
    return Ok(new { Count = subs.Count() });
}
```

### WnsClient
```csharp
private string _cachedToken;
private DateTime _tokenExpiry;

public async Task<string> AcquireAccessTokenAsync() {
    if (DateTime.UtcNow < _tokenExpiry)
        return _cachedToken;

    var form = new Dictionary<string, string> {
        ["grant_type"]    = "client_credentials",
        ["client_id"]     = _sid,
        ["client_secret"] = _secret,
        ["scope"]         = "notify.windows.com"
    };
    var res = await _http.PostAsync("https://login.live.com/accesstoken.srf",
                                    new FormUrlEncodedContent(form));
    var json = await res.Content.ReadAsStringAsync();
    var doc  = JsonDocument.Parse(json);
    _cachedToken = doc.RootElement.GetProperty("access_token").GetString();
    _tokenExpiry = DateTime.UtcNow.AddMinutes(20);
    return _cachedToken;
}

public async Task SendToastAsync(string channelUri, string payloadXml) {
    var token = await AcquireAccessTokenAsync();
    using var req = new HttpRequestMessage(HttpMethod.Post, channelUri) {
        Content = new StringContent(payloadXml, Encoding.UTF8, "text/xml")
    };
    req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
    req.Headers.Add("X-WNS-Type", "wns/toast");

    var res = await _http.SendAsync(req);
    if (res.StatusCode == HttpStatusCode.Gone ||
        res.StatusCode == HttpStatusCode.NotFound) {
        _repo.RemoveSubscriptionForUri(channelUri);
    }
    // log success or error
}
```

---

## 4. Sample Configuration Snippets

### appsettings.json
```json
{
  "LDAP": {
    "Server":                 "ldaps://ad.example.com:636",
    "BaseDN":                 "DC=example,DC=com",
    "ServiceAccount":         "CN=ArquebuseReader,OU=ServiceAccounts,DC=example,DC=com",
    "ServiceAccountPassword": "P@ssw0rd!"
  },
  "ConnectionStrings": {
    "Subscriptions": "Data Source=./Data/subscriptions.db"
  },
  "WNS": {
    "PackageSID": "ms-app://<YourAppSID>",
    "SecretKey":  "<YourWNSSecretKey>"
  },
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url":         "https://0.0.0.0:5001",
        "Certificate": {
          "Path":     "certs/ArquebuseDev.pfx",
          "Password": "DevCertPassword"
        }
      }
    }
  },
  "Logging": {
    "LogLevel": { "Default": "Information" }
  }
}
```

### Dockerfile
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-windowsservercore-ltsc2022 AS final
WORKDIR /app
COPY --from=build /app ./
EXPOSE 5001
ENTRYPOINT ["dotnet", "Arquebuse.ControllerCore.dll"]
```

### docker-compose.yml
```yaml
version: '3.8'
services:
  controller-core:
    build: .
    image: arquebuse/controller-core:dev
    ports:
      - "5001:5001"
    volumes:
      - ./Data:/app/Data
      - ./certs:/app/certs:ro
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
      LDAP__ServiceAccountPassword: "P@ssw0rd!"
      WNS__SecretKey:            "<YourWNSSecretKey>"
      ASPNETCORE_Kestrel__Endpoints__Https__Url:              "https://0.0.0.0:5001"
      ASPNETCORE_Kestrel__Endpoints__Https__Certificate__Path: "/app/certs/ArquebuseDev.pfx"
      ASPNETCORE_Kestrel__Endpoints__Https__Certificate__Password: "DevCertPassword"
```

---

## 5. Step-by-Step Setup Instructions (Development)

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

---

## 6. Best Practices

- **HTTPS Hardening**  
  Enforce HTTPS only, enable HSTS, restrict to TLS 1.2 or higher, and rotate certificates regularly.
- **LDAP Querying**  
  Use LDAPS or StartTLS; apply efficient filters; retrieve only required attributes; implement paging for large groups; and dispose of Directory objects promptly.
- **WNS Credentials**  
  Store Package SID and secret in a secure vault (environment variables, Key Vault, etc.); cache OAuth tokens to minimize token requests; and never log secrets.
- **Logging & Auditing**  
  Emit structured JSON logs with timestamps, user IDs, message IDs, and delivery statuses—avoid logging full channel URIs or tokens.

---

## 7. Integration

### Desktop-App Integration

- On startup, the Windows service obtains a WNS channel URI and calls `POST /register` with AD credentials.
- The app renews its channel periodically (before the ~30-day expiry) and updates the server.
- Incoming toasts are displayed via the Windows Notifications API; events are logged locally in JSON.

### Controller-CLI Integration

- A PowerShell module offers cmdlets (e.g. `Get-Subscribers`, `Send-Notification`) that wrap the Controller-Core REST API.
- Authentication uses AD credentials; the CLI handles JSON payloads and HTTP calls.
- Supports skipping certificate validation in dev, and can be scripted for automation scenarios.
