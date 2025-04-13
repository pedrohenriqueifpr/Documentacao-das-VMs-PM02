# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Back-End**. Ela será responsável por executar a nossa **API** utilizando o **Java 17**, ela terá um usuário dedicado **backend** que executará o processo, e tambem um script de inicialização que executará automaticamente ao ligar a maquina.

---

### 📋 1. Clonando a VM Base

Começaremos clonando a **VM base**, que já foi criada previamente.  
A configuração dessa VM base está documentada no **Arquivo de configuração da VM base**.

---

### 🛠️ 2. Configurações Iniciais da VM Back-End

Antes de passar para as configurações dessa VM vamos personalizar a identidade dela e garantir que ela esteja preparada para se comunicar corretamente com as outras.

---

#### 🏷️ Alterando o Hostname

Para diferenciar a VM Back-End das demais, alteraremos seu hostname para **backend**, editando o arquivo:

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

Para que o back-end possa fazer a conexão com o endereço correto ajuste a resolução de ip de hosts, por isso, edite o arquivo:

```bash
vi /etc/hosts
```

E adicione a resolução do ip corretamente:

```
"192.168.0.1" database.llw
```

> O ip acima é apenas um exemplo, coloque o ip da sua VM Database no lugar daquele, e saiba que sempre que nessa documentação for referida um ip com fim .llw significa que ele está resolvendo o ip de uma das VM's, Front, Back ou Database.

> Priorize um **reboot** da maquina, após mudanças no arquivo de hosts, porque por mais que "não precise", outros serviços lerão o arquivo de hosts somente na hora do boot, e não vao atualizando sua leitura, por isso nomes podem não ser resolvidos corretamente.

> Isso permite que a VM Back-End resolva o domínio `database.llw` para o IP especificado, facilitando possíveis mudanças de IP no ambiente sem a necessidade de re-buildar o projeto.

---

### ☕ 3. Instalando o Java 17

Adicione o serviço openjdk17 e todas suas depedencias:

````bash
apk add openjdk17
````

---

### ⚙️ 4. Buildando o Projeto

Antes de buildar o projeto, revise as configurações de conexão com o banco no arquivo **application.properties**:

````properties
spring.datasource.url=jdbc:mysql://database.llw:3306/adv
spring.datasource.username=advocacia
spring.datasource.password=PePeFaFe!05
````

> Nesse caso deve ser preenchido com o Usuário e senha mysql corretos, que foram criados ao configurar o ambiente do banco de dados na **VM Database**, tambem o ip de conexão que nesse caso será um ip resolvido pelo arquivo de hosts na **VM Back-End**, na porta liberada 3306.

No host complete a build do projeto e mande o arquivo jar para o caminho correto na VM:

````bash
scp api.jar root@192.168.105:/opt/backend/
````

> Tenha certeza de que o caminho que está tentando ser mandando o arquivo, realmente exista, caso o contrário falhará.

Agora já é possivel testar o projeto executando o jar com o comando:

````bash
java -jar api.jar
````

>Se todas as informações de conexão com o banco na projeto buildado estiverem corretas, e se o banco de dados estiver configurado corretamente, o projeto spring deve rodar sem nenhum erro.

---

### 📝 5. Criando o script para rodar a API automaticamente

Antes de criarmos o script, será necessário criar um usuário dedicado apenas a rodar a API, pois não será o `root` quem irá fazer isso:

```bash
adduser -h /opt/backend/ backend
```

> Será solicitado a senha e depois a confirmação da senha, usaremos: `PePeFaFe!05`

> Definimos a home do user backend como sendo **/opt/backend/** porque é só ali que ele vai operar, rodando a API.

Agora passamos para criação do script, criamos o arquivo:

```bash
vi /etc/init.d/backend
```

E adicionamos ao script o seguinte conteúdo:

```sh
#!/sbin/openrc-run

description="Lionlaw"
java="/usr/bin/java"
arquivo="/opt/backend/api.jar"
pidfile="/opt/backend/lionlaw.pid"
output_log="/opt/backend/backend.out"
usuario="backend"

depend() {
    need net
    after local
}

start_pre() {
    echo "Iniciando Spring..."
}

start() {
    cd /opt/backend || exit 1
    su -s /bin/sh -c "nohup $java -jar $arquivo > $output_log 2>&1 & echo \$! > $pidfile" $usuario
    echo "Aguardando LionLaw Iniciar..."
    for i in $(seq 1 60); do
        if netstat -tlnp 2>/dev/null | grep -q ":8080 "; then
            echo "Backend LionLaw iniciado!"
            return 0
        fi
        sleep 1
    done
    echo "Falha ao iniciar LionLaw."
    return 1
}

stop() {
    echo "Parando serviço..."
    if [ -f "$pidfile" ]; then
        kill -9 "$(cat $pidfile)" && rm -f "$pidfile"
        echo "LionLaw parado."
    else
        echo "Nenhum serviço rodando na porta 8080."
    fi
}
```

Daremos permissão de execução ao script:

```bash
chmod +x /etc/init.d/backend-api
```

Testar se o script inicia corretamente:

```bash
service backend start
```

E adicionar para iniciar automaticamente com o sistema:

```bash
rc-update add backend boot
```

> Agora toda vez que iniciarmos essa VM, a API vai ser iniciada pelo user backend.

### 📜 6. Configurarando o script de backup

Crie o arquivo de script em algum lugar seguro

````bash
vi /root/backup_backend
````

> Criamos na pasta root, não há nada de sensivel no backup do back, mas no dump do banco é necessario ter a senha e user mysql, oque é importante proteger.

Esse será nosso script:

````bash
#!/bin/sh

#Timestamp para identificar horario dos backups
timestamp=$(date '+%Y-%m-%d_%H-%M-%S')

#Arquivo temporario, para organizar tudo
temp_dir="/tmp/backup_backend_$timestamp"

#Localização dos arquivos alvo do backup
init_script="/etc/init.d/backend"
authorized_keys="/root/.ssh/authorized_keys"

#Local e nome do arquivo compactado tar gz
tar_file="/tmp/backup_backend_$timestamp.tar.gz"

# Caminhos de destino
remote_user="backup_sys"
remote_host="frontend.llw"
remote_path="/opt/backup/backend"

echo "Criando diretórios temporários, Data: $timestamp"
mkdir -p "$temp_dir" || { echo "Erro ao criar diretório temporário, Data: $timestamp"; exit 1; }

echo "Copiando script de inicialização, Data: $timestamp"
cp "$init_script" "$temp_dir/" || { echo "Erro ao copiar o script backend, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "Copiando authorized_keys, Data: $timestamp"
cp "$authorized_keys" "$temp_dir/" || { echo "Erro ao copiar authorized_keys, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "Compactando arquivos, Data: $timestamp"
tar -czf "$tar_file" -C "$(dirname "$temp_dir")" "$(basename "$temp_dir")" || { echo "Erro ao compactar, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "Enviando backup para $remote_host:$remote_path, Data: $timestamp"
scp "$tar_file" "$remote_user@$remote_host:$remote_path/" || { echo "Erro ao enviar o backup via SCP, Data: $timestamp"; rm -rf "$temp_dir" "$tar_file"; exit 1; }

echo "-> Backup enviado com sucesso, Data: $timestamp"

rm -rf "$temp_dir" "$tar_file"
exit 0
````

Torne-o executavel com o comando:

````bash
chmod +x /root/backup_backend
````

> Agora já será possivel chamar o script manualmente, **/root/backup_backend**.

---

## 🕝 7. Agendamento de Script Backup com Crontab

Edite o arquivo de agendamento do service padrão do Linux Alpine com o codigo:

````bash
crontab -e
````

Dentro do arquivo adicione essa linha ao final:

````bash
0 */3 * * * /root/backup_backend 1>> /var/log/backup_backend.log 2>> /var/log/backup_backend_error.log
````

> Essa linha garantirá que o script será executado no **minuto 0** a cada **3 horas**, e também redireciona a saida padrão **stdout 1>>** para um arquivo de log, e a saida de erros **stderr 2>>** para um arquivo de log separado, apenas para erros.

---

## Medidas finais de segurança

Começaremos reforçando as medidas para as conexões ssh:

````bash
PermitRootLogin no
PasswordAuthentication no
````

> Garantimos que não será possivel conectar-se ao root, nem conectar utilizando senha, apenas chaves RSA.

### Firewall

Utilizamos iptables, para bloquear todos o acesso as portas inutilizadas

````bash
apk add iptables
````

Adicionamos as seguintes regras de trafico:

````bash
iptables -P INPUT DROP
````
>Bloqueia qualquer conexão de entrada (menos as **excessões**).

````bash
iptables -P OUTPUT ACCEPT
````
> Permite que a VM envie dados para fora, importante para podermos acessar serviços externos como a **internet**.

````bash
iptables -P FORWARD DROP
````
>Bloqueia a VM de "atuar como roteador", bloqueando o encaminhamento de pacotes entre interfaces de rede. É "opcional" nessa ocasião mas é uma boa pratica, já que essa VM apenas recebe conexões e não encaminha dados para nenhum outro dispositivo, e como isso não deveria acontecer mesmo podemos bloquear se ocorrer.

Agora para as excessões

````bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
````
>Essa é crucial. Ela permite respostas a conexões que já foram estabelecidas, se não a VM só enviaria as requisições mas barraria a resposta, Para a vm Back-End é importante para ela se conectar e receber a resposta do Banco de Dados.


````bash
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
````
>Essa vai liberar a porta 8080 que será utilizada para receber requisições da VM Front-End.


````bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
````
>Essa vai liberar a porta 22 utilizada em nossas VM para conexão SSH, mas a VM Back-End não recebe mais conexão ssh no nosso cenário entao é **opcional**.

---