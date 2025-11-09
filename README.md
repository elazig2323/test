# 🚀 ClubSmarter CI/CD Pipeline

GitHub Actions CI/CD Pipeline für ClubSmarter - automatisches Builden und Deployment aller Komponenten.

## 📋 Was wird gebaut?

Die Pipeline baut automatisch bei jedem Push:

- ✅ **Backend** (Django) - Docker Image → **Automatisches Deployment** 🚀
- ✅ **Portal** (Flutter Web) - Docker Image
- ✅ **Admin** (Flutter Web) - Docker Image → **Automatisches Deployment** 🚀
- ✅ **Frontend Standard** (Flutter Web) - Docker Image
- ✅ **Frontend Premium** (Flutter Web) - Docker Image
- ✅ **Frontend iOS** (Standard & Premium) - App Builds
- ✅ **Frontend Android** (Standard & Premium) - APK & App Bundle

## 🛠️ Setup

### Schritt 1: Bitbucket App Password erstellen

1. Gehe zu: https://bitbucket.org/account/settings/app-passwords/
2. Klicke auf **Create app password**
3. **Label**: `GitHub Actions`
4. **Permissions**: ✅ **Repositories: Read**
5. Kopiere das Password

### Schritt 2: Secrets in GitHub hinzufügen

1. Gehe zu: `https://github.com/DEIN-USERNAME/clubsmarter_actions/settings/secrets/actions`
2. Klicke auf **New repository secret**

**Secret 1: Bitbucket Username**
- **Name**: `BITBUCKET_USERNAME`
- **Value**: Dein Bitbucket Username (z.B. `muazX`)
- **Add secret**

**Secret 2: Bitbucket App Password**
- **Name**: `BITBUCKET_APP_PASSWORD`
- **Value**: Das App Password aus Schritt 1
- **Add secret**

**Secret 3: Premium Tenants (Optional)**
- **Name**: `PREMIUM_TENANTS`
- **Value**: Komma-separierte Liste der Premium-Vereine (z.B. `verein1,verein2,verein3`)
- **Hinweis**: Verwende den `schema_name` oder `tenant.name` aus dem Backend
- **Add secret**

### Schritt 2.5: Deployment Secrets hinzufügen (Optional)

Für automatisches Deployment auf deinen Server:

1. Gehe zu: `https://github.com/DEIN-USERNAME/clubsmarter_actions/settings/secrets/actions`
2. Klicke auf **New repository secret**

**Secret 1: Server Host**
- **Name**: `DEPLOY_SERVER_HOST`
- **Value**: Deine Server-IP oder Domain (z.B. `123.456.789.0` oder `server.example.com`)
- **Add secret**

**Secret 2: Server User**
- **Name**: `DEPLOY_SERVER_USER`
- **Value**: SSH Username (z.B. `root` oder `ubuntu`)
- **Add secret**

**Secret 3: SSH Private Key**
- **Name**: `DEPLOY_SSH_PRIVATE_KEY`
- **Value**: Dein privater SSH-Schlüssel (kompletter Inhalt, inkl. `-----BEGIN ... -----END ...`)
- **Hinweis**: Erstelle einen separaten SSH-Key nur für Deployment
- **Add secret**

**Secret 4: Server Port (Optional)**
- **Name**: `DEPLOY_SERVER_PORT`
- **Value**: SSH Port (Standard: `22`)
- **Add secret** (nur wenn nicht Standard-Port)

**Secret 5: Projekt Pfad (Optional)**
- **Name**: `DEPLOY_PROJECT_PATH`
- **Value**: Pfad zum Docker Compose Projekt auf dem Server (Standard: `/opt/vereinshub`)
- **Add secret** (nur wenn abweichend)

**Secret 6: GitHub Container Registry Token (Optional)**
- **Name**: `GHCR_TOKEN`
- **Value**: GitHub Personal Access Token mit `read:packages` Permission
- **Hinweis**: Wird automatisch verwendet, falls nicht gesetzt wird `GITHUB_TOKEN` verwendet
- **Add secret** (nur wenn nötig)

### Schritt 3: Pipeline testen

1. Gehe zu: `https://github.com/DEIN-USERNAME/clubsmarter_actions/actions`
2. Klicke auf **CI/CD Pipeline (Bitbucket)**
3. Klicke auf **Run workflow**
4. Wähle Branch: `main`
5. Klicke auf **Run workflow**

## 📦 Docker Images

Die Docker Images werden zu **GitHub Container Registry** gepusht:

- `ghcr.io/DEIN-USERNAME/clubsmarter-backend:latest`
- `ghcr.io/DEIN-USERNAME/clubsmarter-portal:latest`
- `ghcr.io/DEIN-USERNAME/clubsmarter-admin:latest`
- `ghcr.io/DEIN-USERNAME/clubsmarter-frontend-standard:latest`
- `ghcr.io/DEIN-USERNAME/clubsmarter-frontend-premium:latest`

## 🚀 Automatisches Deployment

### Was wird automatisch deployed?

Bei jedem Push auf `main` oder `master` Branch werden automatisch deployed:

- ✅ **Backend** (clubsmarter_backend) - Django Backend
- ✅ **Admin** (clubsmarter_admin) - Flutter Web Admin Portal

### Wie funktioniert das Deployment?

1. **Build**: Docker Image wird gebaut und zu GitHub Container Registry gepusht
2. **Deploy**: Nach erfolgreichem Build wird automatisch:
   - SSH-Verbindung zum Server hergestellt
   - Neues Docker Image von GitHub Container Registry gepullt
   - Docker Compose Container neu gestartet
   - Container-Status geprüft

### Deployment-Voraussetzungen

**Auf deinem Server muss vorhanden sein:**

1. **Docker & Docker Compose** installiert
2. **Docker Compose Datei** (`docker-compose.yml`) im Projektverzeichnis
3. **GitHub Container Registry Zugriff** (wird automatisch über Token gehandhabt)
4. **SSH-Zugriff** mit dem konfigurierten SSH-Key

**Docker Compose Beispiel:**

```yaml
services:
  clubsmarter_backend:
    image: ghcr.io/DEIN-USERNAME/clubsmarter-backend:latest
    # ... weitere Konfiguration

  clubsmarter_admin:
    image: ghcr.io/DEIN-USERNAME/clubsmarter-admin:latest
    # ... weitere Konfiguration
```

### Deployment deaktivieren

Falls du das automatische Deployment nicht möchtest, entferne einfach die Deployment-Secrets oder lösche den `deploy` Job aus den Workflow-Dateien.

## 📱 Mobile Apps

### Build-Strategie

**Web (Subdomain-basiert):**
- ✅ **1 Standard-Build** für alle Standard-Vereine
- ✅ **1 Premium-Build** für alle Premium-Vereine
- Die App erkennt den Tenant zur Laufzeit über die Subdomain

**iOS & Android (Pro-Verein-Builds):**
- ✅ **1 Standard-Build** für alle Standard-Vereine
- ✅ **Pro Premium-Verein ein eigener Build** (konfiguriert über `PREMIUM_TENANTS` Secret)

### Premium-Tenants konfigurieren

1. **Premium-Vereine identifizieren:**
   - Öffne das Backend Admin-Panel oder die API
   - Finde alle Tenants mit `is_premium=true`
   - Notiere die `schema_name` oder `tenant.name` Werte

2. **Secret konfigurieren:**
   - Gehe zu: `https://github.com/DEIN-USERNAME/clubsmarter_actions/settings/secrets/actions`
   - Erstelle Secret: `PREMIUM_TENANTS`
   - Wert: Komma-separierte Liste (z.B. `shootingpoint,premiumverein1,premiumverein2`)
   - **Wichtig**: Keine Leerzeichen, nur Kommas als Trennzeichen

3. **Build-Ergebnisse:**
   - **Standard**: `app-release.apk` / `app-release.aab` (für alle Standard-Vereine)
   - **Premium**: `app-verein1-release.apk`, `app-verein2-release.apk`, etc. (pro Premium-Verein)

### Artifacts

- **iOS Builds**: Werden als Artifacts gespeichert (7 Tage)
  - Standard: `Runner.app`
  - Premium: `verein1/Runner.app`, `verein2/Runner.app`, etc.
- **Android Builds**: APK und App Bundle als Artifacts (7 Tage)
  - Standard: `app-release.apk`, `app-release.aab`
  - Premium: `app-verein1-release.apk`, `app-verein1-release.aab`, etc.

## 🔗 Repositories

Die Pipeline checkt folgende Bitbucket Repositories aus:

- `vereinshub/vereinhub_backend`
- `vereinshub/vereinshub_portal`
- `vereinshub/vereinshub_admin`
- `vereinshub/vereinshub_frontend`

## 📚 Weitere Dokumentation

- [🚀 Hetzner Deployment Guide - Schritt für Schritt](HETZNER_DEPLOYMENT_GUIDE.md) - **Komplette Anleitung für Hetzner Server Setup**
- [Detaillierte Setup-Anleitung](../SETUP_GITHUB_ACTIONS_BITBUCKET.md)
- [Docker Registry Setup](../DOCKER_REGISTRY_SETUP.md)
- [Quick Start Guide](../QUICK_START.md)

## 🐛 Troubleshooting

### Pipeline startet nicht
- Prüfe, ob `BITBUCKET_APP_PASSWORD` Secret gesetzt ist
- Prüfe, ob die Repository-URLs korrekt sind

### Checkout failed
- Prüfe, ob das App Password noch gültig ist
- Prüfe, ob das App Password die richtigen Permissions hat

### Deployment schlägt fehl

**SSH-Verbindung fehlgeschlagen:**
- Prüfe, ob `DEPLOY_SERVER_HOST`, `DEPLOY_SERVER_USER` und `DEPLOY_SSH_PRIVATE_KEY` korrekt gesetzt sind
- Teste SSH-Verbindung manuell: `ssh -i key.pem user@host`
- Prüfe, ob der SSH-Key die richtigen Permissions hat (nur für dich lesbar)

**Docker Login fehlgeschlagen:**
- Prüfe, ob `GHCR_TOKEN` oder `GITHUB_TOKEN` verfügbar ist
- Prüfe, ob der Token `read:packages` Permission hat
- Prüfe, ob das Image in GitHub Container Registry existiert

**Docker Compose Fehler:**
- Prüfe, ob `docker-compose.yml` im konfigurierten Pfad existiert
- Prüfe, ob Docker Compose auf dem Server installiert ist
- Prüfe Container-Logs: `docker-compose logs clubsmarter_backend` oder `docker-compose logs clubsmarter_admin`

**Container startet nicht:**
- Prüfe Container-Logs: `docker-compose logs clubsmarter_backend`
- Prüfe, ob alle Environment-Variablen in `docker-compose.yml` gesetzt sind
- Prüfe, ob abhängige Services (z.B. PostgreSQL) laufen

## 📝 Workflow-Datei

Die Workflow-Datei befindet sich unter:
```
.github/workflows/ci-cd-bitbucket.yml
```

---

**Viel Erfolg! 🚀**
