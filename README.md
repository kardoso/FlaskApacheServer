
# Servidor Apache com aplicação Flask
Configuração de um servidor Amazon Lightsail para fazer deploy de uma aplicação Flask com Apache e WSGI

**Observação:** _Os links e endereços de IP referentes ao servidor não estão mais disponíveis._

## Resumo do servidor
- O servidor se encontra no IP http://54.226.128.222/
- A conexão SSH é feita através da porta 2200

## Resumo do aplicativo no servidor
Nesse server iremos rodar o aplicativo [CatalogApp](https://github.com/kardoso/CatalogApp), que é um aplicativo que mostra itens organizados por categoria, possui também API de endpoints json, possui criação e edição de itens.

**Observação**: *O aplicativo possui login google, porém é preciso ter um domínio de alto nível. 
A Amazon Lightsail apenas provê o endereço IP para o servidor, portanto não será possível acessar a página de login, de edição ou de criação de itens através do aplicativo.*

## Conteúdo
[1 - Crie uma nova instância no Amazon Lightsail](#1---Crie-uma-nova-instância-no-Amazon-Lightsail)<br>
[2 - Atualizar o sistema](#2---Atualizar-o-sistema)<br>
[3 - Alterar porta ssh](#3---Alterar-porta-ssh)<br>
[4 - Ativando o firewall](#4---Ativando-o-firewall)<br>
[5 - Criar um novo usuário](#5---Criar-um-novo-usuário)<br>
[6 - Permissões sudo para usuário](#6---Permissões-sudo-para-usuário)<br>
[7 - Crie uma chave de acesso para o usuário grader](#7---Crie-uma-chave-de-acesso-para-o-usuário-grader)<br>
[8 - Faça login como grader](#8---Faça-login-como-grader)<br>
[9 - Preparar ambiente de deploy](#9---Preparar-ambiente-de-deploy)<br>
[10 - Preparar aplicativo](#10---Preparar-aplicativo)<br>
[11 - Preparar banco de dados](#11---Preparar-banco-de-dados)<br>
[12 - Editar arquivo de configuração do VirtualHost](#12---Editar-arquivo-de-configuração-do-virtualhost)<br>
[13 - Executar o aplicativo](#13---Executar-o-aplicativo)<br>
[14 - Principais links consultados](#14---Principais-links-consultados)<br>

## 1 - Crie uma nova instância no Amazon Lightsail
Selecione "Somente SO" e Ubuntu 18.04 LTS

Você pode logar na instância agora. Clique em "Conectar usando SSH" e vá para a [atualização do sistema](#2---Atualizar-o-sistema).

Caso queira continuar no seu próprio terminal
- Faça o downloada da chave ssh em "Conta > Chaves SSH".<br>
- Salve na pasta ~/.ssh como "LightsailKey.pem"
- Na pasta .ssh mude a permissão da chave com `chmod 600 LightsailKey.pem`
- Faça login no sistema através do seu terminal com `ssh  ubuntu@54.226.128.222 -p 22 -i .ssh/LightsailKey.pem`
*Altere 54.226.128.222 pelo IP do seu servidor*

## 2 - Atualizar o sistema
Utilize o comando `sudo apt update` para atualizar os pacotes.

Depois de atualizado utilize `sudo apt upgrade` para instalar os pacotes.

**Mantenha a versão local já instalada do grub, do sshd_config e do menu.lst durante essa instalação**

Após isso será preciso reiniciar o sistema. Use o comando `sudo reboot`.

Faça o login novamente.

## 3 - Alterar porta ssh
Utilize o comando `sudo nano /etc/ssh/sshd_config` para editar o arquivo sshd_config.

Encontre a linha onde está `Port 22` e altere para `Port 2200`, caso essa linha estiver comentada, descomente.

Encontre a linha `PasswordAuthentication` e certifique-se de que seu valor seja `no` para permitir apenas conexões ssh.

Encontre a linha `PermitRootLogin yes` e altere para `PermitRootLogin no` para não permitir login com o root, descomente a linha também caso esteja comentada.

Salve o arquivo.

Reinicie o serviço ssh com o comando `sudo service sshd restart`

## 4 - Ativando o firewall
Na aba "Rede" do Lightsail adicione uma nova regra personalizada no firewall com o protocolo TCP com a porta 2200, pois essa será a porta padrão para conexão ssh.

Salve o arquivo.

Na máquina cheque o status do firewall com `sudo ufw status` para ter certeza de que não está ativo

Utilize a regra padrão para negar conexões de entrada
`sudo ufw default deny incoming`

Utilize a regra padrão para permirir conexões de saída
`sudo ufw default allow outgoing`

Permita as conexões de entrada para SSH(porta 2200), HTTP(porta 80) e NTP(porta 123) com os seguintes comandos:

`sudo ufw allow 2200/tcp`

`sudo ufw allow 80/tcp`

`sudo ufw allow 123/tcp`

Ative o firewall `sudo ufw enable`

Cheque o status do firewall com `sudo ufw status`

## 5 - Criar um novo usuário
Utilize o comando `sudo adduser grader` para adicionar um usuario com o nome grader

## 6 - Permissões sudo para usuário
Dê permissões de sudo para o usuário grader:

Crie e abra o arquivo grader dentro da pasta /etc/sudoers.d com o comando
`sudo nano /etc/sudoers.d/grader`

Dentro do arquivo insira a seguinte linha: `grader ALL=(ALL) NOPASSWD:ALL`

Salve o arquivo.

## 7 - Crie uma chave de acesso para o usuário grader
**Atenção! Não gere chaves privadas de acesso no servidor.**

**Localmente** utilize o comando `ssh-keygen` para gerar um par de chaves de autenticação.

Salve a chave com o nome *LightsailGrader*.

Esse comando vai gerar dois arquivos. O arquivo com extensão *.pub* é o arquivo que tem a chave que vai para o servidor.

Agora **no servidor** acesse a home do usuário grader com o comando `cd /home/grader`.

Crie a pasta .ssh com o comando `sudo mkdir .ssh`

Crie o arquivo authorized_keys dentro da pasta com o comando `sudo touch .ssh/authorized_keys`

**Localmente**, leia o arquivo de chave com a extensão *.pub* com `cat .ssh/LightsailGrader.pub`
Copie o conteúdo.

**No servidor**, abra o arquivo authorized_keys com o comando `sudo nano .ssh/authorized_keys` e cole o conteúdo do arquivo *LightsailGrader.pub*

Salve o arquivo.

Altere a permissão da pasta *.ssh* para 700 com o comando

 `sudo chmod 700 .ssh`

E a permissão do arquivo *authorized_keys* para 644 com o comando

`sudo chmod 644 .ssh/authorized_keys`

Altere o dono e o grupo do arquivo *authorized_keys*

`sudo chown grader .ssh/authorized_keys; sudo chgrp grader .ssh/authorized_keys`

Altere o dono e o grupo da pasta *.ssh*

`sudo chown grader .ssh; sudo chgrp grader .ssh`

Faça logoff utilizando o comando `exit`

## 8 - Faça login como grader
**Localmente** a sua chave privada *LightsailGrader* deve estar na pasta *~/.ssh*

Defina a permissão da chave com `chmod 600 ~/.ssh/LightsailGrader`

Faça login com o comando:
`ssh grader@54.226.128.222 -p 2200 -i ~/.ssh/LightsailGrader`

**Lembrando que 54.226.128.222 deve ser substituído pelo IP do seu servidor**

Defina a time zone para UTC com `sudo dpkg-reconfigure tzdata`. Selecione *None of the above* e *UTC*

## 9 - Preparar ambiente de deploy
Cheque se python3 está instalado com `python3 --version`, se não estiver instalado instale com `sudo apt install python3`

Instalar pip com `sudo apt install python3-pip`

Instalar apache com `sudo apt install apache2`

Instalar WSGI com `sudo apt install libapache2-mod-wsgi-py3`

Nesse momento já é possível ver o site padrão do apache acessando o IP público do servidor através do navegador.

## 10 - Preparar aplicativo
- Cheque se o git está instalado com `git --version`, se não estiver instalado instale com `sudo apt install git`
- Vá até o diretório */var/www* com `cd /var/www`
- Baixe o aplicativo com git clone na pasta *catalog_app*:
`sudo git clone https://github.com/kardoso/CatalogApp.git catalog_app`
- Instale os requerimento para o aplicativo com `sudo pip3 install -r catalog_app/requirements.txt`

## 11 - Preparar banco de dados
- Vá até o diretório */var/www/catalog_app*
- Monte o banco de dados com o comando `sudo python3 initdb.py`

## 12 - Editar arquivo de configuração do VirtualHost
Abra o arquivo de configuração com `sudo nano /etc/apache2/sites-enabled/000-default.conf`
Insira o seguinte conteúdo no arquivo:
**Altere 54.226.128.222 pelo IP do seu servidor**
```
<VirtualHost *:80>
    ServerName 54.226.128.222

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/catalog_app

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    WSGIDaemonProcess catalog user=grader group=grader threads=5

    WSGIProcessGroup catalog
    WSGIApplicationGroup %{GLOBAL}

    WSGIScriptAlias / /var/www/catalog_app/wsgi.py

    <Directory /var/www/catalog_app>
            Order allow,deny
            Allow from all
    </Directory>
</VirtualHost>
```

## 13 - Executar o aplicativo
Reinicie o apache com `sudo apache2ctl restart`

O servidor deve está funcionando e já pode ser acessado agora.

Acesse o IP http://54.226.128.222/ para checar.

Se algum erro ocorrer cheque os logs com o comando `sudo tail /var/log/apache2/error.log`

## 14 - Principais links consultados
[Mod Wsgi(Apache) - Flask tutorial](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/)

[Flask App no Apache2](https://blog.lucaskatayama.com/posts/2015/12/02/Flask-Apache2/)

[SQLite relative and absolute path](https://gist.github.com/ekiara/7676136)

[Python Flask Fazendo deploy com Apache](http://www.devfuria.com.br/python/flask-apache/)
