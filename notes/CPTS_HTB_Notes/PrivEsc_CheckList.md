Pre-Escalation Checklist:

- OS and version
- Kernel version
- Users and groups
- Home directories (SSH keys, .bash_history)
- Sudo privileges
- Cron jobs
- SETUID/SETGID binaries
- Writeable directories/files
- Unmounted drives
- systemctl list-timers
```
cat /etc/systemd/system/*dlq* 2>/dev/null
cat /etc/systemd/system/*drain* 2>/dev/null
cat /etc/systemd/system/*aidx* 2>/dev/null
```
- sudoers for every vhost and user
