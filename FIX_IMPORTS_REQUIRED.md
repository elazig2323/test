# WICHTIG: Imports fehlen im Bitbucket-Repository

## Problem
Der CI/CD Build schlägt fehl, weil die Imports in den Dateien im **Bitbucket-Repository** fehlen.

Die lokalen Dateien haben die Imports bereits, aber der CI/CD Build holt sich die Dateien aus Bitbucket!

## Lösung: Änderungen pushen

Du musst die folgenden Dateien committen und pushen:

### 1. `lib/services/location_service.dart`
- Hat bereits: `import 'package:geolocator/geolocator.dart';`
- Hat bereits: `Geolocator.openAppSettings()` statt `openAppSettings()`

### 2. `lib/widgets/comment_widget.dart`
- Hat bereits: `import 'package:flutter/material.dart';`

### 3. `analysis_options.yaml`
- Hat bereits: Reduzierte Warnungen

## Befehle zum Pushen

```bash
cd vereinshub_frontend/vereinshub_frontend

# Status prüfen
git status

# Dateien hinzufügen
git add lib/services/location_service.dart
git add lib/widgets/comment_widget.dart
git add analysis_options.yaml

# Committen
git commit -m "Fix: Add missing imports for geolocator and material.dart"

# Pushen
git push origin main
```

## Nach dem Push

1. Warte auf den CI/CD Build
2. Der Build sollte jetzt erfolgreich sein
3. Falls weiterhin Fehler auftreten, prüfe die Build-Logs

## Verifizierung

Nach dem Push kannst du im Bitbucket-Repository prüfen:
- `lib/services/location_service.dart` sollte `import 'package:geolocator/geolocator.dart';` haben
- `lib/widgets/comment_widget.dart` sollte `import 'package:flutter/material.dart';` haben



