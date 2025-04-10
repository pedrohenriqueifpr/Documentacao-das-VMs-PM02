# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Database**. Ela será responsável por armazenar o banco de dados que será utilizado pela API para armazenar as informações necessarias para seu funcionamento, usaremos **MariaDB** como sgbd e para acesso e configuração do banco o **mysql-client**.

---

### 📋 1. Clonando a VM Base

Começaremos clonando a **VM base**, que já foi criada previamente.  
A configuração dessa VM base está documentada no **Arquivo de configuração da VM base**.

---

### 🏷️ 2. Alterando o Hostname

Para diferenciar a VM Back-End das demais, alteraremos seu hostname editando o arquivo:

````bash
vi /etc/hostname
````

> Mudaremos o nome de **localhost** para **database** (alterações no hostname são aplicadas após um reboot).

---

### 🐬 3. Instalando MariaDB

Adicione o serviço do MariaDB e todas suas depedencias, junto do cliente mysql:

````bash
apk add mariadb mysql-client
````

> MariaDB será nosso **SGBD**, mas o mysql-client será usado para acessar a base de dados para, manutenção e configuração de usuários e permissões.

Rode o script de setup do MariaDB com o comando:

````bash
/etc/init.d/mariadb setup
````

Agora edite o arquivo de configuração do MariaDB

````bash
vi /etc/my.cnf.d/mariadb-server.cnf
````

Modifique o conjunto de configuração do mysql:

````bash
[mysqld]
#skip-networking

bind-address=database.llw
port=3306
````

Inicie e adicione o serviço a nivel de boot, para inicar junto com o sistema:

````bash
rc-service mariadb start
rc-update add mariadb boot
````

> Após esses passos o MariaDB vai rodar sempre que ligar VM, e vai receber conexões de todas as interfaces de rede pela porta 3306.

---

### 📦 4. Configurando o ambiente do Banco de Dados

Entrar no mysql-client utilizando o usuario root, sem senha (padrão)

````bash
mysql -u root
````

Adicionaremos uma senha forte ao user root, por questões de segurança e requisito de projeto

````sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'PePeFaFe!05';
````

> A senha do root será a mesma do user advocacia por questões de facilitar a implementação, mas deveriam ser distintas.

Criar a base de dados utilizada pela API:

````sql
create database adv;
````

Agora criaremos o user mysql que será utilizado pela API para acessar a database:

````sql
CREATE USER 'llw'@'backend.llw' IDENTIFIED BY 'PePeFaFe!05';
GRANT SELECT,INSERT,UPDATE,DELETE ON adv.* TO 'llw'@'backend.llw';
````

> Caso seja a primeira execução do back, e o banco estiver vazio, pode ser necessario dar permissões de **CREATE**, mas lembre-se de remove-las depois com **REVOKE**.

> No fim esse usuário só terá permissão para operar na database que será utilizada pela API, e só podera fazer comandos SQL limitados, nada que altere as estruturas das tabelas.

---

### 💾 5. Primeiros passos para realizar o backup

>Criar chaves e enviar para VM Front

---

### 📜 6. Configurar o script de backup

>Script de backup 

---
