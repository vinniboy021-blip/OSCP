# 2. Recon & Nmap

### 2.1 Recon Passivo
Objetivos: identificar IPs, sites externos, usuários, tecnologias, conteúdo, vulnerabilidades e subdomínios.

```bash
theHarvester -d cisco.com -b google -l 500    # emails / hosts / usuários
theHarvester -d cisco.com -b all -l 500
```
- **Shodan.io**, **Netcraft** (`searchdns.netcraft.com`), **SiteDigger**, **punk.sh**.

### 2.2 Nmap — scans principais

```bash
# Full connect (necessário via VPN/proxy):
nmap -sT -p- --min-rate=1000 -vvv -T4 $IP -oA nmap/full-connect

# Rápido com versão/OS:
nmap -O -sV -Pn -v -T4 $IP -oA nmap/quick

# Todas as portas:
nmap -Pn -p- -v -T4 $IP -oA nmap/full

# Scripts default + versão numa porta:
nmap -Pn -p 139 -sC -sV -v -T4 $IP -oA nmap/port139

# Scripts de vulnerabilidade:
nmap -Pn -p- -sV --script "vuln and safe" -vvv -T4 $IP -oA nmap/vuln

# Rápido através de proxy (connect scan, sem OS):
nmap -sT -Pn -v -T4 $IP -oA nmap/proxy
```

### 2.3 Nmap — coleção de scans úteis

```bash
nmap -sS $IP                                   # SYN stealth
nmap -sF $IP                                   # FIN scan
nmap -sV -sT $IP                               # banner grabbing
nmap -O $IP                                    # fingerprint de OS
nmap -p 1-65535 -sV -sS -A -T4 $IP -oN nmap.txt        # enum completa
nmap -oN nmap2.txt -v -sU -sS -p- -A -T4 $IP           # TCP+UDP full
nmap -sV -T4 -O -F --version-light $IP         # rápido plus
nmap -sn $IP/24                                # ping sweep
nmap -sU --open -p 161 $IP-190 -oG snmp.txt    # UDP SNMP
# Comprehensive lento:
nmap -sS -sU -T4 -A -v -PE -PP -PS80,443 -PA3389 -PU40125 -PY -g 53 \
     --script "default or (discovery and safe)" $IP/24
```

### 2.4 NSE por vulnerabilidade (exemplos)

```bash
nmap -v -p 80  --script=http-vuln-cve2010-2861 --script-args vulns.showall $IP   # ColdFusion dir traversal
nmap -v -p 21  --script=ftp-anon.nse $IP                                          # FTP anônimo
nmap -v -p 139,445 --script=smb-security-mode $IP                                 # SMB
nmap -v -p 80  --script=http-vuln-cve2011-3192 --script-args vulns.showall $IP    # checar patch Apache
```

### 2.5 Outros scanners

```bash
uniscan -u $IP -qweds
unicornscan -i tap0 -mT $IP:a
nc -vvn -z  10.10.10.10 1-9000        # scan TCP
nc -vvn -u -z 10.10.10.10 1-9000      # scan UDP
```

### 2.6 Automatizadores (recon rápido)

```bash
# nmapAutomator — encadeia varreduras (Quick/Full/UDP/Vulns/All):
./nmapAutomator.sh $IP All
# AutoRecon — enum paralela por serviço; resultados em ./results/$IP/:
autorecon $IP
autorecon $IP --output ./resultados
# One-liner "portas encontradas → scan profundo":
portas=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d/ -f1 | paste -sd,)
nmap -sC -sV -p$portas $IP -oA nmap/profundo
```
