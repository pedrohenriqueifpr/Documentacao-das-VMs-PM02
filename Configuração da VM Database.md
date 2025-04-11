# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Database**. Ela será responsável por armazenar o banco de dados que será utilizado pela API para armazenar as informações necessarias para seu funcionamento, usaremos **MariaDB** como sgbd e para acesso e configuração do banco o **mysql-client**.

---

### 📋 1. Clonando a VM Base

Começaremos clonando a **VM base**, que já foi criada previamente.  
A configuração dessa VM base está documentada no **Arquivo de configuração da VM base**.

---

### 🛠️ 2. Configurações Iniciais da VM Database

Antes de passar para as configurações dessa VM vamos personalizar a identidade dela e garantir que ela esteja preparada para se comunicar corretamente com as outras.

---

#### 🏷️ Alterando o Hostname

Para diferenciar a VM Database das demais, alteraremos seu hostname para **database**, editando o arquivo:

```bash
vi /etc/hostname
```

> As alterações no hostname só têm efeito após um reboot:

---

#### 🔐 Definindo uma senha segura para o usuário root

Se ainda **não definiu uma senha forte para o root** durante o setup da VM, faça-o:

```bash
passwd
```

> Crie uma senha segura e guarde-a. Considere uma senha com no mínimo 12 caracteres, incluindo letras maiúsculas, minúsculas, números e símbolos, já que viemos da VM-base, a senha antiga era root (nada seguro).

---

#### 📡 Configurando resolução de IPs no arquivo /etc/hosts

Para que o mariadb possa saber quem está se conectando a ela ajuste a resolução de ip de hosts, por isso, edite o arquivo:

```bash
vi /etc/hosts
```

E adicione a resolução do ip corretamente:

```
"192.168.0.1" backend.llw
```

> O ip acima é apenas um exemplo, coloque o ip da sua vm back-end no lugar daquele, e saiba que sempre que nessa documentação for referida um ip com fim .llw significa que ele está resolvendo o ip de uma das VM's, Front, Back ou Database.

> Priorize um **reboot** da maquina, após mudanças no arquivo de hosts, porque por mais que "não precise", outros serviços lerão o arquivo de hosts somente na hora do boot, e não vao atualizando sua leitura, por isso nomes podem não ser resolvidos corretamente.

> Isso permite que a VM Front-End resolva o domínio `backend.llw` para o IP especificado, facilitando possíveis mudanças de IP no ambiente sem a necessidade de re-buildar o projeto.

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

### 📜 6. Configurarando o script de backup

Crie o arquivo de script em algum lugar seguro

````bash
vi /root/backup_database
````

> Criamos na pasta root pois o script terá a senha e user mysql, que é importante proteger.

Esse será nosso script:

````bash
#!/bin/ash

# Timestamp para identificar o backup
timestamp=$(date '+%Y-%m-%d_%H-%M-%S')

#Arquivo temporario, para organizar tudo
temp_dir="/tmp/backup_database_$timestamp"

#Localização dos arquivos alvo do backup
dump_file="/tmp/lionlaw_$timestamp.sql"
ssh_key="/root/.ssh/authorized_keys"

#Local e nome do arquivo compactado tar gz
tar_file="/tmp/backup_database_$timestamp.tar.gz"

# Caminhos de destino
remote_user="backup_sys"
remote_host="frontend.llw"
remote_path="/opt/backup/database"

# Variáveis do banco
db_user="USER"
db_pass="SENHA"
db_name="adv"

echo "Criando diretório temporário, Data: $timestamp"
mkdir -p "$temp_dir" || { echo "Erro ao criar diretório temporário, Data: $timestamp"; exit 1; }

echo "Realizando dump do banco de dados, , Data: $timestamp"
mariadb-dump -u"$db_user" -p"$db_pass" "$db_name" > "$dump_file" || { echo "Erro ao gerar o dump, Data: $timestamp"; rm -rf "$temp_dir" "$dump_file"; exit 1; }

echo "Copiando dump do banco de dados, Data: $timestamp"
cp "$dump_file" "$temp_dir/" || { echo "Erro ao dump do banco de dados, Data: $timestamp"; rm -rf "$temp_dir" "$dump_file"; exit 1; }


cp "$ssh_key" "$temp_dir/" || { echo "Erro ao copiar authorized_keys, Data: $timestamp"; rm -rf "$temp_dir" "$dump_file"; exit 1; }

echo "Compactando tudo, Data: $timestamp"
tar -czf "$tar_file" -C "$(dirname "$temp_dir")" "$(basename "$temp_dir")" || { echo "Erro ao compactar arquivos, Data: $timestamp"; rm -rf "$temp_dir" "$dump_file"; exit 1; }

echo "Enviando para $remote_user@$remote_host:$remote_path, Data $timestamp"
scp "$tar_file" "$remote_user@$remote_host:$remote_path/" || { echo "Erro ao enviar o backup via SCP, Data: $timestamp"; rm -rf "$temp_dir" "$dump_file" "$tar_file"; exit 1; }

echo "-> Backup enviado com sucesso, Data: $timestamp"

rm -rf "$temp_dir" "$dump_file" "$tar_file"
exit 0
````

Torne-o executavel com o comando:

````bash
chmod +x /root/backup_database
````

> Agora já será possivel chamar o script manualmente, **/root/backup_database**.

---

## 🕝 7. Agendamento de Script Backup com Crontab

Edite o arquivo de agendamento do service padrão do Linux Alpine com o codigo:

````bash
crontab -e
````

Dentro do arquivo adicione essa linha ao final:

````bash
0 */6 * * * /root/backup_database 1>> /var/log/backup_database.log 2>> /var/log/backup_database_error.log
````

> Essa linha garantirá que o script será executado no **minuto 0** a cada **6 horas**, e também redireciona a saida padrão **stdout 1>>** para um arquivo de log, e a saida de erros **stderr 2>>** para um arquivo de log separado, apenas para erros.

---