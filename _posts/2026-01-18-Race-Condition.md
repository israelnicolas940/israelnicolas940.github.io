---
title: "PortSwigger - Race Condition"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Race Condition da PortSwigger"
keywords: ["PortSwigger", "Race Condition", "Concorrência", "Web Security", "Concurrency"]
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

## Lab: Limit overrun race conditions
"This lab's purchasing flow contains a race condition that enables you to purchase items for an unintended price.

To solve the lab, successfully purchase a Lightweight L33t Leather Jacket.

You can log in to your account with the following credentials: wiener:peter.

For a faster and more convenient way to trigger the race condition, we recommend that you solve this lab using the Trigger race conditions custom action. This is only available in Burp Suite Professional."

1. Autentique-se e ponha a **Lightweight L33t Leather Jacket** no carrinho.
2. Intercepte a requisição de adicionar um cupom e envie para o Repeater.
3. Use custom action [ProbeForRaceCondition](https://github.com/PortSwigger/bambdas/blob/main/CustomAction/ProbeForRaceCondition.bambda) para facilitar o teste. Modifique `NUMBER_OF_REQUESTS` para um número acima de 20 pelo menos (valor escolhido de forma arbitrária para obter um preço pequeno o suficiente para ser comprado com o crédito do usuário que temos controle).
4. Faça o ataque com o custom action e compre o item a um preço muito reduzido.
