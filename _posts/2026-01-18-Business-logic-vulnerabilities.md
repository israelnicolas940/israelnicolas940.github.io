---
title: "PortSwigger - Business Logic Vulnerabilities"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Business Logic Vulnerabilities da PortSwigger"
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

## Lab: Excessive trust in client-side controls
"This lab doesn't adequately validate user input. You can exploit a logic flaw in its purchasing workflow to buy items for an unintended price. To solve the lab, buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter"

1. Autentique-se com a conta do `wiener`.
2. Ao adicionar um produto no carrinho, um dos parâmetros nos dados é o preço. Diante disso, intercepte a requisição para adicionar um item no carrinho e altere o parâmetro `price` para 1:
```
POST /cart HTTP/2
[...]
[Headers]
[...]

productId=1&redir=PRODUCT&quantity=1&price=1
```
3. Envie a requisição `POST /cart/checkout` para comprar o produto "Lightweight l33t leather jacket" a um preço muito reduzido. 

## Lab: High-level logic vulnerability
"This lab doesn't adequately validate user input. You can exploit a logic flaw in its purchasing workflow to buy items for an unintended price. To solve the lab, buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter"

1. Autentique-se com a conta wiener.
2. Tentativa 1: na requisição POST /cart, altere o parâmetro para uma quantidade negativa (productId=2&redir=PRODUCT&quantity=-1).
3. Observe que a resposta indica validação apenas do total no server side, não da quantidade individual.
4. Envie uma requisição adicionando -1183 unidades do produto “Babbage Web Spray”.
5. Adicione 1 unidade do “Lightweight "l33t" Leather Jacket”.
6. O total do carrinho ficará $0.21, permitindo finalizar a compra do item.

## Lab: Inconsistent security controls
"This lab's flawed logic allows arbitrary users to access administrative functionality that should only be available to company employees. To solve the lab, access the admin panel and delete the user carlos."

1. Registre-se como um usuário qualquer e acesse o caminho `/admin`. O acesso será negado, mas a resposta indica que usuários `DontWannaCry` possuem permissão.
2. Modifique o email do usuário criado por `@dontwannacry.com`. Com isso o acesso ao painel de administrador será concedido e poderá remover o usuário `carlos`.

## Lab: Flawed enforcement of business rules
"This lab has a logic flaw in its purchasing workflow. To solve the lab, exploit this flaw to buy a "Lightweight l33t leather jacket".

You can log in to your own account using the following credentials: wiener:peter"

1. No fim da Home assine a newslatter, ponha um e-mail qualquer. Será entregue um cupom de desconto: `SIGNUP30`  
2. Autentique-se como wiener.
3. Insira o produto no carrinho: 
```
POST /cart HTTP/2
...
<Headers>
...

productId=1&redir=PRODUCT&quantity=1
```
4. Alternadamente insira os cupons `SIGNUP30` e `NEWCUST5` até que o preço total do produto seja 0: 
```
POST /cart/coupon HTTP/2
<Headers>

csrf=<CSRF>&coupon=SIGNUP30
```
```
POST /cart/coupon HTTP/2
<Headers>

csrf=<CSRF>&coupon=SIGNUP30
```
5. Envie a requisição `POST /cart/checkout` para realizar a compra do "Lightweight l33t leather jacket" por $0.
