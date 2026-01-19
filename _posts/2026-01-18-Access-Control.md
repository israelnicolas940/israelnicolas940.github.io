---
title: "PortSwigger - Access Control"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de controle de acesso da PortSwigger, com foco em falhas de autorização e bypasses."
keywords: ["PortSwigger", "Access Control", "Authorization Bypass", "IDOR", "Logic Flaws", "Web Security"]
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

## Lab: Unprotected admin functionality
"This lab has an unprotected admin panel.

Solve the lab by deleting the user carlos."

1. Acesse a URL `https://LAB-ID.web-security-academy.net/robots.txt` que contém o seguinte conteúdo:
```
User-agent: *
Disallow: /administrator-panel
```
2. Acesse o painel do administrador pelo endpoint `/administrator-panel` e delete o usuário `carlos`

## Lab: Unprotected admin functionality with unpredictable URL
"This lab has an unprotected admin panel. It's located at an unpredictable location, but the location is disclosed somewhere in the application.

Solve the lab by accessing the admin panel, and using it to delete the user carlos."

1. Acesse o código fonte da página root. Nele encontra-se o código fonte que vaza o endpoint do painel de administrador.
```html
<script> 
var isAdmin = false; 
if (isAdmin) {
	(...)
	adminPanelTag.setAttribute('href', '/admin-nelz55'); 
	(...)
}
</script>
```
2. Acesse o endpoint do painel de administrador vazado e delete o usuário `carlos`http://burpsuite/show/1/aeutg6325iwlaeuej8tgilh7tnwgpv0x

## Lab: User role controlled by request parameter
"This lab has an admin panel at /admin, which identifies administrators using a forgeable cookie.

Solve the lab by accessing the admin panel and using it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter"

1. Intercepte as requisições pelo Burp, observe a requisição do endpoint `GET /my-account?id=wiener` após realizar o login.
2. Observer que o cookie possui o parâmetro Admin `Cookie: session=FdfKox3WwYrDnFL9L5Er7yyY2a1Qo1my; Admin=false`, caso ele seja alterado para `true`, não ocasionará em erros. 
3. Altere o parâmetro Admin para true e acesse o endpoint `/admin`. A respostá conterá uma URL para remover o usuário Carlos (`/admin/delete?username=carlos`). 
4. Faça a requisição para `/admin/delete?username=carlos`  mantendo `Admin=true`.

## Lab: User role can be modified in user profile
"This lab has an admin panel at /admin. It's only accessible to logged-in users with a roleid of 2.

Solve the lab by accessing the admin panel and using it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter"

1. Observe o endpoint `POST /my-account/change-email`. Data:
```
{"email":"email@email.com",
"roleid":1}
```
2. Modifique o `roleid` por 2 e a resposta conterá o roleid 2.
3. Conforme essa descoberta, ao inserirmos o parâmetro de query `?roleid=2` no endpoint admin, o acesso ao painel é obtido: `GET /admin?roleid=2`.
4. Remova o usuário `carlos` com a URL `/admin/delete?username=carlos&roleid=2`.

## Lab: User ID controlled by request parameter
"This lab has a horizontal privilege escalation vulnerability on the user account page.

To solve the lab, obtain the API key for the user carlos and submit it as the solution.

You can log in to your own account using the following credentials: wiener:peter"

1. Intercepte e envie para o Burp repeater a requisição: `GET /my-account?id=wiener
2. Modifique o parâmetro de id para `carlos` e submeta a API Key desse usuário.

## Lab: User ID controlled by request parameter, with unpredictable user IDs
"This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with GUIDs.

To solve the lab, find the GUID for carlos, then submit his API key as the solution.

You can log in to your own account using the following credentials: wiener:peter"

1. No seguinte post `https://LAB-ID.web-security-academy.net/post?postId=9` usuário `carlos` foi quem o enviou.  Na página da postagem o id do usuário carlos é vazado: 
```html
<span id=blog-author><a href='/blogs?userId=<CARLOS-ID>'>carlos</a></span>
```
2. Intercepte a requisição de acesso à conta do `wiener`: `GET /my-account?id=<WIENER-ID>` e modifique o id para o id do `carlos` descoberto no passo anterior.
3. Submeta a API key de `carlos`

## Lab: User ID controlled by request parameter with data leakage in redirect
"This lab contains an access control vulnerability where sensitive information is leaked in the body of a redirect response.

To solve the lab, obtain the API key for the user carlos and submit it as the solution.

You can log in to your own account using the following credentials: wiener:peter"

1. Intercepte e envie para o Burp repeater a requisição: `GET /my-account?id=wiener
2. Modifique o id para `carlos` e a resposta recebida será de redirecionamento, mas com os dados do usuário `carlos` vazados. 
3. Submeta a API key de `carlos`

## Lab: User ID controlled by request parameter with password disclosure
"This lab has user account page that contains the current user's existing password, prefilled in a masked input.

To solve the lab, retrieve the administrator's password, then use it to delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter"

1. Intercepte e envie para o Burp repeater a requisição: `GET /my-account?id=wiener
2. Modifique o id para `administrator` e recupere a senha do administrador vazada no campo `Update Password` obtida na resposta da requisição.
3. Acesse a conta do administrador e remova o usuário `carlos` (`https://LAB-ID.web-security-academy.net/admin/delete?username=carlos`).

## Lab: Insecure direct object references
"This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs.

Solve the lab by finding the password for the user carlos, and logging into their account."

1. No painel de chat, a requisição de recuperar o transcript realiza a seguinte requisição:  `GET /download-transcript/2.txt`. Iniciando com endpoint 2.txt.
2. Modifique o identificador do transcript: `GET /download-transcript/1.txt`. A resposta será uma conversa com uma senha vazada  Essa senha pode ser utilizada para autenticar-se como `carlos`
