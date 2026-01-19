---
title: "PortSwigger - Insecure Deserialization"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Insecure Deserialization da PortSwigger"
keywords: ["PortSwigger", "Insecure Deserialization", "Deserialization", "Web Security", "Exploit"]
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

### Modifying serialized objects
"This lab uses a serialization-based session mechanism and is vulnerable to privilege escalation as a result. To solve the lab, edit the serialized object in the session cookie to exploit this vulnerability and gain administrative privileges. Then, delete the user carlos."

You can log in to your own account using the following credentials: wiener:peter
1. Intercepte a requisição `GET /my-account?id=wiener` e envie para o Burp repeater.
2. Insira um caractere no final do valor do Cookie, a resposta indicará o campo como vulnerável a insecure deserialization:

```html
<p class=is-warning>
	PHP Fatal error:  Uncaught Exception: unserialize() failed in
	/var/www/index.php:4
	Stack trace:
	#0 {main}
	thrown in /var/www/index.php on line 4
</p>
```

3. Ao decodificar o base64, será exibido um campo `admin` com valor 0. Modifique esse valor para 1 e envie novamente a requisição. Agora o usuário wiener possui permissões de administrador.
4. Remova o usuário `carlos`.
