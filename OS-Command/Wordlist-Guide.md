# Guia de Wordlists

## Índice
- [Locais das Wordlists](#locais-das-wordlists)
- [Wordlists de Senhas](#wordlists-de-senhas)
- [Wordlists de Usuários](#wordlists-de-usuários)
- [Wordlists de Diretórios](#wordlists-de-diretórios)
- [Wordlists de Subdomínios](#wordlists-de-subdomínios)
- [Wordlists por Serviço](#wordlists-por-serviço)
- [Geração de Wordlists Personalizadas](#geração-de-wordlists-personalizadas)
- [Ferramentas de Geração de Usuários](#ferramentas-de-geração-de-usuários)

---

## Locais das Wordlists

### Verificação Rápida (One-liner)

```shell
# Verificação rápida de disponibilidade de wordlists
ls -la /usr/share/wordlists/rockyou.txt /usr/share/seclists/Passwords/Common-Credentials/ 2>/dev/null && wc -l /usr/share/wordlists/rockyou.txt
```

### Caminhos Padrão no Kali Linux

| Caminho | Descrição |
| :--- | :--- |
| `/usr/share/wordlists/` | Diretório principal de wordlists |
| `/usr/share/wordlists/rockyou.txt` | Lista de senhas mais popular (14M) |
| `/usr/share/wordlists/dirb/` | Wordlists de diretórios do DIRB |
| `/usr/share/wordlists/dirbuster/` | Wordlists do DirBuster |
| `/usr/share/wordlists/wfuzz/` | Wordlists do WFuzz |
| `/usr/share/wordlists/seclists/` | Coleção SecLists |
| `/usr/share/wordlists/fasttrack.txt` | Senhas comuns do Fast Track |
| `/usr/share/wordlists/metasploit/` | Wordlists do Metasploit |

### Caminhos do SecLists

```shell
# Instalar SecLists
apt install seclists

# Localização
/usr/share/seclists/
├── Discovery/
│   ├── DNS/
│   ├── Web-Content/
│   └── Infrastructure/
├── Fuzzing/
├── Passwords/
│   ├── Common-Credentials/
│   ├── Leaked-Databases/
│   └── Default-Credentials/
├── Usernames/
└── Miscellaneous/
```

---

## Wordlists de Senhas

### Uso Geral

| Wordlist | Caminho | Tamanho | Caso de Uso |
| :--- | :--- | :--- | :--- |
| rockyou.txt | `/usr/share/wordlists/rockyou.txt` | 14M | Quebra geral de senhas |
| fasttrack.txt | `/usr/share/wordlists/fasttrack.txt` | 222 | Senhas comuns rápidas |
| common-passwords | `/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt` | 10K | Tentativa inicial rápida |
| best1050 | `/usr/share/seclists/Passwords/Common-Credentials/best1050.txt` | 1050 | Senhas muito comuns |

### Por Complexidade

```shell
# Top 100 senhas (rápido)
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100.txt

# Top 1000 senhas
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000.txt

# Top 10000 senhas
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt

# Top 100000 senhas
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100000.txt

# Top 1 milhão de senhas
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
```

### Credenciais Padrão

```shell
# Usuários/senhas padrão
/usr/share/seclists/Passwords/Default-Credentials/default-passwords.csv
/usr/share/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt
/usr/share/seclists/Passwords/Default-Credentials/ssh-betterdefaultpasslist.txt
/usr/share/seclists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt
/usr/share/seclists/Passwords/Default-Credentials/postgres-betterdefaultpasslist.txt
/usr/share/seclists/Passwords/Default-Credentials/mssql-betterdefaultpasslist.txt
/usr/share/seclists/Passwords/Default-Credentials/mysql-betterdefaultpasslist.txt
```

### Senhas de Bancos de Dados Vazados

```shell
# Senhas vazadas do Hashcat
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
/usr/share/seclists/Passwords/Leaked-Databases/phpbb.txt
/usr/share/seclists/Passwords/Leaked-Databases/myspace.txt
/usr/share/seclists/Passwords/Leaked-Databases/ashley-madison.txt
```

---

## Wordlists de Usuários

### Usuários Gerais

```shell
# Usuários comuns
/usr/share/seclists/Usernames/top-usernames-shortlist.txt
/usr/share/seclists/Usernames/Names/names.txt
/usr/share/seclists/Usernames/cirt-default-usernames.txt

# Usuários Unix
/usr/share/seclists/Usernames/unix-users.txt

# Específicos por serviço
/usr/share/metasploit-framework/data/wordlists/unix_users.txt
/usr/share/metasploit-framework/data/wordlists/default_users_for_services_unhash.txt
```

### Por Nome/Sobrenome

```shell
# Nomes para geração de usuários
/usr/share/seclists/Usernames/Names/familynames-usa-top1000.txt
/usr/share/seclists/Usernames/Names/femalenames-usa-top1000.txt
/usr/share/seclists/Usernames/Names/malenames-usa-top1000.txt
```

---

## Wordlists de Diretórios

### Diretórios Web

```shell
# Wordlists DIRB
/usr/share/wordlists/dirb/common.txt           # 4614 palavras
/usr/share/wordlists/dirb/big.txt              # 20469 palavras
/usr/share/wordlists/dirb/small.txt            # 959 palavras

# Wordlists DirBuster
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt    # 87K
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt   # 220K
/usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt

# SecLists Web Content
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt
```

### Tecnologias Específicas

```shell
# Apache
/usr/share/seclists/Discovery/Web-Content/Apache.fuzz.txt

# IIS
/usr/share/seclists/Discovery/Web-Content/IIS.fuzz.txt

# Tomcat
/usr/share/seclists/Discovery/Web-Content/tomcat.txt

# PHP
/usr/share/seclists/Discovery/Web-Content/Common-PHP-Filenames.txt

# API
/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
/usr/share/seclists/Discovery/Web-Content/api/api-endpoints-res.txt

# CMS
/usr/share/seclists/Discovery/Web-Content/CMS/wordpress.fuzz.txt
/usr/share/seclists/Discovery/Web-Content/CMS/drupal.txt
/usr/share/seclists/Discovery/Web-Content/CMS/joomla-plugins.txt
```

---

## Wordlists de Subdomínios

```shell
# Enumeração de subdomínios DNS
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
/usr/share/seclists/Discovery/DNS/namelist.txt
/usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
/usr/share/seclists/Discovery/DNS/dns-Jhaddix.txt   # 2M+ (abrangente)
```

---

## Wordlists por Serviço

### SSH

```shell
# Brute force SSH com Hydra
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt \
      ssh://$rhost
```

### FTP

```shell
# Brute force FTP com Hydra
hydra -L /usr/share/metasploit-framework/data/wordlists/unix_users.txt \
      -P /usr/share/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt \
      ftp://$rhost
```

### SMB

```shell
# NetExec SMB
nxc smb $rhost -u /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
                   -p /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt
```

### HTTP Basic Auth

```shell
# Hydra HTTP Basic
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/wordlists/rockyou.txt \
      $rhost http-get /admin
```

### WordPress

```shell
# WPScan
wpscan --url http://$rhost --passwords /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt \
       --usernames admin
```

### Kerberos

```shell
# Enumeração de usuários com Kerbrute
kerbrute userenum -d $domain \
         /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
         --dc $dc_ip

# ASREPRoast
impacket-GetNPUsers $domain/ -usersfile /usr/share/seclists/Usernames/Names/names.txt \
                    -dc-ip $dc_ip -format hashcat
```

### Banco de Dados

```shell
# MySQL
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Default-Credentials/mysql-betterdefaultpasslist.txt \
      mysql://$rhost

# MSSQL
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Default-Credentials/mssql-betterdefaultpasslist.txt \
      mssql://$rhost

# PostgreSQL
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Default-Credentials/postgres-betterdefaultpasslist.txt \
      postgres://$rhost
```

---

## Geração de Wordlists Personalizadas

### CeWL - Gerador de Wordlist Personalizada

```shell
# Gerar wordlist a partir de site
cewl http://$rhost -d 3 -m 5 -w custom_wordlist.txt

# Incluir endereços de e-mail
cewl http://$rhost -d 3 -m 5 -e --email_file emails.txt -w custom_wordlist.txt

# Com autenticação
cewl http://$rhost -d 3 -m 5 --auth_type basic --auth_user admin --auth_pass password -w wordlist.txt
```

### Crunch - Gerador de Wordlist

```shell
# Gerar senhas de 4-6 chars com números
crunch 4 6 0123456789 -o numeric.txt

# Gerar senhas com padrão
crunch 8 8 -t @@@@%%%% -o pattern.txt   # 4 minúsculas + 4 dígitos

# Usando charset
crunch 4 8 -f /usr/share/crunch/charset.lst mixalpha-numeric -o mixed.txt
```

### Geração de Wordlist de Usuários

```shell
# De nomes para usuários
# João Silva -> jsilva, joao.silva, silvaj, etc.

# Usando username-anarchy
git clone https://github.com/urbanadventurer/username-anarchy.git
./username-anarchy Joao Silva > usernames.txt

# Padrões manuais
echo -e "jsilva\njoao.silva\nsilwaj\njoaos\nj.silva" > usernames.txt
```

### Combinar Wordlists

```shell
# Mesclar e remover duplicatas
cat wordlist1.txt wordlist2.txt | sort -u > combined.txt

# Remover senhas curtas
awk 'length >= 8' /usr/share/wordlists/rockyou.txt > rockyou_8plus.txt
```

---

## Ferramentas de Geração de Usuários

### username-anarchy

> Gerar variações de nome de usuário a partir de nome/sobrenome

```shell
# Instalar
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy

# Gerar a partir de um único nome
./username-anarchy Joao Silva
# Saída: jsilva, joao.silva, silvaj, joao_silva, etc.

# A partir de arquivo de nomes
./username-anarchy -i names.txt > usernames.txt

# Formato de entrada para names.txt:
# Nome Sobrenome
# Joao Silva
# Maria Santos
```

#### Padrões Comuns Gerados

| Padrão | Exemplo |
| :--- | :--- |
| nome.sobrenome | joao.silva |
| n.sobrenome | j.silva |
| nome_sobrenome | joao_silva |
| nomesobrenome | joaosilva |
| sobrenomenome | silvajoao |
| nsobrenome | jsilva |
| sobrenomen | silvaj |
| nome | joao |
| sobrenome | silva |

### Gerar a partir de Dados CSV/Excel

```shell
# A partir de CSV com formato Nome;Sobrenome
awk -F';' '{printf("%s%s\n", tolower(substr($1,1,1)), tolower($2))}' Users.csv > userlist.txt

# Múltiplos padrões a partir de CSV
awk -F';' '{
  first=tolower($1)
  last=tolower($2)
  print first"."last
  print substr(first,1,1)"."last
  print first"_"last
  print substr(first,1,1)last
  print last substr(first,1,1)
}' Users.csv | sort -u > userlist.txt
```

### Namemash.py

> Outro gerador de nomes de usuário

```shell
# Instalar
git clone https://github.com/superkojiman/namemash.git
cd namemash

# Gerar usuários
python namemash.py names.txt > usernames.txt

# Formato de entrada:
# Nome Sobrenome
```

### Geração Manual de Usuários

```shell
# Script rápido para gerar variações
#!/bin/bash
while read line; do
  first=$(echo $line | cut -d' ' -f1 | tr '[:upper:]' '[:lower:]')
  last=$(echo $line | cut -d' ' -f2 | tr '[:upper:]' '[:lower:]')
  f=${first:0:1}
  l=${last:0:1}
  
  echo "$first.$last"
  echo "$f.$last"
  echo "$first$last"
  echo "$f$last"
  echo "$last$f"
  echo "$first"
  echo "$last"
done < names.txt | sort -u > usernames.txt
```

### Extrair Nomes de Metadados

```shell
# Extrair nomes de autores de documentos
exiftool -Creator -Author -LastModifiedBy *.pdf *.docx *.xlsx 2>/dev/null | \
  grep -v "^$" | cut -d':' -f2 | sort -u > names.txt

# Em seguida, gerar usuários
./username-anarchy -i names.txt > usernames.txt
```

---

## Tabela de Referência Rápida

| Caso de Uso | Wordlist Recomendada |
| :--- | :--- |
| Password spray rápido | `/usr/share/seclists/Passwords/Common-Credentials/best1050.txt` |
| Quebra completa de senhas | `/usr/share/wordlists/rockyou.txt` |
| Fuzzing de diretórios web | `/usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt` |
| Enumeração de subdomínios | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt` |
| Enumeração de usuários | `/usr/share/seclists/Usernames/top-usernames-shortlist.txt` |
| Credenciais padrão | `/usr/share/seclists/Passwords/Default-Credentials/default-passwords.csv` |
| Fuzzing de API | `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` |
| Usuários AD | `/usr/share/seclists/Usernames/Names/names.txt` |

---

## Veja Também

- **[Ataques de Senha](../3.AD-Exploit/3.2.Password-Attacks.md)** - hashcat, John, quebra de credenciais
- **[Ataques Kerberos](../3.AD-Exploit/3.3.Kerberos-Attacks.md)** - Uso de wordlists para ASREPRoast/Kerberoasting
- **[Análise de Aplicações Web](../7.Web-Exploit/7.0.Web-Application-Analysis.md)** - Fuzzing de diretórios, brute force de parâmetros
