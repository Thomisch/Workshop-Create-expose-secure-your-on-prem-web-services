# Workshop: Create, expose & secure your on-prem web services

## Table des matières
1. [Prérequis](#prérequis)
2. [Création du conteneur LXC](#création-du-conteneur-lxc)
3. [Installation et configuration de Docker](#installation-et-configuration-de-docker)
4. [Déploiement de l'application](#déploiement-de-lapplication)
5. [Sécurisation SSH](#sécurisation-ssh)
6. [Configuration des outils de sécurité](#configuration-des-outils-de-sécurité)
7. [Configuration du routeur et reverse proxy](#configuration-du-routeur-et-reverse-proxy)
8. [Configuration DNS et domaine](#configuration-dns-et-domaine)
9. [Tests et validation](#tests-et-validation)

## Prérequis

- Serveur Proxmox opérationnel
- Routeur avec accès administrateur
- Nom de domaine (ex: chez Cloudflare)
- Accès internet avec IP publique

## 1. Création du conteneur LXC

### 1.1 Création du conteneur sur Proxmox

```bash
# Créer le conteneur LXC
pct create 100 /var/lib/vz/template/cache/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname webapp-container \
  --memory 2048 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-lvm \
  --rootfs local-lvm:8

# Démarrer le conteneur
pct start 100

# Se connecter au conteneur
pct enter 100
```

### 1.2 Configuration initiale du système

```bash
# Mise à jour du système
apt update && apt upgrade -y

# Installation des outils de base
apt install -y curl wget git nano htop net-tools
```

## 2. Installation et configuration de Docker

### 2.1 Installation de Docker

```bash
# Installation des dépendances
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Ajout de la clé GPG Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajout du repository Docker
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation de Docker
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Démarrage et activation de Docker
systemctl start docker
systemctl enable docker
```

### 2.2 Configuration des permissions

```bash
# Création d'un utilisateur pour le déploiement
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
usermod -aG sudo deploy

# Configuration du mot de passe
passwd deploy
```

## 3. Déploiement de l'application

### 3.1 Exemple d'application web simple

```bash
# Création du répertoire de travail
mkdir -p /home/deploy/webapp
cd /home/deploy/webapp

# Création du Dockerfile
cat > Dockerfile << EOF
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
EOF

# Création d'une page web de test
cat > index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Mon App Web</title>
</head>
<body>
    <h1>Bienvenue sur mon service web sécurisé !</h1>
    <p>Déployé avec Docker dans un conteneur LXC</p>
</body>
</html>
EOF

# Construction et lancement du conteneur
docker build -t mon-webapp .
docker run -d --name webapp -p 8080:80 mon-webapp
```

### 3.2 Docker Compose pour une gestion simplifiée

```yaml
# filepath: /home/deploy/webapp/docker-compose.yml
version: '3.8'

services:
  webapp:
    build: .
    container_name: webapp
    ports:
      - "8080:80"
    restart: unless-stopped
    volumes:
      - ./logs:/var/log/nginx
```

```bash
# Lancement avec Docker Compose
docker compose up -d
```

## 4. Sécurisation SSH

### 4.1 Configuration SSH sécurisée

```bash
# Sauvegarde de la configuration originale
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Configuration SSH sécurisée
cat > /etc/ssh/sshd_config << EOF
Port 2222
Protocol 2
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
AllowUsers deploy
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

# Redémarrage du service SSH
systemctl restart ssh
```

### 4.2 Génération et configuration des clés SSH

```bash
# Sur votre machine locale
ssh-keygen -t ed25519 -C "deploy@webapp"

# Copie de la clé publique sur le serveur
ssh-copy-id -p 2222 deploy@IP_DU_CONTENEUR
```

## 5. Configuration des outils de sécurité

### 5.1 Installation et configuration de Fail2Ban

```bash
# Installation de Fail2Ban
apt install -y fail2ban

# Configuration pour SSH
cat > /etc/fail2ban/jail.local << EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
EOF

# Démarrage de Fail2Ban
systemctl start fail2ban
systemctl enable fail2ban
```

### 5.2 Installation de Let's Encrypt (Certbot)

```bash
# Installation de Certbot
apt install -y certbot

# Note: Les certificats seront générés plus tard avec Nginx
```

### 5.3 Configuration du firewall UFW

```bash
# Installation et configuration d'UFW
apt install -y ufw

# Configuration des règles
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp  # SSH
ufw allow 8080/tcp  # Application web
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS

# Activation du firewall
ufw enable
```

## 6. Configuration du routeur et reverse proxy

### 6.1 Installation de Nginx comme reverse proxy

```bash
# Installation de Nginx
apt install -y nginx

# Configuration du reverse proxy
cat > /etc/nginx/sites-available/webapp << EOF
server {
    listen 80;
    server_name votre-domaine.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

# Activation du site
ln -s /etc/nginx/sites-available/webapp /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### 6.2 Configuration du routeur

```
1. Accéder à l'interface d'administration du routeur
2. Configurer le port forwarding :
   - Port 80 (HTTP) → IP_CONTENEUR:80
   - Port 443 (HTTPS) → IP_CONTENEUR:443
   - Port 2222 (SSH) → IP_CONTENEUR:2222
3. Sauvegarder la configuration
```

## 7. Configuration DNS et domaine

### 7.1 Configuration Cloudflare

```
1. Se connecter à Cloudflare
2. Ajouter un enregistrement A :
   - Type: A
   - Name: webapp (ou votre sous-domaine)
   - Value: VOTRE_IP_PUBLIQUE
   - TTL: Auto
3. Activer le proxy Cloudflare (optionnel pour DDoS protection)
```

### 7.2 Configuration SSL avec Let's Encrypt

```bash
# Génération du certificat SSL
certbot --nginx -d webapp.votre-domaine.com

# Vérification du renouvellement automatique
certbot renew --dry-run

# Configuration du renouvellement automatique
echo "0 12 * * * /usr/bin/certbot renew --quiet" | crontab -
```

### 7.3 Configuration Nginx avec SSL

```nginx
# filepath: /etc/nginx/sites-available/webapp
server {
    listen 80;
    server_name webapp.votre-domaine.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name webapp.votre-domaine.com;

    ssl_certificate /etc/letsencrypt/live/webapp.votre-domaine.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/webapp.votre-domaine.com/privkey.pem;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 8. Tests et validation

### 8.1 Tests de connectivité

```bash
# Test de l'application en local
curl http://localhost:8080

# Test de Nginx
curl http://localhost

# Test du SSL
curl https://webapp.votre-domaine.com

# Vérification des certificats
openssl s_client -connect webapp.votre-domaine.com:443 -servername webapp.votre-domaine.com
```

### 8.2 Tests de sécurité

```bash
# Vérification du statut de Fail2Ban
fail2ban-client status sshd

# Test des règles firewall
ufw status verbose

# Vérification des logs
tail -f /var/log/nginx/access.log
tail -f /var/log/fail2ban.log
```

## 9. Automatisation du déploiement (Bonus)

### 9.1 Script de déploiement

```bash
# filepath: /home/deploy/deploy.sh
#!/bin/bash

cd /home/deploy/webapp

# Pull du code depuis Git
git pull origin main

# Reconstruction et redémarrage du conteneur
docker compose down
docker compose build --no-cache
docker compose up -d

# Vérification du statut
docker compose ps
```

### 9.2 Configuration GitHub Actions (exemple)

```yaml
# filepath: .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to server
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.HOST }}
        username: deploy
        key: ${{ secrets.SSH_KEY }}
        port: 2222
        script: |
          cd /home/deploy/webapp
          git pull origin main
          docker compose down
          docker compose up -d --build
```

## Récapitulatif

Félicitations ! Vous avez maintenant :

✅ Un conteneur LXC avec Docker installé
✅ Une application web déployée
✅ Un accès SSH sécurisé
✅ Des outils de sécurité configurés (Fail2Ban, UFW, SSL)
✅ Un reverse proxy Nginx
✅ Un nom de domaine pointant vers votre service
✅ Un certificat SSL valide

Votre service web est maintenant accessible de manière sécurisée depuis Internet !

## Ressources supplémentaires

- [Documentation Docker](https://docs.docker.com/)
- [Guide Nginx](https://nginx.org/en/docs/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Fail2Ban](https://www.fail2ban.org/)
- [Cloudflare DNS](https://developers.cloudflare.com/dns/)