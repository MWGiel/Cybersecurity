
| Method | Command | Notes |
|---------|------------|--------|
| **Bash/sh** | /bin/sh -i | Runs the shell in interactive mode (`-i`). |
| **Python** | `python3 -c 'import pty; pty.spawn("/bin/sh")'` | Most common method (if Python is available). |
| **Perl** | `perl -e 'exec "/bin/sh";'` | Works if Perl is installed on the system. |
| **Ruby** | `ruby -e 'exec "/bin/sh"'` | Works if Ruby is available. |
| **Lua** | `lua -e 'os.execute("/bin/sh")'` | Less common, but useful if Lua is present. |
| **AWK** | `awk 'BEGIN {system("/bin/sh")}'` | Often available on Unix/Linux systems. |
| **Find** | `find / -name test -exec /bin/sh \;` | Uses `find` to execute a shell. |
| **Vim** | `fim -c ':!/bin/sh'` or `:set shell=/bin/sh` | Niche method, works if Vim is installed. |

```
Wpisz w tym shellu:

bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
Naciśnij Ctrl+Z (wstrzyma proces).

W terminalu Kali wpisz:

bash
stty raw -echo; fg
Naciśnij Enter dwa razy. Wpisz export TERM=xterm. Teraz masz w pełni interaktywny shell.
```
