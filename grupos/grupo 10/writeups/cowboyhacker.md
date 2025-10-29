# Lookup (TryHackMe)

  - Feito em 17/09  /2025
  - [Acessar desafio](https://tryhackme.com/room/cowboyhacker)


## Objetivo

Capturar as flags `user.txt` e `root.txt`.


## Grupo

Matrícula Extraordinária (Iasmim, Lucas Sala, Pedro Paçoca)


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



### 1 - Reconhecendo as portas abertas
Rodar o NMAP e vemos que temos ftp, ssh e http. 
```
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### 2 - Pegando informações do FTP
Entramos no ftp com usuario Anonymous que nos permite entrar sem senha. Dentro do servidor depois de usar o comando ls vemos 2 arquivos e os baixamos.
```
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

### 3 - Usando a informação

No arquivo task podemos ver o nome de quem escreveu a task: lin. Alem disso no arquivo locks temos diversas palavras que parecem senhas. Usando o Hydra para brutar essas senhas com o usuario lin no SSH conseguimos logar com a senha : RedDr4gonSynd1cat3 e pegamos a flag de usuario.

### 4 - Escalando privilégio
Para escalar privilegio primeiro vemos quais comandos podemos executar como sudo usando sudo -l . Com isso vemos que podemos usar o comando tar. Pesquisando no gtfobins temos como criar uma shell atraves desse comando logo apenas copiamos o comando e executamos como sudo.
```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
Pronto temos acesso root a máquina.

