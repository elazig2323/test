# GitHub Container Registry (GHCR) Berechtigungen - Fix

## Problem
Wenn du diesen Fehler siehst:
```
denied: installation not allowed to Create organization package
```

Das bedeutet, dass der `GITHUB_TOKEN` keine Berechtigung hat, Packages in einer Organisation zu erstellen.

## Lösung 1: Workflow-Berechtigungen (bereits implementiert)

Die Workflow-Datei wurde bereits aktualisiert mit:
```yaml
permissions:
  contents: read
  packages: write
```

Dies sollte in den meisten Fällen ausreichen. **Teste zuerst, ob es jetzt funktioniert!**

## Lösung 2: Personal Access Token (PAT) erstellen

Falls Lösung 1 nicht funktioniert, erstelle ein Personal Access Token:

### Schritt 1: PAT erstellen

1. Gehe zu GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Klicke auf "Generate new token (classic)"
3. Gib dem Token einen Namen (z.B. "GHCR Push Token")
4. Wähle die Berechtigung: **`write:packages`**
5. Klicke auf "Generate token"
6. **Kopiere den Token sofort** (er wird nur einmal angezeigt!)

### Schritt 2: Token als Secret hinzufügen

1. Gehe zu deinem GitHub Repository (`ClubSmarter_Actions`)
2. Settings → Secrets and variables → Actions
3. Klicke auf "New repository secret"
4. Name: `GHCR_TOKEN`
5. Value: Füge den kopierten Token ein
6. Klicke auf "Add secret"

### Schritt 3: Workflow testen

Der Workflow verwendet automatisch `GHCR_TOKEN`, falls vorhanden, sonst `GITHUB_TOKEN`.

## Lösung 3: Docker Hub verwenden (Alternative)

Falls GHCR weiterhin Probleme macht, kannst du auf Docker Hub wechseln:

1. Ändere in der Workflow-Datei:
   ```yaml
   env:
     REGISTRY: docker.io  # Statt ghcr.io
   ```

2. Füge Secrets hinzu:
   - `DOCKER_USERNAME`: Dein Docker Hub Benutzername
   - `DOCKER_PASSWORD`: Dein Docker Hub Access Token

## Überprüfung

Nach dem Fix sollte der Push erfolgreich sein:
```
docker push ghcr.io/varizee/clubsmarter-backend:latest
```

Die Images sind dann verfügbar unter:
- `https://github.com/orgs/varizee/packages` (für Organisationen)
- Oder in deinem Profil unter Packages (für persönliche Accounts)

## Wichtige Hinweise

- **Organisationen**: Wenn das Repository zu einer Organisation gehört, muss die Organisation Package-Erstellung erlauben
- **Repository-Sichtbarkeit**: Packages sind standardmäßig privat. Um sie öffentlich zu machen, gehe zu Package Settings → Change visibility
- **Token-Sicherheit**: Bewahre dein PAT sicher auf und teile es niemals öffentlich




