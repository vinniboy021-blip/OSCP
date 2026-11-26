# 1. Fundamentos de Rede & Análise de Tráfego

### Wireshark
- Usa libpcap (Linux) / winpcap (Windows).
- **Capture filter**: captura só o que interessa. **Display filter**: mostra só o que quero.
- **Follow TCP stream**: botão direito → *Follow → TCP Stream*.

### tcpdump

```bash
tcpdump -r password.pcap                      # ler pcap
tcpdump -n -r password.pcap | awk '{print $3}' | sort -u | head
tcpdump -n src host 172.16.40.10 -r password.pcap   # filtrar por origem
tcpdump -n dst host 172.16.40.10 -r password.pcap   # filtrar por destino/host
tcpdump -n port 81 -r password.pcap           # filtrar por porta
tcpdump -nX -r password.pcap                  # dump em hexadecimal
tcpdump -i eth0                               # escutar interface
```

Filtragem avançada por header — mostrar só pacotes com flags PSH+ACK (byte 13 do header TCP = `CEUAPRSF` → `00011000` = 24):

```bash
tcpdump -A -n 'tcp[13]=24' -r password.pcap
```

### Descoberta de hosts (L2/ARP)

```bash
arp-scan $IP/24                     # descobrir IPs ativos via ARP
netdiscover                         # quem mais está na rede
netdiscover -i eth0 -r 10.10.10.0/24 -c 20   # MAC + fabricante
nmap -sn -v -PE 192.168.*.1         # ping no gateway / mapeamento interno
```

### Port scan rápido com netcat

```bash
nc -nvv -w 1 -z $IP 3388-3390
nc -v -z 10.0.3.1 1-65000 > file.txt 2>&1
```
