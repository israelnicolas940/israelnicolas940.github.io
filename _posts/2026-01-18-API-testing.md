---
title: "Port Swigger - API testing"
author: "imendes"
date: 2026-01-18
subject: "Port Swigger labs"
description: "Walkthrough da máquina CodePartTwo (HTB), explorando sandbox escape em js2py para obtenção de execução remota de código e posterior escalada de privilégios."
keywords: ["HTB", "CodePartTwo", "js2py", "sandbox escape", "reverse shell", "SQLite", "sudo", "npbackup-cli"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: \scriptsize
categories: [Hack The Box Machines,Linux, Easy]
---
## Lab: Exploiting an API endpoint using documentation
1.  Em `Target`, no domínio alvo, use engagement tools>Discover content; 
2. Um dos caminhos descobertos é `/api/`, acesse esse caminho para interagir com a API.
3. Selecione a requisição com verb `DELETE` e remova `carlos`.
