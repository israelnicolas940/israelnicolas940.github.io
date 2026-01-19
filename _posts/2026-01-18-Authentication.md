---
title: "PortSwigger - Authentication"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de e autenticação da PortSwigger"
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

## Username enumeration via different responses
"This lab is vulnerable to username enumeration and password brute-force attacks. It has an account with a predictable username and password, which can be found in the following wordlists:
    Candidate usernames
    Candidate passwords
To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page. "
1. Acesse a página de `/login`, intercept a requisição de log in, copie e cole a requisição HTTP em um arquivo.
2. Execute o seguinte script para realizar um ataque de força bruta no sistema de autenticação, com as wordlists disponibilizadas pelo PortSwigger: 
```bash
#!/bin/bash
ffuf -request ./http_request.txt -request-proto https \
  -w ./username_wordlist.txt:FUZZ_USER -w ./password_wordlist.txt:FUZZ_PASS \
  -r -fw 1307
```

## Lab: 2FA simple bypass
"This lab's two-factor authentication can be bypassed. You have already obtained a valid username and password, but do not have access to the user's 2FA verification code. To solve the lab, access Carlos's account page.
- Your credentials: `wiener:peter`
- Victim's credentials `carlos:montoya`
"
1. Acesse a página de `/login`, autentique-se como `wiener`. 
2. Ao ser redirecionado para a página de `/login2` que espera o input do segundo fator de autenticação, saia dessa página clicando no botão para voltar para a página do lab.
3. Acesse a página do usuário clicando em `My account` no canto superior direito da página (abaixo do header) .
4. Repita o mesmo processo com `carlos` e o lab estará concluído. 

## Lab: Password reset broken logic
"This lab's password reset functionality is vulnerable. To solve the lab, reset Carlos's password then log in and access his "My account" page.
    Your credentials: wiener:peter
    Victim's username: carlos
"
1. Acesse a página de `/login`, clique em resetar a senha e coloque o nome de usuário `wiener`.
2. Na página de email do usuário wiener, clique no link para recuperar a senha.
3. Preencha o formulário com uma nova senha, envie a requisição e intercepte-a pelo Burp. 
4. A requisição interceptada (POST) possui uma parâmetro `username`, modifique-o para `carlos` e a nova senha do usuário `carlos` será aquela preenchida no formulário.

## Lab: Username enumeration via subtly different responses
"This lab is subtly vulnerable to username enumeration and password brute-force attacks. It has an account with a predictable username and password, which can be found in the following wordlists:
    Candidate usernames
    Candidate passwords
To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page. "
```bash
#!/bin/bash
ffuf -request ./http_request.txt -request-proto https \
  -r -o enum_result.json -of json \
  -w ./username_wordlist.txt:FUZZ_USR \
  -fw 1319,1328
```
Ao fazer o login com o usuário encontrado e uma senha qualquer, é possível ver um espaço em braco ao invés de ponto na mensagem de erro.
```bash
#!/bin/bash
ffuf -request ./http_request.txt -request-proto https \
  -r -o enum_result.json -of json \
  -w ./password_wordlist.txt:FUZZ_PASS \
  -fw 1329,1320
```

## Lab: Username enumeration via response timing
"This lab is vulnerable to username enumeration using its response times. To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.
    Your credentials: wiener:peter
    Candidate usernames
    Candidate passwords
"
