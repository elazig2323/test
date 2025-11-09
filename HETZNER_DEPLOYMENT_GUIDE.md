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

**⚠️ WICHTIG: Dieser Schritt ist OBLIGATORISCH!**
- Die `docker-compose.yml` Datei muss auf dem Server existieren, bevor das Deployment funktioniert
- Diese Datei wird NICHT automatisch vom Deployment-Skript erstellt
- Ohne diese Datei funktioniert `docker-compose ps` nicht (Fehler: "no configuration file provided")

### 4.1 Docker Compose Datei erstellen

```bash
# Stelle sicher, dass du im richtigen Verzeichnis bist
cd /opt/clubsmarter

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
DJANGO_SECRET_KEY="django-insecure-_&0@rhas!3\$x0ba(&hivg51s-)^(-d=7x+34njhmy6ab)l77y!"
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
- ⚠️ **WICHTIG - DJANGO_SECRET_KEY:**
  - Der Secret Key muss in **Anführungszeichen** gesetzt werden: `DJANGO_SECRET_KEY="..."` 
  - Das `$` Zeichen muss escaped werden: `\$` (sonst interpretiert docker-compose es als Variable)
  - **Falls du Warnungen wie "The 'x0ba' variable is not set" bekommst:**
    - Öffne die `.env` Datei: `nano /opt/clubsmarter/.env`
    - Setze den `DJANGO_SECRET_KEY` in Anführungszeichen und escape das `$`: `DJANGO_SECRET_KEY="...\$..."`
    - Speichere und teste: `docker-compose config` (sollte keine Warnungen mehr zeigen)

### 4.3 Verzeichnisse erstellen

```bash
# Erstelle benötigte Verzeichnisse
mkdir -p /opt/clubsmarter/clubsmarter_backend/{uploads,logs,credentials}
chmod -R 755 /opt/clubsmarter/clubsmarter_backend
```

### 4.4 Erstellung bestätigen

```bash
# Prüfe, ob docker-compose.yml existiert
ls -la /opt/clubsmarter/docker-compose.yml

# Prüfe, ob .env existiert
ls -la /opt/clubsmarter/.env

# Teste docker-compose Syntax (sollte keine Fehler zeigen)
cd /opt/clubsmarter
docker-compose config
```

**✅ Wenn alles korrekt ist:**
- `docker-compose config` zeigt die geparste Konfiguration ohne Fehler
- Du kannst jetzt mit Schritt 5 fortfahren

**❌ Falls Fehler auftreten:**
- Prüfe, ob die YAML-Syntax korrekt ist (Einrückungen!)
- Prüfe, ob beide Dateien existieren: `ls -la /opt/clubsmarter/`

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

**Verfügbare Deployments:**
- ✅ **Backend CI/CD** - Django Backend
- ✅ **Admin CI/CD** - Flutter Admin Interface
- ✅ **Portal CI/CD** - Flutter Portal Interface (neu hinzugefügt!)

**Backend Deployment:**
1. Gehe zu: `https://github.com/varizee/clubsmarter_actions/actions`
2. Wähle den **"Backend CI/CD"** Workflow
3. Klicke auf **"Run workflow"**
4. Wähle Branch: `main`
5. Klicke auf **"Run workflow"**
6. Beobachte den Workflow-Run

**Admin Deployment:**
1. Wähle den **"Admin CI/CD"** Workflow
2. Klicke auf **"Run workflow"**
3. Wähle Branch: `main`
4. Klicke auf **"Run workflow"**

**Portal Deployment:**
1. Wähle den **"Portal CI/CD"** Workflow
2. Klicke auf **"Run workflow"**
3. Wähle Branch: `main`
4. Klicke auf **"Run workflow"**

**Erwartetes Verhalten:**
- ✅ Build Job läuft erfolgreich
- ✅ Docker Image wird zu GHCR gepusht
- ✅ Deploy Job verbindet sich mit Server
- ✅ Docker Image wird gepullt
- ✅ Container wird neu gestartet

### 8.4 Services prüfen

```bash
# WICHTIG: Wechsle zuerst ins Projektverzeichnis!
cd /opt/clubsmarter

# Prüfe, ob docker-compose.yml existiert
ls -la docker-compose.yml

# Auf dem Server: Container-Status prüfen
docker-compose ps

# Logs prüfen
docker-compose logs clubsmarter_backend
docker-compose logs clubsmarter_admin

# Services testen
curl http://localhost:8000/api/health/  # Backend
curl http://localhost:8091/              # Admin
curl http://localhost:8090/              # Portal
curl http://localhost:8092/              # Frontend (Standard)
```

### 8.5 Alle Services testen und Portal prüfen

**1. Container-Status prüfen:**

```bash
# Prüfe, ob alle Container laufen
cd /opt/clubsmarter
docker-compose ps

# Erwartete Ausgabe: Alle Container sollten "Up" sein
# - clubsmarter_postgres: Up (healthy)
# - clubsmarter_backend: Up (healthy)
# - clubsmarter_admin: Up
# - clubsmarter_portal: Up (falls gestartet)
# - clubsmarter_frontend: Up (falls gestartet)
```

**2. Logs prüfen:**

```bash
# Backend Logs (sollte keine Fehler zeigen)
docker-compose logs clubsmarter_backend --tail=50

# Admin Logs
docker-compose logs clubsmarter_admin --tail=50

# Portal Logs (falls gestartet)
docker-compose logs clubsmarter_portal --tail=50

# Alle Logs gleichzeitig
docker-compose logs --tail=50
```

**3. API-Endpunkte testen:**

```bash
# Backend Health Check
curl http://localhost:8000/api/health/
# Erwartete Antwort: {"status": "ok"} oder ähnlich

# Backend API Root
curl http://localhost:8000/api/
# Sollte eine API-Übersicht zurückgeben

# Test-Tenant API (mit Domain)
curl -H "Host: testverein.localhost" http://localhost:8000/api/
# Sollte tenant-spezifische API zurückgeben
```

**4. Web-Services testen (lokal):**

```bash
# Admin Interface (Port 8091)
curl -I http://localhost:8091/
# Sollte HTTP 200 zurückgeben

# Portal (Port 8090, falls gestartet)
curl -I http://localhost:8090/
# Sollte HTTP 200 zurückgeben

# Frontend (Port 8092, falls gestartet)
curl -I http://localhost:8092/
# Sollte HTTP 200 zurückgeben
```

**5. Portal im Browser testen (falls Nginx konfiguriert ist):**

Falls du Nginx bereits konfiguriert hast (Schritt 5), kannst du die Services über die Domain aufrufen:

```bash
# Prüfe, ob Nginx läuft
systemctl status nginx

# Prüfe Nginx-Konfiguration
nginx -t

# Teste Domains (von deinem lokalen Rechner aus):
# - https://clubsmarter.de (Portal)
# - https://admin.clubsmarter.de (Admin)
# - https://api.clubsmarter.de/api/health/ (Backend API)
# - https://testverein.clubsmarter.de (Frontend für Test-Tenant)
```

### 8.6 Nächste Schritte nach Portal-Deployment

**✅ Sofort prüfen:**

```bash
# 1. Prüfe, ob Portal-Container läuft
cd /opt/clubsmarter
docker-compose ps clubsmarter_portal

# 2. Prüfe Portal-Logs
docker-compose logs clubsmarter_portal --tail=50

# 3. Teste Portal lokal
curl -I http://localhost:8090/
# Sollte HTTP 200 zurückgeben

# 4. Prüfe alle Container
docker-compose ps
```

**📋 Checkliste - Was ist noch zu tun?**

- [ ] **Portal läuft** - Container-Status prüfen (`docker-compose ps`)
- [ ] **Portal lokal getestet** - `curl -I http://localhost:8090/` sollte HTTP 200 zurückgeben
- [ ] **Nginx konfiguriert** - Falls noch nicht geschehen, siehe **Schritt 5**
- [ ] **DNS konfiguriert** - Domain auf Server-IP zeigen lassen, siehe **Schritt 11**
- [ ] **SSL-Zertifikat erstellt** - HTTPS einrichten, siehe **Schritt 11**
- [ ] **Portal über Domain erreichbar** - `https://clubsmarter.de` im Browser testen

**🔧 Falls Portal nicht läuft:**

**Problem: `no such service: clubsmarter_portal`**

Dies bedeutet, dass der Portal-Service nicht in der `docker-compose.yml` auf dem Server ist. Füge ihn hinzu:

```bash
# 1. Öffne die docker-compose.yml
nano /opt/clubsmarter/docker-compose.yml

# 2. Füge den Portal-Service hinzu (nach clubsmarter_admin, vor clubsmarter_frontend):
```

**Füge diesen Abschnitt ein:**

```yaml
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
```

**Wichtig:** Stelle sicher, dass die Einrückung korrekt ist (2 Leerzeichen) und dass der Service unter `services:` eingerückt ist!

```bash
# 3. Speichere die Datei (Ctrl+O, Enter, Ctrl+X)

# 4. Prüfe die Syntax
docker-compose config

# 5. Starte den Portal-Container
docker-compose up -d clubsmarter_portal

# 6. Prüfe den Status
docker-compose ps clubsmarter_portal
```

**Andere Probleme:**

```bash
# Prüfe Logs
docker-compose logs clubsmarter_portal --tail=100

# Prüfe, ob Backend erreichbar ist (Portal benötigt Backend)
docker-compose ps clubsmarter_backend

# Starte Portal neu
docker-compose restart clubsmarter_portal

# Falls das nicht hilft, starte alle Container neu
docker-compose down
docker-compose up -d
```

**🌐 Nginx konfigurieren (falls noch nicht geschehen):**

Falls Nginx noch nicht konfiguriert ist, siehe **Schritt 5** in der Anleitung. Das Portal sollte dann über die Domain erreichbar sein.

**🔒 SSL-Zertifikat einrichten (falls noch nicht geschehen):**

Falls SSL noch nicht eingerichtet ist, siehe **Schritt 11** in der Anleitung.

**6. Tenant-spezifische Tests:**

```bash
# Prüfe, ob der Test-Tenant existiert
docker-compose exec clubsmarter_backend python manage.py shell

# Im Python Shell:
from apps.tenants.models import Tenant, Domain
tenants = Tenant.objects.all()
for t in tenants:
    print(f"Tenant: {t.name} (Schema: {t.schema_name})")
    domains = Domain.objects.filter(tenant=t)
    for d in domains:
        print(f"  Domain: {d.domain}")
exit()

# Teste Tenant-API mit korrektem Host-Header
curl -H "Host: testverein.localhost" http://localhost:8000/api/health/
```

**7. Vollständiger Service-Test:**

```bash
# Erstelle ein Test-Script
cat > /tmp/test_services.sh << 'EOF'
#!/bin/bash
echo "=== Container Status ==="
docker-compose ps

echo -e "\n=== Backend Health ==="
curl -s http://localhost:8000/api/health/ | head -1

echo -e "\n=== Admin Interface ==="
curl -sI http://localhost:8091/ | head -1

echo -e "\n=== Portal Interface ==="
curl -sI http://localhost:8090/ | head -1

echo -e "\n=== Test-Tenant API ==="
curl -sH "Host: testverein.localhost" http://localhost:8000/api/health/ | head -1

echo -e "\n=== Test abgeschlossen ==="
EOF

chmod +x /tmp/test_services.sh
/tmp/test_services.sh
```

**Erwartete Ergebnisse:**
- ✅ Alle Container laufen (Status: Up)
- ✅ Backend API antwortet mit `{"status": "ok"}` oder ähnlich
- ✅ Admin Interface lädt (HTTP 200)
- ✅ Portal Interface lädt (HTTP 200, falls gestartet)
- ✅ Tenant-API funktioniert mit korrektem Host-Header

**⚠️ Fehlerbehebung:**

**Problem: Container startet ständig neu (Restarting)**

Falls ein Container ständig neu startet (z.B. `clubsmarter_admin` oder `clubsmarter_portal`):

```bash
# 1. Prüfe die Logs des fehlerhaften Containers
docker-compose logs clubsmarter_admin --tail=100
# oder
docker-compose logs clubsmarter_portal --tail=100

# 2. Prüfe die Container-Details
docker-compose ps

# 3. Prüfe die Container-Logs direkt
docker logs clubsmarter_admin
# oder
docker logs clubsmarter_portal

# 4. Häufige Ursachen:
# - Fehlende Umgebungsvariablen
# - Fehlerhafte Konfiguration
# - Port-Konflikte
# - Fehlende Abhängigkeiten (z.B. Backend nicht erreichbar)

# 5. Prüfe, ob das Backend läuft (Admin/Portal benötigen Backend)
docker-compose ps clubsmarter_backend

# 6. Prüfe die Netzwerk-Verbindung
docker-compose exec clubsmarter_admin ping clubsmarter_backend
# oder
docker-compose exec clubsmarter_portal ping clubsmarter_backend

# 7. Starte den Container manuell neu
docker-compose restart clubsmarter_admin
# oder
docker-compose restart clubsmarter_portal

# 8. Falls das nicht hilft, starte alle Container neu
docker-compose down
docker-compose up -d
```

**Problem: Backend ist "unhealthy"**

```bash
# 1. Prüfe die Backend-Logs
docker-compose logs clubsmarter_backend --tail=100

# 2. Prüfe die Health-Check-Logs
docker inspect clubsmarter_backend | grep -A 10 Health

# 3. Teste die Backend-API manuell
curl http://localhost:8000/api/health/

# 4. Prüfe, ob die Datenbank erreichbar ist
docker-compose exec clubsmarter_backend python manage.py check --database default

# 5. Prüfe die Datenbank-Verbindung
docker-compose exec postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement -c "SELECT 1;"

# 6. Häufige Ursachen:
# - Datenbank nicht erreichbar
# - Fehlende Migrationen
# - Fehlerhafte .env Konfiguration
# - Port-Konflikte

# 7. Starte Backend neu
docker-compose restart clubsmarter_backend
```

**Problem 1: `no configuration file provided: not found`**
1. **Prüfe, ob du im richtigen Verzeichnis bist:** `pwd` (sollte `/opt/clubsmarter` sein)
2. **Prüfe, ob die Datei existiert:** `ls -la /opt/clubsmarter/docker-compose.yml`
3. **Falls die Datei NICHT existiert:**
   - Die `docker-compose.yml` muss manuell erstellt werden (siehe **Schritt 4**)
   - Das Deployment-Skript erstellt diese Datei NICHT automatisch
   - Gehe zurück zu **Schritt 4.1** und erstelle die Datei
4. **Falls die Datei existiert, aber du trotzdem den Fehler bekommst:**
   - Wechsle ins Verzeichnis: `cd /opt/clubsmarter`
   - Prüfe die Syntax: `docker-compose config`

**Problem 2: `relation "tenants_domain" does not exist` oder `database "vereinsmanagement_user" does not exist`**

Dies bedeutet, dass die Datenbank-Migrationen noch nicht ausgeführt wurden. Führe folgende Schritte aus:

**WICHTIG: Behebe zuerst die DJANGO_SECRET_KEY Warnung!**

```bash
# 1. Öffne die .env Datei
nano /opt/clubsmarter/.env

# 2. Finde die Zeile mit DJANGO_SECRET_KEY
# Ändere von:
# DJANGO_SECRET_KEY=django-insecure-_&0@rhas!3$x0ba(&hivg51s-)^(-d=7x+34njhmy6ab)l77y!
# Zu:
DJANGO_SECRET_KEY="django-insecure-_&0@rhas!3\$x0ba(&hivg51s-)^(-d=7x+34njhmy6ab)l77y!"

# 3. Speichere (Ctrl+O, Enter, Ctrl+X)
# 4. Teste: docker-compose config (sollte keine Warnungen mehr zeigen)
# 5. Starte Container neu
docker-compose restart clubsmarter_backend
```

**Dann führe die Migrationen aus:**

```bash
# Stelle sicher, dass du im Projektverzeichnis bist
cd /opt/clubsmarter

# Prüfe, ob die Container laufen
docker-compose ps

# Führe zuerst die Standard-Migrationen aus (für das public Schema)
docker-compose exec clubsmarter_backend python manage.py migrate

# Dann die Tenant-Migrationen (für das shared Schema)
docker-compose exec clubsmarter_backend python manage.py migrate_schemas --shared

# Falls Fehler "relation does not exist" auftreten, führe die Migrationen erneut aus
# Manchmal müssen Migrationen mehrmals ausgeführt werden, wenn Abhängigkeiten bestehen
docker-compose exec clubsmarter_backend python manage.py migrate_schemas --shared
```

**Problem 3: Falscher Datenbankname (`database "vereinsmanagement_user" does not exist`)**

Prüfe die `.env` Datei:

```bash
# Öffne die .env Datei
nano /opt/clubsmarter/.env

# Stelle sicher, dass folgende Werte korrekt sind:
# POSTGRES_DB=vereinsmanagement          (NICHT vereinsmanagement_user!)
# POSTGRES_USER=vereinsmanagement_user    (Das ist der USER, nicht die DB!)
# DB_NAME sollte NICHT gesetzt sein (wird automatisch aus POSTGRES_DB übernommen)

# Speichere und starte Container neu
docker-compose restart clubsmarter_backend
```

**Problem 4: `relation "core_mitglied" does not exist` während Migrationen**

Dies tritt auf, wenn Migrationen in falscher Reihenfolge ausgeführt werden. Die `admin` Migration versucht auf `core_mitglied` zuzugreifen, bevor die `core` Migrationen abgeschlossen sind.

**Lösung 1: Migrationen in der richtigen Reihenfolge ausführen**

```bash
# 1. Prüfe, welche Migrationen bereits ausgeführt wurden
docker-compose exec clubsmarter_backend python manage.py showmigrations

# 2. Führe ZUERST ALLE ausstehenden core Migrationen aus (wichtig!)
# Dies stellt sicher, dass alle core Tabellen erstellt sind, bevor admin Migrationen ausgeführt werden
docker-compose exec clubsmarter_backend python manage.py migrate core

# 3. Dann die anderen Apps (contenttypes und auth sind bereits fertig)
docker-compose exec clubsmarter_backend python manage.py migrate contenttypes
docker-compose exec clubsmarter_backend python manage.py migrate auth

# 4. Jetzt können die admin Migrationen ausgeführt werden
docker-compose exec clubsmarter_backend python manage.py migrate admin

# 5. Dann alle anderen ausstehenden Migrationen
docker-compose exec clubsmarter_backend python manage.py migrate

# 6. Dann die Tenant-Migrationen (für das shared Schema)
docker-compose exec clubsmarter_backend python manage.py migrate_schemas --shared
```

**Wichtig:** Falls `migrate core` fehlschlägt, führe die Migrationen Schritt für Schritt aus:

```bash
# Führe core Migrationen einzeln aus (falls migrate core fehlschlägt)
docker-compose exec clubsmarter_backend python manage.py migrate core 0002_team_mitglied_hauptteam
docker-compose exec clubsmarter_backend python manage.py migrate core 0003_event
# ... usw. für alle ausstehenden core Migrationen
# Oder einfach:
docker-compose exec clubsmarter_backend python manage.py migrate core --fake-initial
docker-compose exec clubsmarter_backend python manage.py migrate core
```

**Falls `admin` Migrationen immer noch fehlschlagen, obwohl core Migrationen ausgeführt wurden:**

**Wichtig:** Bei django-tenants werden App-Tabellen (wie `core_mitglied`) in **Tenant-Schemas** erstellt, nicht im `public` Schema. Das `public` Schema enthält nur shared Tabellen (z.B. `tenants_tenant`, `tenants_domain`).

Das Problem: Die `admin.0001_initial` Migration versucht auf `core_mitglied` zuzugreifen, aber diese Tabelle existiert nur in Tenant-Schemas, nicht im public Schema.

**Lösung: Migrationen für das shared Schema ausführen**

```bash
# 1. Prüfe, welche Schemas existieren
docker-compose exec postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement -c "\dn"

# 2. Prüfe, welche Tabellen im public Schema sind (sollten nur shared Tabellen sein)
docker-compose exec postgres_clubsmarter psql -U vereinsmanagement_user -d vereinsmanagement -c "\dt public.*"

# 3. Führe die Migrationen für das SHARED Schema aus (nicht public!)
# Das shared Schema ist das Schema, das von allen Tenants geteilt wird
docker-compose exec clubsmarter_backend python manage.py migrate_schemas --shared

# 4. Falls die admin Migration immer noch fehlschlägt, könnte sie falsch konfiguriert sein
# Versuche, die admin Migration zu überspringen oder zu fixen:
docker-compose exec clubsmarter_backend python manage.py migrate admin --fake
docker-compose exec clubsmarter_backend python manage.py migrate

# 5. Dann alle anderen Migrationen
docker-compose exec clubsmarter_backend python manage.py migrate
```

**Alternative: Verwende den create_test_verein Command**

Das Backend hat einen Management-Command, der automatisch einen Test-Verein erstellt:

```bash
# Erstelle einen Test-Verein (erstellt automatisch Tenant, Domain, Verein und Admin-Mitglied)
docker-compose exec clubsmarter_backend python manage.py create_test_verein \
  --name=testverein \
  --email=admin@testverein.de \
  --password=admin1234 \
  --domain=localhost

# Der Command erstellt automatisch:
# - Tenant mit Schema "testverein"
# - Domain "testverein.localhost"
# - Verein im Tenant-Schema
# - Admin-Mitglied mit Email admin@testverein.de

# Dann führe Migrationen für alle Tenants aus (inkl. den neuen Test-Verein)
docker-compose exec clubsmarter_backend python manage.py migrate_schemas
```

**Manuelle Alternative (falls der Command nicht funktioniert):**

```bash
# Erstelle einen Test-Tenant manuell
docker-compose exec clubsmarter_backend python manage.py shell

# Im Python Shell:
from apps.tenants.models import Tenant, Domain
tenant = Tenant.objects.create(schema_name='test', name='Test Verein', is_premium=False)
Domain.objects.create(domain='test.localhost', tenant=tenant, is_primary=True)
exit()

# Führe Migrationen für alle Tenants aus
docker-compose exec clubsmarter_backend python manage.py migrate_schemas
```

**Lösung 2: Falls Lösung 1 nicht funktioniert - Datenbank zurücksetzen (NUR wenn keine wichtigen Daten vorhanden sind!)**

```bash
# ⚠️ WARNUNG: Dies löscht alle Daten in der Datenbank!
# Nur ausführen, wenn keine wichtigen Daten vorhanden sind!

# 1. Container stoppen
docker-compose down

# 2. Datenbank-Volume löschen
docker volume rm clubsmarter_postgres_clubsmarter_data

# 3. Container neu starten
docker-compose up -d

# 4. Warte bis PostgreSQL bereit ist (ca. 10 Sekunden)
sleep 10

# 5. Migrationen in der richtigen Reihenfolge ausführen
docker-compose exec clubsmarter_backend python manage.py migrate core
docker-compose exec clubsmarter_backend python manage.py migrate
docker-compose exec clubsmarter_backend python manage.py migrate_schemas --shared
```

**Nach den Migrationen:**
- Prüfe die Logs: `docker-compose logs clubsmarter_backend`
- Teste die API: `curl http://localhost:8000/api/health/`
- Die Fehler sollten jetzt verschwunden sein

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

