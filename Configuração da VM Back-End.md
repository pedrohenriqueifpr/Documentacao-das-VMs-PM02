# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Back-End**. Ela será responsável por executar a nossa **API** utilizando o **Java 17**, ela terá um usuário dedicado **backend** que executará o processo, e tambem um script de inicialização que executará automaticamente ao ligar a maquina.

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

> Mudaremos o nome de **localhost** para **backend** (alterações no hostname são aplicadas após um reboot).

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

### 💾 6. Primeiros passos para realizar o backup

>Criar chaves e enviar para VM Front

---

### 📜 7. Configurar o script de backup

>Script de backup 

---
