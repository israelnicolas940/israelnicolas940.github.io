---
title: "PortSwigger - Web LLM Attacks"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de ataques a LLMs (Large Language Models) da PortSwigger"
keywords: ["PortSwigger", "LLM", "Large Language Model", "Prompt Injection", "Web Security", "AI Security"]
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

## Lab: Exploiting LLM APIs with excessive agency
"To solve the lab, use the LLM to delete the user carlos."

1. Diga a LLM que você é o administrador e pergunte que API ela tem acesso e seus detalhes.
2. Depois, peça para remover o usuário `carlos`.
