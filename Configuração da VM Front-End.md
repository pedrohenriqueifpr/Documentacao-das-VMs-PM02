# 📘 Documentação: Configuração da VM Front-End

> Este arquivo documenta, passo a passo, todo o processo de configuração da **VM Front-End**. Ela será responsável por hospedar a interface do sistema utilizando o **nginx**, e também atuará como **servidor de backups**, armazenando os arquivos de backup de todas as outras VMs — incluindo ela mesma. Por esse motivo, esta VM tambem possuirá a **chave pública RSA** de cada uma das outras máquinas virtuais em seu usuário **backup_sys**, que tambem será o usuario que em sua home terá os arquivos de backup.

---

### 📋 1. Clonando a VM Base

Começaremos clonando a **VM base**, que já foi criada previamente.  
A configuração dessa VM base está documentada no **Arquivo de configuração da VM base**.

---

### 🏷️ 2. Alterando o Hostname

Para diferenciar a VM Front-End das demais, alteraremos seu hostname editando o arquivo:

````bash
vi /etc/hostname
````

> Mudaremos o nome de **localhost** para **frontend** (alterações no hostname são aplicadas após um reboot).

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

### 📡 6. Configurando a resolução de IP's

> Com o Front-End sendo exibido, ele não conseguirá fazer requisições pois não configuramos o back ainda, mas mesmo que o back estivesse configurado ele ainda não iria conseguir fazer requisições pois não está resolvendo corretamente os ip's que colocamos na build.

Por isso agora dentro do arquivo em **/etc/hosts** adicione:

````bash
192.168.1.106 backend.llw
````

> Dessa forma, o endereço backend.llw será resolvido para o IP definido na linha, facilitando possiveis trocas de IP causadas por mudanças no ambiente de rede da VM, sem a necessidade de re-buildar o projeto a cada alteração de endereço.

Adicione a mesma linha no arquivo de **hosts** do host, da mesma forma, mas no caminho (no caso de Windows):

````plaintext
C:\Windows\System32\drivers\etc
````

> Isso é necessário porque, mesmo que o front esteja rodando na VM e resolva o IP corretamente internamente, o acesso ao front é feito pelo navegador do host — e o host não reconhece esse IP. Por isso, é preciso configurar essa resolução também no host.

---

### 💾 7. Primeiros passos para realizar o backup

>Criar user backup_sys

>Receber as chaves das outras VM's

---

### 📜 8. Configurar o script de backup local

>Script de backup local

---
