# Projeto Operating Systems — FIAP 3ESPX

Tecnologias usadas
| ![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-E95420?style=for-the-badge\&logo=ubuntu\&logoColor=white) | 
| ![NodeJS](https://img.shields.io/badge/Node.js-18.x-339933?style=for-the-badge\&logo=node.js\&logoColor=white) | 
| ![SQLite](https://img.shields.io/badge/SQLite-3-blue?style=for-the-badge\&logo=sqlite\&logoColor=white) | 
| ![NGINX](https://img.shields.io/badge/NGINX-Reverse%20Proxy-009639?style=for-the-badge\&logo=nginx\&logoColor=white) | 
| ![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge\&logo=docker\&logoColor=white) | 
| ![License](https://img.shields.io/badge/License-FIAP-lightgrey?style=for-the-badge) | 

---

## Integrantes

| Nome                  | RM      |
| --------------------- | ------- |
| **Augusto Milreu**    | RM98245 |
| **David Denunci**     | RM98603 |
| **Fernando Popolili** | RM99919 |
| **Lucas de Toledo**   | RM97913 |
| **Matheus Zanardi**   | RM98832 |

---

## Descrição do Projeto

Projeto desenvolvido para a disciplina **Operating Systems** da **FIAP**, com o objetivo de:

* Configurar um ambiente **Ubuntu Server** em VM (VirtualBox);
* Instalar e configurar **Docker**, **NGINX**, **Node.js** e **SQLite**;
* Criar uma **API RESTful** com **Node.js/Express**;
* Implementar **scripts de anonimização de PII (dados pessoais)**;
* Registrar **logs de acesso e operação** em `/var/log`;
* Criar **usuários dedicados** para execução e escrita de logs.

---

## Estrutura do Projeto

```
/home/gutofm/
├── app/
│   ├── index.js               # API principal (users + logs)
│   ├── anonymize_pii.sh       # Script de anonimização PII
│   ├── package.json
│
├── sqlite-data/
│   ├── app.db                 # Banco SQLite
│   └── backups/               # Backups automáticos do DB
│
└── /var/log/
    ├── xp_access.log          # Logs de acesso e operação
    └── xp_anonymize.log       # Logs do script de anonimização
```

---

## Etapa 1 — Configuração do Ambiente

### 1. Criação da VM

**VirtualBox (Ubuntu Server 22.04 LTS):**

* 2 vCPUs
* 4 GB RAM
* 25 GB disco dinâmico
* Rede: **Bridged Adapter**

Verificar rede:

```bash
ip a
ping -c 3 8.8.8.8
```

### 2. Acesso via SSH

```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
hostname -I
```

No PowerShell (Windows):

```bash
ssh usuario@IP_DA_VM
```

---

### 3. Instalação Docker e NGINX

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \\
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
sudo systemctl enable --now docker
docker --version

sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Verificar:

```
http://IP_DA_VM
```

---

### 4. Banco de Dados SQLite (Docker)

```bash
mkdir -p ~/sqlite-data
cd ~/sqlite-data
```

**Dockerfile**

```bash
cat > Dockerfile <<'EOF'
FROM alpine:3.18
RUN apk add --no-cache sqlite
VOLUME /data
WORKDIR /data
CMD ["sh", "-c", "tail -f /dev/null"]
EOF
```

**Build e Execução**

```bash
docker build -t sqlite-image .
docker run -d --name sqlite --restart unless-stopped -v ~/sqlite-data:/data sqlite-image
docker exec -it sqlite sh -c "sqlite3 /data/app.db \"CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, cpf TEXT, rg TEXT, email TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP);\""
```

---

### 5. API Node.js

Instalação:

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs build-essential
```

Criação da aplicação:

```bash
mkdir -p ~/app
cd ~/app
npm init -y
npm install express sqlite3 body-parser
```

Execução:

```bash
node index.js
```

Teste:

```bash
curl -X POST http://IP_DA_VM:3000/users -H "Content-Type: application/json" -d '{"name":"Joao","cpf":"123.456.789-09","rg":"12.345.678-9","email":"joao@example.com"}'
curl http://IP_DA_VM:3000/users
```

---

### 6. NGINX Reverse Proxy

```bash
sudo tee /etc/nginx/sites-available/myapp <<'EOF'
server {
  listen 80;
  server_name localhost;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
EOF

sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Verificação:

```
http://IP_DA_VM/users
```

---

## Etapa 2 — Logs, Usuários e Anonimização

### 1. Endpoint de Logs

```js
app.post('/logs', (req, res) => {
  const { message, type } = req.body;
  if (!message) return res.status(400).json({ error: "message required" });

  const logLine = `[${new Date().toISOString()}] [${type || "INFO"}] ${message}\n`;
  fs.appendFile("/var/log/xp_access.log", logLine, (err) => {
    if (err) return res.status(500).json({ error: "Erro ao gravar log" });
    res.json({ status: "ok" });
  });
});
```

**Teste:**

```bash
curl -X POST http://127.0.0.1:3000/logs \
  -H "Content-Type: application/json" \
  -d '{"message":"Usuário João inserido com sucesso","type":"INFO"}'
sudo tail -n 5 /var/log/xp_access.log
```

---

### 2. Script de Anonimização (com logs)

`/home/gutofm/app/anonymize_pii.sh`

```bash
#!/bin/bash
set -euo pipefail
DB="/home/gutofm/sqlite-data/app.db"
BACKUP_DIR="/home/gutofm/sqlite-data/backups"
LOG_FILE="/var/log/xp_anonymize.log"

mkdir -p "$BACKUP_DIR"
BACKUP="$BACKUP_DIR/backup_$(date +%F_%H%M%S).db"

echo "[INFO] Execução iniciada em $(date '+%Y-%m-%d %H:%M:%S')" >> "$LOG_FILE"

cp -f "$DB" "$BACKUP" || { echo "[ERRO] Falha no backup" >> "$LOG_FILE"; exit 1; }

/usr/bin/sqlite3 "$DB" <<'SQL'
UPDATE users SET name = CASE WHEN name IS NOT NULL AND length(name) > 0 THEN substr(name,1,1) || '***' ELSE NULL END WHERE name IS NOT NULL;
UPDATE users SET cpf = '***.***.***-**' WHERE cpf IS NOT NULL;
UPDATE users SET rg  = '*******' WHERE rg IS NOT NULL;
UPDATE users SET email = NULL WHERE email IS NOT NULL;
SQL

echo "[INFO] Execução finalizada em $(date '+%Y-%m-%d %H:%M:%S')" >> "$LOG_FILE"
```

---

### 3. Usuários dedicados

```bash
sudo adduser loguser
sudo adduser anonymuser

sudo touch /var/log/xp_access.log /var/log/xp_anonymize.log
sudo chown loguser:loguser /var/log/xp_access.log
sudo chown anonymuser:anonymuser /var/log/xp_anonymize.log
sudo chmod 664 /var/log/xp_access.log /var/log/xp_anonymize.log

sudo mkdir -p /home/gutofm/sqlite-data/backups
sudo setfacl -R -m u:anonymuser:rwx /home/gutofm/sqlite-data
sudo setfacl -m u:anonymuser:rw /home/gutofm/sqlite-data/app.db
```

---

### 4. Testes Finais

**Rodar a API como `loguser`:**

```bash
sudo -u loguser bash -lc 'node /home/gutofm/app/index.js'
```

**Enviar log:**

```bash
curl -X POST http://127.0.0.1:3000/logs \
  -H "Content-Type: application/json" \
  -d '{"message":"Usuário João inserido","type":"INFO"}'
sudo tail -n 5 /var/log/xp_access.log
```

**Executar anonimização:**

```bash
sudo -u anonymuser bash -lc '/home/gutofm/app/anonymize_pii.sh'
sudo tail -n 5 /var/log/xp_anonymize.log
```

---

## Demonstração (Vídeo Pitch)

**Duração:** até 6 minutos

### O que mostrar:

1. Estrutura das pastas (`tree ~/app ~/sqlite-data`);
2. API rodando (`node index.js`);
3. `curl` para `/logs` e visualização do log em `/var/log/xp_access.log`;
4. Execução do `anonymize_pii.sh` e log em `/var/log/xp_anonymize.log`;
5. Teste final de anonimização via `/users`.

---

## Conclusão

O projeto entrega:

* Ambiente Ubuntu Server completo e funcional;
* API Node.js com persistência SQLite;
* Reverse Proxy NGINX;
* Script de anonimização PII com backup e logs;
* Controle de permissões e usuários isolados;
* Documentação e vídeo de demonstração.

---

**Disciplina:** Operating Systems — FIAP 3ESPX (2025)
**Versão:** 2.0 — Final
