# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Front-End**. Ela será responsável por hospedar a interface do sistema utilizando o **nginx**, e também atuará como **servidor de backups**, armazenando os arquivos de backup de todas as outras VMs — incluindo ela mesma. Por esse motivo, esta VM tambem possuirá a **chave pública RSA** de cada uma das outras máquinas virtuais em seu usuário **backup_sys**, que tambem será o usuario que em sua home terá os arquivos de backup.

---

### 📋 1. Clonando a VM Base

Começaremos clonando a **VM base**, que já foi criada previamente.  
A configuração dessa VM base está documentada no **Arquivo de configuração da VM base**.

---

### 🛠️ 2. Configurações Iniciais da VM Front-End

Antes de passar para as configurações dessa VM vamos personalizar a identidade dela e garantir que ela esteja preparada para se comunicar corretamente com as outras.

---

#### 🏷️ Alterando o Hostname

Para diferenciar a VM Front-End das demais, alteraremos seu hostname para **frontend**, editando o arquivo:

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

Mesmo com o backend funcional, o Front-End pode falhar nas requisições caso o domínio `backend.llw` (no nosso caso utilizado para o front fazer requisições) não esteja sendo resolvido corretamente. Para resolver isso, edite o arquivo:

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

#### 🖥️ Adicionando resolução no host (Windows)

Mesmo com o front rodando na VM, o navegador que acessa o sistema está no host. Por isso, também é necessário configurar essa resolução no **Windows**:

Edite o arquivo:

```
C:\Windows\System32\drivers\etc\hosts
```

E adicione a mesma linha:

```
192.168.1.106 backend.llw
```

> Lembre-se de abrir o editor de texto como **Administrador** para conseguir salvar as alterações nesse arquivo.

---

### 🌐 3. Instalando o Nginx

Adicione o serviço nginx e todas suas depedencias:

````bash
apk add nginx
````

Inicie e adicione o serviço a nivel de boot, para inicar junto com o sistema:

````bash
rc-service nginx start
rc-update add nginx boot
````

> Após essas mudanças ja deve ser possivel acessar o ip da VM em seu navegador, e visualizar a página 404 padrão do nginx.

---

### 📝 4. Configurando o Nginx

Edite o arquivo de configuração do nginx:

````bash
vi /etc/nginx/http.d/default.conf
````

Adicione logs de acesso e de erros, para melhor rastrear e resoluciona-los (Opcional)

````bash
access_log /var/log/nginx/frontend_access.log;
error_log /var/log/nginx/frontend_error.log;
````

Modifique o conteúdo do bloco location / { ... }:

````bash
location / {
    root /opt/frontend;
    index index.html;
    try_files $uri $uri/ /index.html;
}
````

> Adicione o caminho que ira ficar os arquivos do front na linha **root** e o nome do index na **linha index**, o **try_files** retornar para index é necessário para aplicações SPA, como no nosso caso de um projeto com front em Angular.

Recarregue o nginx para que ele reconheça as modificações feitas no arquivo de configuração:

````bash
service nginx reload
````

---

### ⚙️ 5. Buildando o Front-End

Antes de fazer o build do front, é necessário configurar corretamente o endereço IP da **API Back-End** que será utilizado.

#### 🧪 Arquivos de ambiente no Angular:

O Angular geralmente possui arquivos em:

- `src/environments/environment.ts` *(usado em desenvolvimento)*
- `src/environments/environment.server.ts` *(usado em produção ou build final)*

No arquivo `environment.server.ts`, altere o campo `apiUrl` para o IP que será resolvido pela VM Front-End, por exemplo:

````ts
export const environment = {
  production: true,
  apiUrl: 'http://backend.llw:8080' // IP da VM Back-End
};
````
> Finalize a build do Projeto e compacte os arquivos em zip, pois é uma extensão que Linux consegue lidar nativamente.

Agora no host envie o arquivo zip via ssh para a VM no caminho que foi definido no arquivo de configuração do nginx:

````bash
scp site.zip root@192.168.1.105:/opt/frontend/
````

> Tenha certeza de que o caminho que está tentando ser mandando o arquivo, realmente exista, caso o contrário falhará.

Na VM descompacte o arquivo usando o comando:

````bash
unzip site.zip
````

> Agora já deve ser possivel visualizar a tela inicial do projeto ao acessar o endereço de ip da VM no navegador.

---

### 💾 6. Preparando o ambiente para o backup

Primeiramente vamos criar o arquivo onde ficará armazenado os backups:

````bash
mkdir /opt/backup/
````

Tambem vamos criar as pastas para sub-categorizar os backups:

````bash
mkdir /opt/backup/frontend/
mkdir /opt/backup/backend/
mkdir /opt/backup/database/
````

Após isso podemos criar o usuario backup_sys:

````bash
adduser -h /opt/backup/ backup_sys
````

> Definimos a home do user backup_sys como sendo /opt/backup/ porque é só la onde ele vai operar, recebendo os backups

### 🔑 7. Recebendo as chaves RSA das outras VM's:

Na VM **Back-End** e **Database** crie as chaves RSA (**caso não possuam**):

````bash
ssh-keygen -t rsa -b 4096
````

Agora em **uma** das VM's (Back-End **ou** Database) enviamos a chave com scp:

````bash
scp /root/.ssh/id_rsa.pub backup_sys@frontend.llw:/opt/backup/.ssh/authorized_keys
````

> Será necessario permitir conexões ssh por senha de novo (na VM Front) para permitir o envio das chaves, mas temporariamente, desative depois.

Agora na **outra** VM, envie o arquivo tambem, mas dessa vez **não** use scp, e também envie ela como uma **chave temporária**:

````bash
scp /root/.ssh\id_rsa.pub backup_sys@frontend.llw:/opt/backup/.ssh/tempkey.pub
````

Na VM **Front-End**, utilize o codigo para adicionar a chave temporaria ao final da outra:

````bash
cat /opt/backup/.ssh/tempkey.pub >> /opt/backup/.ssh/authorized_keys
````

Depois remova a chave temporaria:

````bash
rm /opt/backup/.ssh/tempkey.pub
````

> Isso é necessario pois o comando scp, sobrescreve qualquer arquivo que já exista com o mesmo nome, então se mandassemos uma chave com o nome de authorized_keys, e depois mandassemos a outra da mesma forma, a segunda iria sobrescrever a primeira, dessa forma você enfileira uma atras da outra dentro do mesmo arquivo sem perder nenhuma.

Para finalizar modifique as permissões da chave:

````bash
chmod 600 /opt/backup/.ssh/authorized_keys
````

E tambem mude o dono da pasta .ssh na home do user backup:

````bash
chown -R backup_sys:backup_sys /opt/backup/.ssh
````

> Isso é necessario porque, por mais que a pasta esteja na home do usuario backup, o root ainda é dono dela, assim essa chave só será valida, quando conexões a essa maquina forem pelo usuario backup_sys.

---

### 📜 8. Configurarando o script de backup local

Crie o arquivo de script em algum lugar seguro

````bash
vi /root/backup_front
````

> Criamos na pasta root, não há nada de sensivel no backup do front, mas no dump do banco é necessario ter a senha e user mysql, oque é importante proteger.

Esse será nosso script:

````bash
#!/bin/sh

#Timestamp para identificar horario dos backups
timestamp=$(date '+%Y-%m-%d_%H-%M-%S')

#Arquivo temporario, para organizar tudo
temp_dir="/tmp/backup_front_$timestamp"

#Arquivo para os itens do front ficarem dentro
front_dir="$temp_dir/front"

#Localização dos arquivos alvo do backup
front_end="/opt/frontend"
authorized_keys="/root/.ssh/authorized_keys"

#Local e nome do arquivo compactado tar gz
tar_file="/opt/backup/frontend/backup_front_$timestamp.tar.gz"

echo "Criando diretórios temporários, Data: $timestamp"
mkdir -p "$front_dir" || { echo "Erro ao criar diretório temporário, Data: $timestamp"; exit 1; }

echo "Copiando arquivos do frontend, Data: $timestamp"
cp -r "$front_end/"* "$front_dir/" || { echo "Erro ao copiar arquivos do frontend, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "Copiando authorized_keys, Data: $timestamp"
cp "$authorized_keys" "$temp_dir/" || { echo "Erro ao copiar authorized_keys, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "Compactando tudo, Data: $timestamp"
tar -czf "$tar_file" -C "$(dirname "$temp_dir")" "$(basename "$temp_dir")" || { echo "Erro ao compactar, Data: $timestamp"; rm -rf "$temp_dir"; exit 1; }

echo "-> Backup criado com sucesso, Data: $timestamp "

rm -rf "$temp_dir"
exit 0
````

Torne-o executavel com o comando:

````bash
chmod +x /root/backup_front
````

> Agora já será possivel chamar o script manualmente, **/root/backup_front**.

---

## 🕝 9. Agendamento de Script Backup com Crontab

Edite o arquivo de agendamento padrão do Linux Alpine com o codigo:

````bash
crontab -e
````

Dentro do arquivo adicione essa linha ao final:

````bash
0 */3 * * * /root/backup_front 1>> /var/log/backup_front.log 2>> /var/log/backup_front_error.log
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
>Essa é crucial. Ela permite respostas a conexões que já foram estabelecidas, se não a VM só enviaria as requisições mas barraria a resposta, Para a vm Front-End é importante para ela se conectar e receber a resposta da API.

````bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
````
>Essa vai liberar a porta 22 utilizada em nossas VM para conexão SSH, por onde ela recebe os backups por exemplo.

---