---
title: "PortSwigger - GraphQL Vulnerabilities"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de vulnerabilidades em GraphQL da PortSwigger"
keywords: ["PortSwigger", "GraphQL", "API Security", "Web Security", "Vulnerabilities"]
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

## Lab: Accessing private GraphQL posts
"The blog page for this lab contains a hidden blog post that has a secret password. To solve the lab, find the hidden blog post and enter the password."

1. Com o Burp proxy ligado, clique em um blog post qualquer e o endpoint será descoberto: `/graphql/v1`
2. Envie a requisição para o repeater e envie uma requisição de introspecção, dois dos fields encontrados para o objeto BlogPost são `postPassword` e `isPrivate`.
3. Observando os blog posts recebidos na home do website, é possível perceber que o post de id 3 está faltando.
4. Realize a requisição para o post de id 3 com o field `postPassword`:
```graphql
 query getBlogPost($id: Int!) {
        getBlogPost(id: $id) {
            image
            title
            author
            date
            paragraphs
            postPassword
        }
    }
```
5. Submeta a senha recuperada na query.

---
