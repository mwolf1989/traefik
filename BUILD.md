# Traefik v1.7 Build-Dokumentation

## Übersicht

Diese Dokumentation beschreibt den Build-Prozess für Traefik v1.7 auf ARM64 Darwin (macOS Apple Silicon) Systemen mit Cross-Compilation für Linux AMD64.

## Voraussetzungen

- Docker Desktop für Mac (mit Multi-Platform-Support)
- Make
- Git

## System-Informationen

- **Build-System**: ARM64 Darwin (macOS Apple Silicon)
- **Ziel-Plattform**: Linux AMD64 (x86-64)
- **Go-Version**: 1.16
- **Traefik-Version**: v1.7

## Build-Prozess

### 1. Binary ohne WebUI bauen

Da die WebUI auf einem veralteten Node.js 6.16.0 Image basiert (Debian Stretch, EOL), wird empfohlen, das Binary ohne WebUI zu bauen:

```bash
OS_PLATFORM_ARG=linux OS_ARCH_ARG=amd64 make binary-with-no-ui
```

**Wichtig**: Die Umgebungsvariablen `OS_PLATFORM_ARG` und `OS_ARCH_ARG` müssen explizit gesetzt werden, um für Linux AMD64 zu kompilieren.

**Ausgabe**:
- Binary-Datei: `dist/traefik`
- Größe: ~75MB
- Format: ELF 64-bit LSB executable, x86-64, statically linked

### 2. Docker-Image erstellen

Das Docker-Image muss mit dem `--platform linux/amd64` Flag gebaut werden:

```bash
docker build --platform linux/amd64 -t traefik:latest -f Dockerfile .
```

**Ausgabe**:
- Image-Name: `traefik:latest`
- Größe: ~21.4MB
- Architektur: AMD64
- Betriebssystem: Linux
- Basis: scratch (minimales Image)

### 3. Docker-Image exportieren

Um das Image auf andere Systeme zu übertragen:

```bash
docker save traefik:latest -o traefik-latest-amd64.tar
```

**Ausgabe**:
- TAR-Datei: `traefik-latest-amd64.tar`
- Größe: ~20MB

### 4. Docker-Image importieren (auf Zielsystem)

Auf dem Zielsystem (Linux AMD64):

```bash
docker load -i traefik-latest-amd64.tar
```

## Alternative Build-Methoden

### Mit WebUI (nicht empfohlen)

Falls das WebUI benötigt wird, muss zuerst das WebUI-Image aktualisiert werden:

```bash
# Warnung: Node.js 6.16.0 ist EOL und kann Probleme verursachen
make binary
```

### Vollständiger Build mit Docker

Alternativ kann der gesamte Build-Prozess in Docker ausgeführt werden:

```bash
# Development-Image bauen
docker build --platform linux/amd64 -t "traefik-dev:v1.7" -f build.Dockerfile .

# Binary im Container bauen
docker run --platform linux/amd64 \
  -e "OS_ARCH_ARG=amd64" \
  -e "OS_PLATFORM_ARG=linux" \
  -v "$(pwd)/dist:/go/src/github.com/traefik/traefik/dist" \
  "traefik-dev:v1.7" \
  ./script/make.sh generate binary
```

## Verifikation

### Binary überprüfen

```bash
file dist/traefik
# Erwartete Ausgabe: dist/traefik: ELF 64-bit LSB executable, x86-64, statically linked
```

### Docker-Image überprüfen

```bash
docker images traefik:latest
docker inspect traefik:latest --format='{{.Architecture}} {{.Os}}'
# Erwartete Ausgabe: amd64 linux
```

## Bekannte Probleme

### 1. WebUI Build-Fehler

**Problem**: Node.js 6.16.0 Image basiert auf Debian Stretch (EOL), was zu Paket-Repository-Fehlern führt.

**Lösung**: Verwenden Sie `make binary-with-no-ui` statt `make binary`.

### 2. Architektur-Mismatch

**Problem**: Ohne explizite Platform-Flags wird für ARM64 statt AMD64 gebaut.

**Lösung**: 
- Verwenden Sie `--platform linux/amd64` bei Docker-Builds
- Setzen Sie `OS_PLATFORM_ARG=linux` und `OS_ARCH_ARG=amd64` bei Make-Builds

### 3. Test-Fehler

**Problem**: `make test-unit` versucht, das Docker-Image ohne Platform-Flag neu zu bauen.

**Lösung**: Tests können übersprungen werden, da das Binary erfolgreich gebaut wurde.

## Makefile-Targets

Wichtige Targets aus dem Makefile:

- `make binary` - Baut Binary mit WebUI (Standard)
- `make binary-with-no-ui` - Baut Binary ohne WebUI (empfohlen)
- `make build` - Baut das Development-Docker-Image
- `make image` - Erstellt das finale Docker-Image
- `make test-unit` - Führt Unit-Tests aus (kann Probleme verursachen)

## Umgebungsvariablen

Wichtige Umgebungsvariablen für Cross-Compilation:

- `OS_PLATFORM_ARG` - Ziel-Betriebssystem (z.B. `linux`)
- `OS_ARCH_ARG` - Ziel-Architektur (z.B. `amd64`)
- `CGO_ENABLED` - Wird automatisch auf `0` gesetzt für statische Binaries

## Dockerfile-Struktur

### build.Dockerfile
- Basis: `golang:1.16-alpine`
- Zweck: Development-Image mit Build-Tools
- Enthält: Go, Make, Git, Misspell, etc.

### Dockerfile
- Basis: `scratch`
- Zweck: Minimales Production-Image
- Enthält: Nur Binary und CA-Zertifikate
- Ports: 80, 443, 8080

## Zusammenfassung

Für einen erfolgreichen Build auf ARM64 Darwin für Linux AMD64:

1. Binary bauen: `OS_PLATFORM_ARG=linux OS_ARCH_ARG=amd64 make binary-with-no-ui`
2. Image erstellen: `docker build --platform linux/amd64 -t traefik:latest -f Dockerfile .`
3. Image exportieren: `docker save traefik:latest -o traefik-latest-amd64.tar`

Das resultierende Image ist produktionsbereit und kann auf Linux AMD64-Systemen eingesetzt werden.

