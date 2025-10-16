# 🧠 Projeto Operating Systems — FIAP 3ESPX

**Integrantes:**
- Augusto Milreu – RM98245  
- David Denunci – RM98603  
- Fernando Popolili – RM99919  
- Lucas de Toledo – RM97913  
- Matheus Zanardi – RM98832  

---

## 🧩 Visão Geral
Este projeto tem como objetivo a configuração de um ambiente Linux com **Ubuntu Server**, instalação e operação de **Docker**, **NGINX** e **Node.js**, além da implementação de **banco de dados SQLite**, **API Node** e **scripts de anonimização (PII)** com logs de operação.

O ambiente foi criado e configurado para simular uma aplicação real com:
- API de usuários (`/users`);
- Script de anonimização de dados pessoais (PII);
- Registro de logs de acesso e execução em `/var/log`;
- Reverse Proxy NGINX;
- Controle de permissões e usuários dedicados.

---

## ⚙️ Etapa 1 — Ambiente e Aplicação

### 1. Criação da VM Ubuntu Server
**Configurações da VM (VirtualBox):**
- Ubuntu Server ISO 22.04 LTS  
- 2 vCPUs | 4 GB RAM | 25 GB Disco (dinâmico)  
- Rede: **Bridged Adapter**

**Verificar rede:**
```bash
ip a
ping -c 3 8.8.8.8
```

### Acesso via SSH (opcional) ###

**No Ubuntu Server:**
```bash
sudo apt install -y openssh-server # Instalação do SSH server
sudo systemctl enable ssh # Instalação do SSH server
sudo systemctl status ssh # Habilitar para iniciar automaticamente após todo reboot
hostname -I # Verfica o IP da VM para ser conectado através do SSH
```

**No Windows PowerShell:**
```bash
ssh usuario@IP_DA_VM # Utilizar apenas o IPV4, resultado do hostname -I
```

### 2. Instalação do Docker e NGINX ###

**Atualização e pacotes iniciais:**
```bash
sudo apt update # Atualizações e instalações de pacotes iniciais
sudo apt install -y ca-certificates curl gnupg lsb-release # instalação de pacotes necessários no Ubuntu
```

**Instalação do Docker:**
```bash
# Adicionar chave e repositório Docker ao OS
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin # instalação do Docker e componentes relacionados no Ubuntu
sudo usermod -aG docker $USER # Adicionar usuário ao grupo docker

# habilita e iniciar docker
sudo systemctl enable Docker
sudo systemctl start docker

docker --version # Verificar instalação
```

**Instalação do NGINX:**
```bash
sudo apt install -y nginx # inicia a instalação do NGINX

# habilita e iniciar NGINX
sudo systemctl enable nginx --now
sudo systemctl status nginx

# Verificar status
systemctl status nginx
```

**Testar acesso**
No navegador -> http://IP.DA.VM (o mesmo ip utilizado para fazer a conexão ssh com a VM) ou no seu terminal:
```bash
curl http://IP.DA.VM
```

Com a instalação correta, é retornado uma mensagem web dizendo que o servidor web NGINX foi instalado corretamente e esta funcionando.