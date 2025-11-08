# 🚀 ClubSmarter CI/CD Pipeline

GitHub Actions CI/CD Pipeline für ClubSmarter - automatisches Builden und Deployment aller Komponenten.

## 📋 Was wird gebaut?

Die Pipeline baut automatisch bei jedem Push:

- ✅ **Backend** (Django) - Docker Image
- ✅ **Portal** (Flutter Web) - Docker Image
- ✅ **Admin** (Flutter Web) - Docker Image
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

### Schritt 2: Secret in GitHub hinzufügen

1. Gehe zu: `https://github.com/DEIN-USERNAME/clubsmarter_actions/settings/secrets/actions`
2. Klicke auf **New repository secret**
3. **Name**: `BITBUCKET_APP_PASSWORD`
4. **Value**: Das App Password aus Schritt 1
5. **Add secret**

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

## 📱 Mobile Apps

- **iOS Builds**: Werden als Artifacts gespeichert (7 Tage)
- **Android Builds**: APK und App Bundle als Artifacts (7 Tage)

## 🔗 Repositories

Die Pipeline checkt folgende Bitbucket Repositories aus:

- `vereinshub/vereinhub_backend`
- `vereinshub/vereinshub_portal`
- `vereinshub/vereinshub_admin`
- `vereinshub/vereinshub_frontend`

## 📚 Weitere Dokumentation

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

## 📝 Workflow-Datei

Die Workflow-Datei befindet sich unter:
```
.github/workflows/ci-cd-bitbucket.yml
```

---

**Viel Erfolg! 🚀**
