# 🚀 Hetzner Deployment Guide - Schritt für Schritt

Komplette Anleitung zum Wechsel von IONOS zu Hetzner mit automatischem CI/CD Deployment.

---

## 📋 Übersicht

Diese Anleitung führt dich durch:
1. ✅ Hetzner Server kaufen und einrichten
2. ✅ Docker & Docker Compose installieren
3. ✅ SSH-Key für CI/CD erstellen
4. ✅ Projekt auf Server vorbereiten (Backend, Admin, Portal, Frontend)
5. ✅ Nginx Reverse Proxy einrichten (alle Services)
6. ✅ SSL Zertifikat (Let's Encrypt)
7. ✅ CI/CD Secrets konfigurieren
8. ✅ Erste Deployment testen
9. ✅ Datenbank-Migration (von IONOS)
10. ✅ Monitoring & Wartung
11. ✅ Domain-Transfer von IONOS zu Hetzner (clubsmarter.de)

---

## 🛒 Schritt 1: Hetzner Server kaufen

### 1.1 Account erstellen

1. Gehe zu: https://www.hetzner.com/
2. Klicke auf **"Jetzt registrieren"** oder **"Sign up"**
3. Fülle das Registrierungsformular aus
4. Verifiziere deine E-Mail-Adresse

### 1.2 Server auswählen

1. Gehe zu: https://console.hetzner.com/
2. Klicke auf **"Add Server"** oder **"Server hinzufügen"**
3. Wähle **"Cloud"** (empfohlen) oder **"Dedicated"**

**Empfohlene Konfiguration:**
- **Location**: Nürnberg oder Falkenstein (Deutschland)
- **Image**: Ubuntu 22.04 oder Ubuntu 24.04
- **Type**: 
  - **Minimum**: CX21 (2 vCPU, 4 GB RAM, 40 GB SSD) - ~€5/Monat
  - **Empfohlen**: CX31 (2 vCPU, 8 GB RAM, 80 GB SSD) - ~€10/Monat
  - **Für größere Projekte**: CPX31 (4 vCPU, 8 GB RAM, 160 GB SSD) - ~€15/Monat
- **SSH Key**: Erstelle einen neuen SSH-Key (siehe Schritt 2)
- **Firewall**: Optional, aber empfohlen
- **Backup**: Optional (kann später aktiviert werden)

### 1.3 Server erstellen

1. Klicke auf **"Create & Buy"** oder **"Erstellen & Kaufen"**
2. Warte bis der Server bereit ist (ca. 1-2 Minuten)
3. Notiere dir:
   - **Server IP-Adresse** (z.B. `123.456.789.0`)
   - **Root-Passwort** (falls kein SSH-Key verwendet wurde)

---

## 🔐 Schritt 2: SSH-Key für CI/CD erstellen

### 2.1 SSH-Key generieren (auf deinem lokalen Rechner)

**Windows (PowerShell):**
```powershell
# Erstelle einen neuen SSH-Key
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/hetzner_deploy

# Zeige den öffentlichen Key an
cat ~/.ssh/hetzner_deploy.pub
```

**Linux/Mac:**
```bash
# Erstelle einen neuen SSH-Key
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/hetzner_deploy

# Zeige den öffentlichen Key an
cat ~/.ssh/hetzner_deploy.pub
```

### 2.2 Öffentlichen Key auf Hetzner Server hinzufügen

**Option A: Beim Server-Erstellen (empfohlen)**
- Füge den öffentlichen Key beim Server-Erstellen hinzu

**Option B: Nachträglich hinzufügen**
```bash
# Verbinde dich mit dem Server
ssh root@DEINE_SERVER_IP

# Füge den öffentlichen Key hinzu
mkdir -p ~/.ssh
echo "DEIN_OEFFENTLICHER_KEY_HIER" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### 2.3 Privaten Key für GitHub Actions vorbereiten

```bash
# Zeige den privaten Key an (WICHTIG: Kopiere den kompletten Inhalt!)
cat ~/.ssh/hetzner_deploy
```

**Kopiere den kompletten Inhalt** (inkl. `-----BEGIN OPENSSH PRIVATE KEY-----` und `-----END OPENSSH PRIVATE KEY-----`)

---

## 🖥️ Schritt 3: Server einrichten

### 3.1 Verbindung zum Server

```bash
# Verbinde dich mit dem Server
ssh root@DEINE_SERVER_IP

# Oder mit SSH-Key
ssh -i ~/.ssh/hetzner_deploy root@DEINE_SERVER_IP
```

### 3.2 System aktualisieren

```bash
# System aktualisieren
apt update && apt upgrade -y

# Wichtige Tools installieren
apt install -y curl wget git nano ufw
```

### 3.3 Firewall konfigurieren

```bash
# Firewall aktivieren
ufw enable

# SSH erlauben (wichtig, sonst verlierst du Zugriff!)
ufw allow 22/tcp

# HTTP und HTTPS erlauben
ufw allow 80/tcp
ufw allow 443/tcp

# Status prüfen
ufw status
```

### 3.4 Docker installieren

```bash
# Docker installieren
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Docker Compose installieren
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Docker ohne sudo verwenden (optional)
usermod -aG docker root

# Installation prüfen
docker --version
docker-compose --version
```

### 3.5 Projektverzeichnis erstellen

```bash
# Projektverzeichnis erstellen
mkdir -p /opt/clubsmarter
cd /opt/clubsmarter
```

---

## 📁 Schritt 4: Docker Compose Datei erstellen

### 4.1 Docker Compose Datei erstellen

```bash
# Erstelle docker-compose.yml
nano /opt/clubsmarter/docker-compose.yml
```

**Füge folgenden Inhalt ein:**

```yaml
services:
  # PostgreSQL Database für ClubSmarter
  postgres_clubsmarter:
    image: postgres:15-alpine
    container_name: clubsmarter_postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-vereinsmanagement}
      POSTGRES_USER: ${POSTGRES_USER:-vereinsmanagement_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-IhrSicheresPasswort}
    ports:
      - "127.0.0.1:5432:5432"  # Nur lokal erreichbar
    volumes:
      - postgres_clubsmarter_data:/var/lib/postgresql/data
    networks:
      - clubsmarter_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-vereinsmanagement_user}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ClubSmarter Backend (Django)
  clubsmarter_backend:
    image: ghcr.io/varizee/clubsmarter-backend:latest
    container_name: clubsmarter_backend
    environment:
      # Django Settings
      DJANGO_SECRET_KEY: ${DJANGO_SECRET_KEY:-change_me_in_production}
      DEBUG: ${DEBUG:-False}
      
      # Database (verwendet automatisch POSTGRES_* Werte, falls DB_* nicht gesetzt)
      DB_NAME: ${DB_NAME:-${POSTGRES_DB}}
      DB_USER: ${DB_USER:-${POSTGRES_USER}}
      DB_PASSWORD: ${DB_PASSWORD:-${POSTGRES_PASSWORD}}
      DB_HOST: postgres_clubsmarter
      
      # AWS S3 (optional)
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID:-}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY:-}
      AWS_STORAGE_BUCKET_NAME: ${AWS_STORAGE_BUCKET_NAME:-}
      AWS_S3_REGION_NAME: ${AWS_S3_REGION_NAME:-eu-central-1}
      
      # Firebase (optional)
      FIREBASE_CREDENTIALS_PATH: ${FIREBASE_CREDENTIALS_PATH:-/app/credentials/firebase_credentials.json}
    ports:
      - "127.0.0.1:8000:8000"  # Nur lokal erreichbar (Nginx als Reverse Proxy)
    volumes:
      - ./clubsmarter_backend/uploads:/app/uploads
      - ./clubsmarter_backend/logs:/app/logs
      - ./clubsmarter_backend/credentials:/app/credentials:ro
    depends_on:
      postgres_clubsmarter:
        condition: service_healthy
    networks:
      - clubsmarter_network
    restart: unless-stopped

  # ClubSmarter Admin (Flutter Web)
  clubsmarter_admin:
    image: ghcr.io/varizee/clubsmarter-admin:latest
    container_name: clubsmarter_admin
    ports:
      - "127.0.0.1:8091:80"  # Nur lokal erreichbar (Nginx als Reverse Proxy)
    depends_on:
      - clubsmarter_backend
    networks:
      - clubsmarter_network
    restart: unless-stopped

  # ClubSmarter Portal (Flutter Web) - Hauptdomain: clubsmarter.de
  clubsmarter_portal:
    image: ghcr.io/varizee/clubsmarter-portal:latest
    container_name: clubsmarter_portal
    ports:
      - "127.0.0.1:8090:80"  # Nur lokal erreichbar (Nginx als Reverse Proxy)
    depends_on:
      - clubsmarter_backend
    networks:
      - clubsmarter_network
    restart: unless-stopped

  # ClubSmarter Frontend - Standard Tenants (Flutter Web) - Subdomains: *.clubsmarter.de
  clubsmarter_frontend:
    image: ghcr.io/varizee/clubsmarter-frontend-standard:latest
    container_name: clubsmarter_frontend
    environment:
      TENANT_DOMAIN: ${TENANT_DOMAIN:-}
      IS_PREMIUM: "false"
    ports:
      - "127.0.0.1:8092:80"  # Nur lokal erreichbar (Nginx als Reverse Proxy)
    depends_on:
      - clubsmarter_backend
    networks:
      - clubsmarter_network
    restart: unless-stopped

  # ClubSmarter Frontend - Premium Tenants (Flutter Web) - Subdomains: *.clubsmarter.de
  clubsmarter_frontend_premium:
    image: ghcr.io/varizee/clubsmarter-frontend-premium:latest
    container_name: clubsmarter_frontend_premium
    environment:
      TENANT_DOMAIN: ${TENANT_DOMAIN:-}
      IS_PREMIUM: "true"
    ports:
      - "127.0.0.1:8093:80"  # Nur lokal erreichbar (Nginx als Reverse Proxy)
    depends_on:
      - clubsmarter_backend
    networks:
      - clubsmarter_network
    restart: unless-stopped

volumes:
  postgres_clubsmarter_data:
    driver: local

networks:
  clubsmarter_network:
    driver: bridge
```

**Wichtig:**
- ✅ **GitHub Username:** `varizee` (bereits eingetragen)
- ❌ **NICHT ändern:** Alle anderen Werte wie `${POSTGRES_PASSWORD}`, `${DJANGO_SECRET_KEY}`, etc. bleiben so!
- 📝 **Warum?** Diese Werte werden aus der `.env` Datei geladen (siehe Schritt 4.2)
- 🔒 **Sicherheit:** Passwörter und Keys gehören NICHT in die docker-compose.yml, sondern in die `.env` Datei!

### 4.2 Environment-Datei erstellen

**Hier kommen die echten Werte rein!** Die `.env` Datei wird von `docker-compose.yml` automatisch geladen.

```bash
# Erstelle .env Datei
nano /opt/clubsmarter/.env
```

**Füge folgenden Inhalt ein** (ersetze die Platzhalter mit echten Werten):

```env
# Django Settings
DJANGO_SECRET_KEY=django-insecure-_&0@rhas!3$x0ba(&hivg51s-)^(-d=7x+34njhmy6ab)l77y!
DEBUG=False

# Database (PostgreSQL Container)
POSTGRES_DB=vereinsmanagement
POSTGRES_USER=vereinsmanagement_user
POSTGRES_PASSWORD=IhrSicheresPasswort

# Database (für Django Backend - optional, verwendet automatisch POSTGRES_* Werte)
# DB_NAME wird automatisch aus POSTGRES_DB verwendet
# DB_USER wird automatisch aus POSTGRES_USER verwendet  
# DB_PASSWORD wird automatisch aus POSTGRES_PASSWORD verwendet
DB_HOST=postgres_clubsmarter



```


**Wichtig:** 
- ✅ **Bereits ausgefüllt mit deinen Werten:**
  - `POSTGRES_DB=vereinsmanagement` - Wird vom PostgreSQL Container verwendet
  - `POSTGRES_USER=vereinsmanagement_user` - Wird vom PostgreSQL Container verwendet
  - `POSTGRES_PASSWORD=IhrSicheresPasswort` - Wird vom PostgreSQL Container verwendet
  - `DB_HOST=postgres_clubsmarter` - Service-Name aus docker-compose (NICHT localhost!)
- 📝 **Vereinfacht:** `DB_NAME`, `DB_USER`, `DB_PASSWORD` werden automatisch aus `POSTGRES_*` übernommen (siehe docker-compose.yml)
- 🔒 **Sicherheit:** Diese Datei enthält sensible Daten - niemals committen oder teilen!

### 4.3 Verzeichnisse erstellen

```bash
# Erstelle benötigte Verzeichnisse
mkdir -p /opt/clubsmarter/clubsmarter_backend/{uploads,logs,credentials}
chmod -R 755 /opt/clubsmarter/clubsmarter_backend
```

---

## 🌐 Schritt 5: Nginx Reverse Proxy einrichten

### 5.1 Nginx installieren

```bash
# Nginx installieren
apt install -y nginx

# Nginx starten
systemctl start nginx
systemctl enable nginx
```

### 5.2 Nginx Konfiguration erstellen

```bash
# Erstelle Nginx Konfiguration
nano /etc/nginx/sites-available/clubsmarter
```

**Füge folgenden Inhalt ein** (passe die Domains an):

```nginx
# Upstream Services
upstream clubsmarter_backend {
    server 127.0.0.1:8000;
}

upstream clubsmarter_portal {
    server 127.0.0.1:8090;
}

upstream clubsmarter_admin {
    server 127.0.0.1:8091;
}

upstream clubsmarter_frontend {
    server 127.0.0.1:8092;
}

upstream clubsmarter_frontend_premium {
    server 127.0.0.1:8093;
}

# Portal - Hauptdomain: clubsmarter.de
server {
    listen 80;
    server_name clubsmarter.de www.clubsmarter.de;

    location / {
        proxy_pass http://clubsmarter_portal;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://clubsmarter_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Backend API - api.clubsmarter.de
server {
    listen 80;
    server_name api.clubsmarter.de;

    location / {
        proxy_pass http://clubsmarter_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket Support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# Admin Panel - admin.clubsmarter.de
server {
    listen 80;
    server_name admin.clubsmarter.de;

    location / {
        proxy_pass http://clubsmarter_admin;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Frontend - Standard Tenants - *.clubsmarter.de (Subdomains)
server {
    listen 80;
    server_name *.clubsmarter.de;

    # Prüfe ob Premium-Tenant (kann später erweitert werden)
    set $frontend_upstream clubsmarter_frontend;
    
    location / {
        proxy_pass http://$frontend_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://clubsmarter_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Wichtig:** 
- Hauptdomain: `clubsmarter.de` → Portal
- API: `api.clubsmarter.de` → Backend
- Admin: `admin.clubsmarter.de` → Admin Panel
- Frontend: `*.clubsmarter.de` → Frontend App (Subdomains für Tenants)

### 5.3 Nginx Konfiguration aktivieren

```bash
# Erstelle Symlink
ln -s /etc/nginx/sites-available/clubsmarter /etc/nginx/sites-enabled/

# Entferne Default-Konfiguration (optional)
rm /etc/nginx/sites-enabled/default

# Teste Nginx Konfiguration
nginx -t

# Nginx neu laden
systemctl reload nginx
```

---

## 🔒 Schritt 6: SSL Zertifikat (Let's Encrypt)

### 6.1 Certbot installieren

```bash
# Certbot installieren
apt install -y certbot python3-certbot-nginx
```

### 6.2 DNS-Einträge konfigurieren

**Bevor du SSL einrichtest, musst du DNS-Einträge erstellen:**

1. Gehe zu deinem Domain-Provider (z.B. IONOS, Cloudflare, etc.)
2. Erstelle folgende A-Records (zeige auf deine Hetzner Server IP):
   - `clubsmarter.de` → `DEINE_SERVER_IP`
   - `www.clubsmarter.de` → `DEINE_SERVER_IP`
   - `api.clubsmarter.de` → `DEINE_SERVER_IP`
   - `admin.clubsmarter.de` → `DEINE_SERVER_IP`
   - `*.clubsmarter.de` → `DEINE_SERVER_IP` (Wildcard für Subdomains/Tenants)
3. Warte bis DNS propagiert ist (kann 5-60 Minuten dauern)
4. Prüfe DNS-Propagierung: 
   ```bash
   nslookup clubsmarter.de
   nslookup api.clubsmarter.de
   nslookup admin.clubsmarter.de
   ```

### 6.3 SSL Zertifikat erstellen

```bash
# SSL Zertifikat für alle Domains erstellen (inkl. Wildcard)
certbot --nginx -d clubsmarter.de -d www.clubsmarter.de -d api.clubsmarter.de -d admin.clubsmarter.de -d *.clubsmarter.de

# Folgende Fragen beantworten:
# - E-Mail-Adresse eingeben
# - Terms of Service akzeptieren
# - E-Mail für Benachrichtigungen (optional)
# - Automatische Weiterleitung von HTTP zu HTTPS (empfohlen: 2)
```

**Hinweis:** Für Wildcard-Zertifikate (`*.clubsmarter.de`) benötigst du möglicherweise DNS-Validation statt HTTP-Validation. Falls das nicht funktioniert, erstelle separate Zertifikate für jede Subdomain.

### 6.4 Auto-Renewal testen

```bash
# Teste Auto-Renewal
certbot renew --dry-run

# Auto-Renewal ist automatisch aktiviert (systemd timer)
```

---

## 🔑 Schritt 7: CI/CD Secrets in GitHub konfigurieren

### 7.1 GitHub Secrets hinzufügen

1. Gehe zu: `https://github.com/varizee/clubsmarter_actions/settings/secrets/actions`
2. Klicke auf **"New repository secret"**

**Secret 1: Server Host**
- **Name**: `DEPLOY_SERVER_HOST`
- **Value**: Deine Hetzner Server IP (z.B. `123.456.789.0`) oder Domain
- **Add secret**

**Secret 2: Server User**
- **Name**: `DEPLOY_SERVER_USER`
- **Value**: `root` (oder dein SSH-User)
- **Add secret**

**Secret 3: SSH Private Key**
- **Name**: `DEPLOY_SSH_PRIVATE_KEY`
- **Value**: Der komplette Inhalt von `~/.ssh/hetzner_deploy` (inkl. BEGIN/END Zeilen)
- **Add secret**

**Secret 4: Projekt Pfad (Optional)**
- **Name**: `DEPLOY_PROJECT_PATH`
- **Value**: `/opt/clubsmarter`
- **Add secret** (nur wenn abweichend)

**Secret 5: GitHub Container Registry Token (Optional)**
- **Name**: `GHCR_TOKEN`
- **Value**: GitHub Personal Access Token mit `read:packages` Permission (aus Schritt 7.2)
- **Wann hinzufügen?**
  - ✅ **JA**: Wenn du in Schritt 7.2 einen GitHub Personal Access Token erstellt hast
  - ❌ **NEIN**: Wenn du Schritt 7.2 übersprungen hast (dann wird automatisch `GITHUB_TOKEN` verwendet)
- **Hinweis**: Falls du diesen Secret NICHT hinzufügst, wird automatisch `GITHUB_TOKEN` verwendet - das funktioniert in den meisten Fällen!
- **Add secret** (nur wenn du einen Token erstellt hast)

### 7.2 GitHub Personal Access Token erstellen (OPTIONAL - nur falls nötig)

**⚠️ WICHTIG: Dieser Schritt ist OPTIONAL!**
- Falls du `GHCR_TOKEN` nicht setzt, wird automatisch `GITHUB_TOKEN` verwendet
- Du kannst diesen Schritt überspringen und direkt mit Schritt 8 fortfahren
- Erstelle den Token nur, wenn du explizit einen separaten Token verwenden möchtest

**Falls du dennoch einen Token erstellen möchtest:**

1. Gehe zu: https://github.com/settings/tokens
2. Klicke auf **"Generate new token"** → **"Generate new token (classic)"**
3. **Note** (Token Name): `GitHub Actions - GHCR Access`
4. **Description** (optional): `Token für GitHub Container Registry Zugriff von Hetzner Server`
5. **Expiration**: Wähle eine Ablaufzeit (z.B. 90 Tage oder "No expiration")
6. **Repository access**: 
   - ✅ **All repositories** (empfohlen)
   - Oder: **Only select repositories** (wenn du nur bestimmte Repos freigeben willst)
7. **Permissions** (Scopes):
   - **Problem**: Die `read:packages` Berechtigung ist manchmal nicht direkt sichtbar
   - **Lösung 1**: Scrolle durch ALLE Kategorien (Actions, Admin, Codespaces, Contents, Deployments, Environments, Issues, Metadata, Packages, Pages, Pull requests, Repository secrets, Secrets, Variables, Workflows, etc.)
   - **Lösung 2**: Falls "packages" nicht gefunden wird, verwende stattdessen:
     - ✅ **"repo"** → **"Full control of private repositories"** aktivieren
     - Dies gewährt automatisch Zugriff auf Packages
   - **Lösung 3**: Falls auch das nicht funktioniert, **überspringe diesen Schritt** - der `GITHUB_TOKEN` funktioniert auch ohne expliziten Token
8. Klicke auf **"Generate token"** (ganz unten)
9. **⚠️ WICHTIG: Kopiere den Token sofort** (wird nur einmal angezeigt!)
   - Der Token beginnt mit `ghp_...`
10. **Token als Secret hinzufügen:**
    - Gehe zurück zu Schritt 7.1
    - Füge den kopierten Token als Secret `GHCR_TOKEN` hinzu
    - **Name**: `GHCR_TOKEN`
    - **Value**: Der kopierte Token (beginnt mit `ghp_...`)

**Empfehlung**: Wenn die `read:packages` Berechtigung nicht gefunden wird, überspringe diesen Schritt einfach. Das System funktioniert auch ohne expliziten `GHCR_TOKEN` - dann musst du auch Secret 5 nicht hinzufügen!

---

## 🚀 Schritt 8: Erste Deployment testen

### 8.1 GitHub Container Registry Login auf Server testen (OPTIONAL)

**⚠️ Dieser Schritt ist OPTIONAL!**
- Dieser Schritt ist nur für **manuellen Test** auf dem Server
- Für das **automatische Deployment** (Schritt 8.3) wird dieser Test NICHT benötigt
- Du kannst diesen Schritt überspringen und direkt zu Schritt 8.3 gehen

**Falls du dennoch manuell testen möchtest:**

```bash
# Verbinde dich mit dem Server
ssh root@DEINE_SERVER_IP

# Option 1: Mit GHCR_TOKEN (falls du einen erstellt hast)
echo "DEIN_GHCR_TOKEN" | docker login ghcr.io -u varizee --password-stdin

# Option 2: Mit GITHUB_TOKEN (falls du keinen GHCR_TOKEN hast)
# Erstelle einen GitHub Personal Access Token mit "read:packages" oder "repo" Berechtigung
# Dann verwende:
echo "DEIN_GITHUB_TOKEN" | docker login ghcr.io -u varizee --password-stdin

# Teste Image Pull
docker pull ghcr.io/varizee/clubsmarter-backend:latest
```

**Hinweis**: Für das automatische Deployment über GitHub Actions (Schritt 8.3) wird automatisch `GITHUB_TOKEN` verwendet, auch wenn du keinen `GHCR_TOKEN` gesetzt hast. Du musst also für das Deployment keinen Token manuell testen!

### 8.2 Docker Compose testen (OPTIONAL)

**⚠️ Dieser Schritt ist OPTIONAL!**
- Dieser Schritt ist nur für **manuellen Test** auf dem Server
- Für das **automatische Deployment** (Schritt 8.3) wird dieser Test NICHT benötigt
- Du kannst diesen Schritt überspringen und direkt zu Schritt 8.3 gehen

**Falls du dennoch manuell testen möchtest:**

```bash
# Wechsle ins Projektverzeichnis
cd /opt/clubsmarter

# Teste Docker Compose (ohne Start)
docker-compose config

# Starte Services
docker-compose up -d

# Prüfe Status
docker-compose ps

# Prüfe Logs
docker-compose logs -f
```

### 8.3 Erste Deployment via CI/CD auslösen

1. Gehe zu: `https://github.com/varizee/clubsmarter_actions/actions`
2. Wähle den **"Backend CI/CD"** Workflow
3. Klicke auf **"Run workflow"**
4. Wähle Branch: `main`
5. Klicke auf **"Run workflow"**
6. Beobachte den Workflow-Run

**Erwartetes Verhalten:**
- ✅ Build Job läuft erfolgreich
- ✅ Docker Image wird zu GHCR gepusht
- ✅ Deploy Job verbindet sich mit Server
- ✅ Docker Image wird gepullt
- ✅ Container wird neu gestartet

### 8.4 Services prüfen

```bash
# Auf dem Server: Container-Status prüfen
docker-compose ps

# Logs prüfen
docker-compose logs clubsmarter_backend
docker-compose logs clubsmarter_admin

# Services testen
curl http://localhost:8000/api/health/  # Backend
curl http://localhost:8091/              # Admin
```

---

## 🔄 Schritt 9: Datenbank-Migration (von IONOS)

### 9.1 Datenbank-Backup von IONOS erstellen

**Auf IONOS Server (oder lokal, falls du Zugriff hast):**

```bash
# PostgreSQL Dump erstellen
pg_dump -h IONOS_DB_HOST -U IONOS_DB_USER -d IONOS_DB_NAME > backup.sql

# Oder mit Passwort
PGPASSWORD=IONOS_DB_PASSWORD pg_dump -h IONOS_DB_HOST -U IONOS_DB_USER -d IONOS_DB_NAME > backup.sql
```

### 9.2 Datenbank auf Hetzner Server wiederherstellen

**Auf Hetzner Server:**

```bash
# Backup-Datei auf Server kopieren (von lokalem Rechner)
scp backup.sql root@DEINE_SERVER_IP:/tmp/

# Auf Server: Datenbank wiederherstellen
cd /opt/clubsmarter
docker-compose exec postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement < /tmp/backup.sql

# Oder direkt
docker-compose exec -T postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement < /tmp/backup.sql
```

### 9.3 Datenbank-Verbindung testen

```bash
# Verbinde dich mit Datenbank
docker-compose exec postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement

# Prüfe Tabellen
\dt

# Prüfe Tenants
SELECT * FROM public.tenants_tenant;
```

---

## 📊 Schritt 10: Monitoring & Wartung

### 10.1 Systemd Service für Auto-Start

```bash
# Erstelle Systemd Service
nano /etc/systemd/system/clubsmarter.service
```

**Füge folgenden Inhalt ein:**

```ini
[Unit]
Description=ClubSmarter Docker Compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/clubsmarter
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

**Service aktivieren:**

```bash
# Service aktivieren
systemctl daemon-reload
systemctl enable clubsmarter.service
systemctl start clubsmarter.service

# Status prüfen
systemctl status clubsmarter.service
```

### 10.2 Log-Rotation einrichten

```bash
# Erstelle Log-Rotation Konfiguration
nano /etc/logrotate.d/clubsmarter
```

**Füge folgenden Inhalt ein:**

```
/opt/clubsmarter/clubsmarter_backend/logs/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
}
```

### 10.3 Backup-Script erstellen

```bash
# Erstelle Backup-Script
nano /opt/clubsmarter/backup.sh
```

**Füge folgenden Inhalt ein:**

```bash
#!/bin/bash
BACKUP_DIR="/opt/clubsmarter/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Datenbank-Backup
docker-compose exec -T postgres_clubsmarter pg_dump -U vereinsmanagement_user vereinsmanagement > $BACKUP_DIR/db_$DATE.sql

# Alte Backups löschen (älter als 7 Tage)
find $BACKUP_DIR -name "db_*.sql" -mtime +7 -delete

echo "Backup erstellt: $BACKUP_DIR/db_$DATE.sql"
```

**Script ausführbar machen:**

```bash
chmod +x /opt/clubsmarter/backup.sh
```

**Cron-Job für tägliche Backups:**

```bash
# Crontab bearbeiten
crontab -e

# Füge folgende Zeile hinzu (täglich um 2 Uhr morgens)
0 2 * * * /opt/clubsmarter/backup.sh
```

---

## 🌐 Schritt 11: Domain-Transfer von IONOS zu Hetzner

### 11.1 Domain bei Hetzner registrieren/übertragen

**Option A: Domain zu Hetzner übertragen (empfohlen)**

1. Gehe zu: https://www.hetzner.com/domains
2. Klicke auf **"Domain übertragen"** oder **"Transfer Domain"**
3. Gib deine Domain ein: `clubsmarter.de`
4. Folge den Anweisungen für den Domain-Transfer
5. **Wichtig:** Du benötigst den **Auth-Code (EPP-Code)** von IONOS

**Auth-Code von IONOS holen:**
1. Logge dich bei IONOS ein
2. Gehe zu Domain-Verwaltung
3. Wähle `clubsmarter.de`
4. Klicke auf **"Domain übertragen"** oder **"Auth-Code anzeigen"**
5. Kopiere den Auth-Code

**Option B: Nameserver bei Hetzner verwenden (einfacher, Domain bleibt bei IONOS)**

1. Gehe zu: https://www.hetzner.com/dns-console
2. Erstelle eine neue DNS-Zone für `clubsmarter.de`
3. Notiere dir die Hetzner Nameserver (z.B. `ns1.first-ns.de`, `ns2.first-ns.de`)
4. Gehe zu IONOS Domain-Verwaltung
5. Ändere die Nameserver zu den Hetzner Nameservern
6. Warte bis DNS propagiert ist (kann 24-48 Stunden dauern)

### 11.2 DNS-Einträge bei Hetzner konfigurieren

**Falls du Option B gewählt hast (Nameserver bei Hetzner):**

1. Gehe zu: https://www.hetzner.com/dns-console
2. Wähle deine Zone: `clubsmarter.de`
3. Erstelle folgende A-Records:
   - **Name:** `@` → **Value:** `DEINE_HETZNER_SERVER_IP` → **TTL:** `3600`
   - **Name:** `www` → **Value:** `DEINE_HETZNER_SERVER_IP` → **TTL:** `3600`
   - **Name:** `api` → **Value:** `DEINE_HETZNER_SERVER_IP` → **TTL:** `3600`
   - **Name:** `admin` → **Value:** `DEINE_HETZNER_SERVER_IP` → **TTL:** `3600`
   - **Name:** `*` (Wildcard) → **Value:** `DEINE_HETZNER_SERVER_IP` → **TTL:** `3600`

**Falls du Option A gewählt hast (Domain bei Hetzner):**

Die DNS-Einträge werden automatisch bei Hetzner verwaltet. Folge den Schritten oben.

### 11.3 DNS-Propagierung prüfen

```bash
# Prüfe DNS-Propagierung (von deinem lokalen Rechner)
nslookup clubsmarter.de
nslookup www.clubsmarter.de
nslookup api.clubsmarter.de
nslookup admin.clubsmarter.de
nslookup test.clubsmarter.de  # Test für Wildcard

# Oder mit dig
dig clubsmarter.de
dig api.clubsmarter.de
```

**Wartezeit:**
- DNS-Propagierung kann 5 Minuten bis 48 Stunden dauern
- Nameserver-Änderungen: 24-48 Stunden
- A-Record-Änderungen: 5-60 Minuten

### 11.4 SSL Zertifikat für alle Domains erstellen

```bash
# SSL Zertifikat für alle Domains erstellen
certbot --nginx \
  -d clubsmarter.de \
  -d www.clubsmarter.de \
  -d api.clubsmarter.de \
  -d admin.clubsmarter.de \
  --expand  # Erweitert bestehendes Zertifikat

# Für Wildcard-Subdomains (falls benötigt):
certbot --nginx -d clubsmarter.de -d *.clubsmarter.de --manual --preferred-challenges dns
```

**Hinweis:** Wildcard-Zertifikate benötigen DNS-Validation. Du musst einen TXT-Record bei Hetzner DNS hinzufügen.

### 11.5 Services testen

Nach DNS-Propagierung und SSL-Setup:

```bash
# Teste alle Domains
curl -I https://clubsmarter.de
curl -I https://www.clubsmarter.de
curl -I https://api.clubsmarter.de/api/health/
curl -I https://admin.clubsmarter.de
curl -I https://test.clubsmarter.de  # Test für Tenant-Subdomain
```

---

## ✅ Checkliste

- [X ] Hetzner Server gekauft und eingerichtet
- [ X] Docker & Docker Compose installiert
- [ X] SSH-Key für CI/CD erstellt
- [X ] Docker Compose Datei erstellt (Backend, Admin, Portal, Frontend)
- [ X] Environment-Datei (.env) konfiguriert
- [X ] Nginx Reverse Proxy eingerichtet (Portal, API, Admin, Frontend)
- [ X warten noch] Domain clubsmarter.de zu Hetzner übertragen oder Nameserver geändert
- [ ] DNS-Einträge konfiguriert (inkl. Wildcard für Subdomains)
- [ ] SSL Zertifikat erstellt (inkl. Wildcard falls benötigt)
- [ ] GitHub Secrets konfiguriert
- [ ] Erste Deployment getestet
- [ ] Datenbank migriert (falls vorhanden)
- [ ] Systemd Service aktiviert
- [ ] Backup-Script eingerichtet
- [ ] Alle Domains getestet (clubsmarter.de, api.clubsmarter.de, admin.clubsmarter.de, *.clubsmarter.de)

---

## 🐛 Troubleshooting

### Problem: SSH-Verbindung schlägt fehl

**Lösung:**
```bash
# Prüfe SSH-Key Permissions
chmod 600 ~/.ssh/hetzner_deploy
chmod 644 ~/.ssh/hetzner_deploy.pub

# Teste SSH-Verbindung
ssh -i ~/.ssh/hetzner_deploy root@DEINE_SERVER_IP
```

### Problem: Docker Image kann nicht gepullt werden

**Lösung:**
```bash
# Prüfe GitHub Container Registry Login
echo "DEIN_TOKEN" | docker login ghcr.io -u varizee --password-stdin

# Prüfe Image-Name
docker pull ghcr.io/varizee/clubsmarter-backend:latest
```

### Problem: Container startet nicht

**Lösung:**
```bash
# Prüfe Logs
docker-compose logs clubsmarter_backend

# Prüfe Environment-Variablen
docker-compose config

# Prüfe Container-Status
docker-compose ps
```

### Problem: Nginx zeigt 502 Bad Gateway

**Lösung:**
```bash
# Prüfe ob Container laufen
docker-compose ps

# Prüfe Nginx Logs
tail -f /var/log/nginx/error.log

# Prüfe Backend-Verbindung
curl http://localhost:8000/api/health/
```

### Problem: SSL Zertifikat kann nicht erstellt werden

**Lösung:**
```bash
# Prüfe DNS-Propagierung
nslookup api.deine-domain.de

# Prüfe ob Port 80 erreichbar ist
curl -I http://api.deine-domain.de

# Prüfe Firewall
ufw status
```

---

## 📚 Weitere Ressourcen

- [Hetzner Cloud Dokumentation](https://docs.hetzner.com/)
- [Docker Dokumentation](https://docs.docker.com/)
- [Nginx Dokumentation](https://nginx.org/en/docs/)
- [Let's Encrypt Dokumentation](https://letsencrypt.org/docs/)

---

**Viel Erfolg mit deinem Hetzner Deployment! 🚀**

