# Task: Capítulo 1 — Setup Híbrido (VM + EC2) & SSH

## O que foi feito

- Criação da VM local no VirtualBox do Windows com Ubuntu Server LTS.
- Configuração do OpenSSH Server durante a instalação do Ubuntu na VM.
- Identificação do IP da VM e teste de conexão por SSH.
- Criação da chave SSH no notebook (WSL) com `ssh-keygen -t rsa -b 4096 -C "giropops@notebook"`.
- Configuração de `~/.ssh/config` para criar o alias `vm`.
- Uso de `ssh-copy-id vm` para copiar a chave pública para a VM.
- Teste bem-sucedido de login sem senha usando apenas a chave SSH.

## Detalhes Técnicos

### Criação da VM no VirtualBox

#### Passo 1: Criar nova máquina virtual

1. Abra o VirtualBox no Windows.
2. Clique em "Novo" ou "New".
3. Preencha os campos:
   - **Nome**: `giropops-vm`
   - **Tipo de SO**: Linux
   - **Versão**: Ubuntu (64-bit)
4. Clique em "Próximo" (Next).

#### Passo 2: Configurar memória

- **Memória RAM**: 2048 MB (mínimo) ou 4096 MB (recomendado)
- Clique em "Próximo".

#### Passo 3: Criar disco virtual

- Selecione "Criar um disco rígido agora" (Create a virtual hard disk now).
- **Tipo de disco**: VDI (VirtualBox Disk Image)
- **Armazenamento**: Dinamicamente alocado (Dynamically allocated)
- **Tamanho**: 25 GB ou mais
- Clique em "Criar".

#### Passo 4: Configurar rede (Bridge Adapter)

Após criar a VM, ela aparecerá na lista. Clique nela e depois em "Configurações" (Settings):

- Abra a aba "Rede" (Network).
- Na "Adaptador 1":
  - **Conectado a**: Adaptador com bridge (Bridged Adapter)
  - **Nome**: escolha sua interface de rede física (ex: Ethernet, Wi-Fi)
- Clique em "OK".

**Por que Bridge?** Assim a VM recebe um IP na mesma rede que seu Windows, permitindo acesso direto do WSL sem complexidade de NAT.

#### Passo 5: Inserir ISO de instalação

Na aba "Armazenamento" (Storage):

- Clique em "Controlador: IDE" e depois no ícone de disco vazio.
- Procure o arquivo ISO do Ubuntu Server (ex: `ubuntu-24.04-live-server-amd64.iso`).
- Selecione e clique em "OK".

#### Passo 6: Iniciar a VM e instalar Ubuntu

- Clique em "Iniciar" (Start).
- O instalador do Ubuntu vai aparecer.
- **Importante**: durante a instalação, quando perguntado, marque "Install OpenSSH server".
- Complete a instalação normalmente:
  - Selecione idioma
  - Layout de teclado
  - Nome da máquina: `giropops-vm`
  - Username: escolha um (ex: `giropops`)
  - Password: escolha uma
  - Disk: deixe padrão (vai usar todo o disco virtual)
- Reinicie quando solicitado.

### Descobrir o IP da VM

Após o Ubuntu iniciar, faça login na VM e rode:

```bash
ip a
```

ou

```bash
hostname -I
```

Procure por uma linha que comece com "inet" (não "inet6"). O formato é:

```
inet 192.168.1.45/24 brd 192.168.1.255 scope global dynamic eth0
```

Anote o IP: **192.168.1.45** (será diferente no seu caso).

---

### Configuração de chave SSH no WSL

#### Gerar a chave RSA

No terminal WSL (Linux):

```bash
ssh-keygen -t rsa -b 4096 -C "giropops@notebook"
```

Flags explicadas:
- `-t rsa`: tipo RSA (2048 bits é padrão, mas 4096 é mais seguro)
- `-b 4096`: 4096 bits de tamanho
- `-C "giropops@notebook"`: comentário para identificação

Quando perguntar:
- **Enter file in which to save the key**: deixe em branco e pressione Enter (usa `~/.ssh/id_rsa`)
- **Enter passphrase**: deixe em branco e pressione Enter (sem frase-senha, mais simples)

Resultado:

```
Your public key has been saved in /home/cinty/.ssh/id_rsa.pub
Your private key has been saved in /home/cinty/.ssh/id_rsa
```

Verifique:

```bash
ls -lh ~/.ssh/id_rsa*
```

Deve mostrar:

```
-rw------- 1 cinty cinty 3.2K Jun 26 10:00 /home/cinty/.ssh/id_rsa
-rw-r--r-- 1 cinty cinty 747 Jun 26 10:00 /home/cinty/.ssh/id_rsa.pub
```

---

### Configurar `~/.ssh/config`

Crie ou edite o arquivo `~/.ssh/config` no WSL:

```bash
nano ~/.ssh/config
```

Adicione:

```text
Host vm
  HostName 192.168.1.45
  User giropops
  IdentityFile ~/.ssh/id_rsa
  StrictHostKeyChecking accept-new
  UserKnownHostsFile ~/.ssh/known_hosts
```

Explicação de cada linha:
- `Host vm`: nome do alias (você usará `ssh vm`)
- `HostName 192.168.1.45`: IP real da VM descoberto com `ip a`
- `User giropops`: usuário criado durante a instalação do Ubuntu
- `IdentityFile ~/.ssh/id_rsa`: arquivo da chave privada a usar
- `StrictHostKeyChecking accept-new`: aceita a chave do servidor na primeira conexão
- `UserKnownHostsFile ~/.ssh/known_hosts`: arquivo de hosts conhecidos

Salve:
- Pressione `Ctrl+O`, depois Enter, depois `Ctrl+X`.

Teste a conexão:

```bash
ssh vm
```

Se conectar, ótimo. Se não, veja a seção "Troubleshooting" abaixo.

---

### Copiar chave pública para a VM

Use `ssh-copy-id` (recomendado):

```bash
ssh-copy-id vm
```

Ele vai:
1. Solicitar sua senha de login da VM (a que você criou na instalação).
2. Copiar `~/.ssh/id_rsa.pub` para `~/.ssh/authorized_keys` na VM.
3. Ajustar permissões automaticamente.

Alternativa manual (se `ssh-copy-id` não existir):

```bash
cat ~/.ssh/id_rsa.pub | ssh vm 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Verifique na VM:

```bash
ssh vm
cat ~/.ssh/authorized_keys
```

Deve mostrar sua chave pública (uma linha longa começando com "ssh-rsa").

---

### Testar login sem senha

Teste que está funcionando **apenas com a chave**:

```bash
ssh -o PasswordAuthentication=no vm
```

Flags explicadas:
- `-o PasswordAuthentication=no`: desabilita autenticação por senha nesta conexão
- `vm`: usa o alias definido em `~/.ssh/config`

Se conectar sem pedir senha, perfeito.

Se pedir senha ou falhar, verifique:

Na VM:

```bash
# Verificar permissões
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys

# Verificar conteúdo
cat ~/.ssh/authorized_keys | head -1
```

Devem estar assim:

```
drwx------ (700)
-rw------- (600)
```

Corrigir se necessário:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

### Comandos úteis para SSH

#### Conectar usando o alias

```bash
ssh vm
```

Usa automaticamente as configurações de `~/.ssh/config`.

#### Conectar e rodar comando remoto

```bash
ssh vm "sudo apt update"
```

Executa `apt update` na VM sem abrir um shell interativo.

#### Executar múltiplos comandos

```bash
ssh vm "sudo apt update && sudo apt upgrade -y"
```

#### Copiar arquivo para a VM

```bash
scp arquivo.txt vm:/home/giropops/
```

#### Copiar arquivo da VM para o local

```bash
scp vm:/home/giropops/arquivo.txt ./
```

#### Debug de conexão

Se houver problema, use `-v` para ver detalhes:

```bash
ssh -v vm
```

ou com mais detalhes:

```bash
ssh -vv vm
```

Verá logs como:

```
Offering public key: /home/cinty/.ssh/id_rsa RSA SHA256:abc123...
Authentications that can continue: publickey
```

Isso confirma que está tentando usar a chave correta.

---

### Forçar SSH apenas por chave (opcional)

Se quiser que a VM **nunca** aceite login por senha (apenas chave):

Na VM, edite `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Procure (ou adicione) estas linhas:

```text
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM no
```

Salve e reinicie o SSH:

```bash
sudo systemctl restart ssh
```

Agora, `ssh vm` **obrigatoriamente** usará a chave.

---

### Troubleshooting

#### "Connection refused" ou "No route to host"

- Verifique o IP correto: `ssh vm "ip a"` não funciona, então na VM rode `ip a`.
- Verifique que a VM está em Bridge Adapter (Configurações → Rede).
- Verifique firewall do Windows.

#### "Permission denied (publickey)"

- Rode `ssh-copy-id vm` novamente.
- Verifique permissões em `~/.ssh/authorized_keys` na VM.
- Use `ssh -v vm` para debugar.

#### "Host key verification failed"

- Adicione manualmente: `ssh-keyscan vm >> ~/.ssh/known_hosts`
- Ou configure `StrictHostKeyChecking accept-new` em `~/.ssh/config` (já feito acima).

#### SSH trava ou demora muito

- Pode ser DNS. Teste com IP direto:
```bash
ssh -o StrictHostKeyChecking=no giropops@192.168.1.45
```

- Ou edite `/etc/ssh/sshd_config` na VM e adicione:
```text
UseDNS no
```

Depois `sudo systemctl restart ssh`.

## EC2 na AWS Console

A lógica é parecida com a VM, mas a autenticação na EC2 costuma ser feita com uma chave `.pem` criada na AWS.

### 1. Criar a instância EC2

1. Acesse a AWS Console e entre no serviço EC2.
2. Clique em "Launch instance".
3. Preencha:
   - **Name**: `giropops-ec2`
   - **AMI**: Ubuntu Server 24.04 LTS (ou a versão LTS disponível)
   - **Instance type**: `t2.micro`
4. Em **Key pair (login)**:
   - Clique em "Create new key pair"
   - Escolha o nome `giropops-ec2`
   - Formato `.pem`
   - Baixe a chave e guarde em um local seguro, por exemplo `~/Downloads/giropops-ec2.pem`
5. Em **Network settings**:
   - Marque "Allow SSH traffic from"
   - Selecione "My IP" (mais seguro) ou `0.0.0.0/0` se estiver testando em laboratório
6. Clique em "Launch instance".

### 2. Obter o IP público da EC2

Depois que a instância ficar em estado `running`:

- Abra o painel da instância.
- Copie o valor de **Public IPv4 address**.

Esse IP será usado no alias SSH.

### 3. Conectar na EC2 pelo terminal

No WSL ou terminal Linux local:

```bash
chmod 400 ~/Downloads/giropops-ec2.pem
ssh -i ~/Downloads/giropops-ec2.pem ubuntu@<IP_PUBLICO_DA_EC2>
```

> No AMI Ubuntu da AWS, o usuário padrão costuma ser `ubuntu`.

Se a conexão funcionar, você já concluiu a parte inicial da EC2.

### 4. Configurar o alias `ec2` no `~/.ssh/config`

Edite o arquivo:

```bash
nano ~/.ssh/config
```

Adicione:

```text
Host ec2
  HostName <IP_PUBLICO_DA_EC2>
  User ubuntu
  IdentityFile ~/.ssh/giropops-ec2.pem
  StrictHostKeyChecking accept-new
  UserKnownHostsFile ~/.ssh/known_hosts
```

Salve e teste:

```bash
ssh ec2
```

### 5. Atualizar pacotes na EC2

Dentro da EC2, rode:

```bash
sudo apt update
```

### 6. Comparando com a VM

Você agora tem dois ambientes prontos para o laboratório:

- `ssh vm` → VM local no VirtualBox
- `ssh ec2` → instância EC2 na AWS

Em dois terminais diferentes, teste:

```bash
ssh vm
ssh ec2
```

E em cada um deles:

```bash
sudo apt update
```

### 7. (Opcional) Usar `ssh-copy-id` na EC2

Se quiser, também é possível colocar a sua chave pública na EC2 para logar sem usar o `.pem`.

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ec2
```

Mas para o fluxo inicial da AWS Console, usar o `.pem` é o caminho mais simples e recomendado.

### 8. Testar `rsync` entre a VM e a EC2

O `rsync` é ótimo para copiar arquivos entre os ambientes usando SSH.

#### 8.1. Instalar o `rsync` nas duas máquinas

Na VM e na EC2:

```bash
sudo apt update
sudo apt install -y rsync
```

#### 8.2. Criar um arquivo de teste

No seu notebook ou terminal local:

```bash
echo "teste rsync" > /tmp/arquivo-rsync.txt
```

#### 8.3. Enviar o arquivo para a VM

```bash
rsync -av --progress /tmp/arquivo-rsync.txt vm:/tmp/
```

Verifique na VM:

```bash
ssh vm "ls -l /tmp/arquivo-rsync.txt"
```

#### 8.4. Enviar o mesmo arquivo para a EC2

```bash
rsync -av --progress /tmp/arquivo-rsync.txt ec2:/tmp/
```

Verifique na EC2:

```bash
ssh ec2 "ls -l /tmp/arquivo-rsync.txt"
```

#### 8.5. (Opcional) Trazer de volta para o notebook

```bash
rsync -av --progress ec2:/tmp/arquivo-rsync.txt /tmp/arquivo-rsync-retornado.txt
```

Se os comandos acima funcionarem, o `rsync` já está validado como bônus do capítulo.

## O que falta continuar

- Provisionar a instância EC2 `t2.micro` na AWS Free Tier.
- Configurar um alias `ec2` em `~/.ssh/config` com o `HostName` público da EC2.
- Copiar ou usar a chave correta para logar na EC2 (`ssh-copy-id ec2` ou `IdentityFile` para o `.pem`).
- Conectar em dois terminais ao mesmo tempo: um para `ssh vm` e outro para `ssh ec2`.
- Rodar `sudo apt update` em ambos os terminais.
- (Bônus) Testar `rsync` entre a máquina local e os servidores, ou entre VM e EC2.

## Observações

- O arquivo `docs/fase-1/cap-01.md` descreve o objetivo do capítulo e o checklist.
- O próximo passo é completar a parte AWS e a configuração do SSH para a instância EC2.
