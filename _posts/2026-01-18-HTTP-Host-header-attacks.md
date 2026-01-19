---
title: "PortSwigger - Http host header attacks"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de CSRF da PortSwigger"
keywords: ["PortSwigger", "Http", "Host", "Web Security"]
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

## Lab: Basic password reset poisoning
"This lab is vulnerable to password reset poisoning. The user carlos will carelessly click on any links in emails that he receives. To solve the lab, log in to Carlos's account.

You can log in to your own account using the following credentials: wiener:peter. Any emails sent to this account can be read via the email client on the exploit server. "

1. Com o intercept do Burp ligado, vá para a página de password reset insira o username `carlos`.
2. No Burp altere o header `Host` para o domínio que o atacante controla: `https://exploit-<ID>.exploit-server.net/`
3. Envie a requisição com o header Host alterado.
4. Verifique os logs do seu exploit server. O token temporário terá sido enviado pela vítima.
5. Acesse a página de password reset legítima com o token recuperado nos logs: 

```
<LAB-ID>.web-security-academy.net/forgot-password?temp-forgot-password-token=<TOKEN> 
```

6. Troque a senha do usuário `carlos` e acesse a conta dele.

## Lab: Host header authentication bypass
"This lab makes an assumption about the privilege level of the user based on the HTTP Host header.

To solve the lab, access the admin panel and delete the user carlos. "

1. Ao tentar acessar o caminho `/admin`, a resposta será `401 Unauthorized`, com body `Admin interface only available to local users`
2. No Burp Repeater modifique o header Host para `localhost`  e envie a requisição. Agora, você terá acesso ao painel de admin.
3. Remova o usuário `carlos`
