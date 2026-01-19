---
title: "PortSwigger - Information Disclosure"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Information Disclosure da PortSwigger"
keywords: ["PortSwigger", "Information Disclosure", "Info Leak", "Web Security"]
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

## Lab: Information disclosure in error messages
"This lab's verbose error messages reveal that it is using a vulnerable version of a third-party framework. To solve the lab, obtain and submit the version number of this framework. "

1. Intercepte a requisição `GET /product?productId=1` e comece a testar caracteres especiais no parâmetro.
2. `GET /product?productId=(` aciona um erro interno e retorna no final da resposta a versão da third party.

---

## Lab: Information disclosure on debug page
"This lab contains a debug page that discloses sensitive information about the application. To solve the lab, obtain and submit the SECRET_KEY environment variable. "

1. Na raiz da página web, examinando o código fonte procurando pela keyword "debug", o seguinte trecho poderá ser encontrado:
```
<!-- <a href=/cgi-bin/phpinfo.php>Debug</a> -->
```
2. Acesse o caminho encontrado, que corresponde a uma página de debug, e submeta a API key (`SECRET_KEY`).

---

## Lab: Source code disclosure via backup files
"This lab leaks its source code via backup files in a hidden directory. To solve the lab, identify and submit the database password, which is hard-coded in the leaked source code."

1. Realize o Scan do Burp ou utilize a ferramenta `gobuster` para directory brute force.
2. O caminho `backup` será retornado, nele haverá um link para arquivo de backup java `backup/ProductTemplate.java.bak`
3. Submeta a senha do banco de dados encontrada nesse arquivo de backup.

---

## Lab: Authentication bypass via information disclosure
"This lab's administration interface has an authentication bypass vulnerability, but it is impractical to exploit without knowledge of a custom HTTP header used by the front-end.

To solve the lab, obtain the header name then use it to bypass the lab's authentication. Access the admin interface and delete the user carlos."

You can log in to your own account using the following credentials: wiener:peter 
1. Na requisição `GET /` modifique o método para `TRACE`. A resposta é 200, indicando que o método `TRACE` pode ser utilizado.
2. Ao tentar acessar o caminho `/admin`, será retornada uma mensagem com o conteúdo: `Admin interface only available to local users`
3. Com o método TRACE, o endereço de IP local é vazado em um header `X-Custom-IP-Authorization: 200.19.177.1`
4. Utilize esse header na requisição, porém alterando o endereço de IP para `127.0.0.1`.
5. Assim, o acesso à página de admin será concedido. Por fim, basta remover o usuário `carlos`.

---

## Lab: Information disclosure in version control history
"This lab discloses sensitive information via its version control history. To solve the lab, obtain the password for the administrator user then log in and delete the user carlos."

1. Acesse o caminho `/.git`.
2. Recupere os arquivos para consulta local por meio do comando `wget`: 
```bash
wget -r -np https://LAB-ID.web-security-academy.net/.git
```
3. Interaja com o git para recuperar a senha do administrador:
```bash
$ git log
commit e9f579772db1438aca529b83ffeccaa395377e42 (HEAD -> master)
Author: Carlos Montoya <carlos@carlos-montoya.net>
Date:   Tue Jun 23 14:05:07 2020 +0000

    Remove admin password from config

commit 3cd071c0a62d7d8df0cd87dcfc1e224c8f21bfd3
Author: Carlos Montoya <carlos@carlos-montoya.net>
Date:   Mon Jun 22 16:23:42 2020 +0000

    Add skeleton admin panel
$ git diff 3cd071c0a62d7d8df0cd87dcfc1e224c8f21bfd3
diff --git a/admin.conf b/admin.conf
deleted file mode 100644
index 6f8244d..0000000
--- a/admin.conf
+++ /dev/null
@@ -1 +0,0 @@
-ADMIN_PASSWORD=<ADMIN-PASSWORD>
diff --git a/admin_panel.php b/admin_panel.php
deleted file mode 100644
index 8944e3b..0000000
--- a/admin_panel.php
+++ /dev/null
@@ -1 +0,0 @@
-<?php echo 'TODO: build an amazing admin panel, but remember to check the password!'; ?>
\ No newline at end of file
```
4. Autentique-se como administrador e remova o usuário `carlos`.
