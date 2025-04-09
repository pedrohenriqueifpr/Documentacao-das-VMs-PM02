# 📘 Documentação: Instalação e Configuração do Alpine Linux no VirtualBox (VM Base)

> Este arquivo documenta passo a passo cada configuração feita desde o inicio para configurar a **VM Base**, ela será utilizada e clonada como maquina base para não ter que passar por todas essas configurações para cada vm que criarmos, seja front, back e bd, mas isso **não** significa que as configurações feitas aqui irão perdurar até o final, um exemplo é a conexão ssh com root, que será usada apenas para facilitar a configuração da VM, mas será removida depois como medida de segurança e requisito do projeto.

---

## 📥 1. Download dos Recursos Necessários

Começaremos baixando um **hypervisor**, um software que permite executar várias máquinas virtuais em uma única máquina física.  
No nosso caso, utilizaremos o **Oracle VirtualBox**.

Também baixaremos a **ISO do Alpine Linux**, em sua versão virtual, que dispensa interface gráfica, contando apenas com o Bash, uma interface de linha de comando (CLI) usada para interpretar comandos.

---

## 🛠️ 2. Instalação e Criação da Máquina Virtual

Após instalar o VirtualBox e baixar a ISO do Alpine, passaremos para a criação da **máquina virtual (VM)**.

### ⚙️ Configurações da VM:

- **CPU**: 1 core  
- **RAM**: 512 MB  
- **Disco**: 20 GB  
- **ISO de Boot**: Alpine Linux (modo virtual)

---

## ⚙️ 3. Ajustes Iniciais da VM

### 🧷 Tecla do Hospedeiro

Por padrão:

- **Ctrl esquerdo**: comandos dentro da VM  
- **Ctrl direito**: comandos para o host

Se você não tiver a tecla **Ctrl direita**, altere a "Tecla do Hospedeiro" em:  
**Preferências > Input > Máquina Virtual**

### 🌐 Configuração de Rede

Adicione uma nova **interface de rede em modo Bridge**, permitindo que a VM se conecte à mesma rede do host como se fosse um dispositivo físico separado.

---

## 🚀 4. Boot e Setup do Alpine Linux

Dê boot na VM, entre com o usuário:

- **Login**: `root`  
- **Senha**: *(vazio)*

Inicie o processo de setup com:

```bash
setup-alpine
```

---

## 🧾 5. Configurações do Setup do Alpine Linux

- **Layout de Teclado**: `br-br`  
- **Hostname**: `localhost`  
- **Interfaces de Rede**: `eth0 (NAT)`, `eth1 (Bridge)`  
- **Endereço IP**: `DHCP`  
- **Configuração Manual da Interface**: `n`  
- **Senha do usuário root**: `root`  
- **Fuso Horário**: `America/Sao_Paulo`  
- **Proxy**: `none`  
- **NTP**: `chrony`  
- **Repositório (Mirror)**: `1`  
- **Criar um usuário comum**: `n`  
- **Servidor SSH**: `openssh`  
- **Permitir login root via SSH**: `prohibit-password`  
- **Chave SSH para root**: `none`  
- **Disco selecionado**: `sda`  
- **Modo de uso do disco**: `sys`  
- **Apagar dados do disco**: `y`

> Essas foram as configurações para configurar a VM base em um ambiente doméstico, mas todas essas configurações podem ser mudadas mesmo após completar o setup, e serão modificadas ao longo das configurações das outras 3 máquinas: **front**, **back** e **bd**.

---

## 🧱 6. Finalizando a Instalação

Após concluir o processo de instalação (setup), desligue a máquina com o comando:

```bash
poweroff
```

> Remova o dispositivo que contem ISO do Alpine, pois o sistema já foi instalado no disco rígido.

---

## 📦 7. Gerenciamento de Pacotes

### 7.1 Habilitando Repositórios da Comunidade

Edite o arquivo de repositórios:

```bash
vi /etc/apk/repositories
```

> O vi é um editor de texto padrão dos sistemas operacionais Linux, que permite editar arquivos.

Procure pela seguinte linha:

```bash
#http://dl-cdn.alpinelinux.org/alpine/v3.19/community
```

> (Remova o `#` (Descomente) para habilitar o repositório `community`, ele contém pacotes contribuídos pela comunidade, que podem não ser tão estáveis quanto os do repositório `main`, mas possui pacotes que iremos utilzar em nossa VM.

### 7.2 Atualizando o Sistema

Execute os seguintes comandos para atualizar todos os pacotes:

```bash
apk update
apk upgrade
```

### 7.3 Instalando um Editor de Texto Alternativo (Opcional)

- Para instalar o **Nano** (mais simples e intuitivo):

```bash
apk add nano
```

- Para instalar o **Vim** (uma versão aprimorada do Vi):

```bash
apk add vim
```

---

## 🧩 8. Integração com o Host (Apenas para o VirtualBox)

### 8.1 Instalando o VirtualBox Guest Additions

Melhora a integração com o host (como redimensionamento de tela, clipboard compartilhado etc):

```bash
apk add virtualbox-guest-additions
```

Inicie o serviço e adicione-o à inicialização automática:

```bash
rc-service virtualbox-guest-additions start
rc-update add virtualbox-guest-additions
```

---

## 🔐 9. Configurando Acesso SSH

### 9.1 Permitindo Acesso Temporário por Senha

Edite o arquivo de configuração do SSH:

```bash
vi /etc/ssh/sshd_config
```

Descomente e altere as seguintes linhas:

```
PermitRootLogin yes
PasswordAuthentication yes
```

Reinicie o serviço:

```bash
rc-service sshd restart
```

---

## 🔑 10. Acesso via Chave SSH

### 10.1 Gerando Chave no Host (caso ainda não tenha)

No host (Windows, por exemplo):

```bash
ssh-keygen -t rsa -b 4096
```

>Ele ira pedir uma senha para as chaves, preencha-o com uma senha segura (tambem pode ser deixada vazia).

---

### 10.2 Preparando a VM para Receber a Chave

Na VM, crie o diretório `.ssh` se ainda não existir:

```bash
mkdir /root/.ssh
```

No host, copie a chave pública para a VM:
```bash
scp .ssh/id_rsa.pub root@192.168.1.108:/root/.ssh/authorized_keys
```

### ⚠️ Atenção: Esse comando irá sobrescrever qualquer chave que já estiver no arquivo /root/.ssh/authorized_keys. Se você deseja preservar as chaves já existentes, utilize uma abordagem alternativa:

No host, envie a sua chave como uma chave temporária:

````bash
scp .ssh\id_rsa.pub root@192.168.1.108:/root/.ssh/tempkey.pub
````

Na VM, utilize o codigo abaixo para adicionar a chave temporaria ao final das outras:

````bash
cat /root/.ssh/tempkey.pub >> /root/.ssh/authorized_keys
````

Depois remova a chave temporaria:

````bash
rm /root/.ssh/tempkey.pub
````

---

Ajuste as permissões:

```bash
chmod 600 /root/.ssh/authorized_keys
```

Teste a conexão via SSH:

```bash
ssh root@192.168.1.108
```

---

### 10.3 Desabilitando Acesso por Senha

Edite novamente o `sshd_config`:

```
PasswordAuthentication no
```

Reinicie o serviço SSH:

```bash
rc-service sshd restart
```

> A partir deste ponto, o acesso à VM será feito **exclusivamente via chave SSH**.
