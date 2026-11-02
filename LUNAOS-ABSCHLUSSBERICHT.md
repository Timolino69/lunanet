# LunaOS Phase 1 (Web) – Abschlussbericht (09.10.2026)

## 1. Implementierte Funktionen

- Zentraler Speicherstandard (`lunaos/storage.js`): Mount-Prüfung
  (/proc/mounts, rw, 50-MB-Reserve, Schreibtest), Baum-Anlage nur bei
  verifiziertem Laufwerk, `safeJoin` (Traversal-Schutz), `checkName`,
  atomare JSON-Writes, rotierte Logs, Temp-/Backup-Pfade. vfat beachtet
  (kein chmod-Vertrauen, keine Symlink-Folge).
- Wartung: `lunaos/maintenance.js` + `maintenance/luna-maintenance.js`
  (Dry-Run Standard, nur temp/cache, nur Dateien > 7 Tage, lock/pid- und
  Scope-Schutz, Berichte + Status). systemd-User-Timer (So 04:30),
  installiert, Service-Lauf verifiziert (result ok).
- Backend `lunaos/lunaos.js` (`/api/lunaos/*`): Status, Erststart,
  Login/Logout (Rate-Limit), Sperre/Entsperren (423-Guard), Profil,
  Passwort (scrypt), Einstellungen, VFS (listen/download/read/mkdir/
  upload/write/move/copy/rename), Papierkorb (ablegen/listen/restore/
  delete/empty), Apps-Registry (9 Apps), echte Sysinfo, begrenzte Konsole
  (nur VFS, kein Host-Exec), Runtime-Schnittstelle (ehrlich 503),
  Backup (Manifest + SHA-256) + Liste, Wartungs-Steuerung.
- Frontend `public/lunaos/` (index/style/core/desktop/apps): Setup-
  Assistent (6 Schritte, Validierung), Login, Sperrbildschirm (Uhr, Datum,
  Initialen, maskiertes PW), Homescreen (Uhr, Begrüßung, Favoriten,
  Recent, alle Apps, Widgets, Dateien/ Einstellungen/ Sperren/ Abmelden),
  Desktop (Icons mit persistenten Positionen, Taskleiste, Startmenü mit
  Suche, Tray mit Speicher, Kontextmenüs, App-Drag-auf-Desktop,
  Datei-Drag im Explorer), Fenster (öffnen/schließen/min/max/restore,
  Drag, Resize, Fokus, Taskleisten-Toggle, Doppelklick-Max).
- 9 Apps, alle funktional: Explorer (CRUD, Suche, Sortierung,
  Mehrfachauswahl, Up/Download, Ordnerwahl-Dialog, DnD), Editor
  (öffnen/speichern, Dirty-Schutz), Rechner (Maus + Tastatur), Konsole,
  Bibliothek (Status, Favoriten, Desktop-Verknüpfung), Systeminfo (nur
  echte Daten), Papierkorb, Einstellungen (6 Kategorien), Externe Apps
  (Wine-/Runtime-Status, Registrierung, Start mit 503 + Bestätigung).
- LunaNet-Integration: lila LunaOS-Rail-Eintrag (eigener Monitor-Icon,
  Hover/Aktiv/Fokus), Route `#/lunaos` (iframe, gleicher Tab, kein Reload,
  Zurück-Button ok), Registry-Eintrag (Server + Frontend), Sidebar-Navi,
  Vollbild-Link, Server-Fallbacks `/lunaos/*`.

## 2. Geänderte und neue Dateien

- Neu: `lunaos/{storage,maintenance,lunaos}.js`, `lunaos/README.md`,
  `lunaos/systemd/luna-maintenance.{service,timer}`,
  `public/lunaos/{index.html,style.css,core.js,desktop.js,apps.js}`,
  `maintenance/luna-maintenance.js`, `tests-lunaos.sh`,
  `LUNA-STORAGE-STANDARD.md`, `LUNAOS-ABSCHLUSSBERICHT.md` (diese Datei).
- Geändert (additiv): `server.js` (LunaOS-Router, Registry-Eintrag,
  Fallbacks), `public/index.html` (Rail, Route, Seite, Registry).
- Backups: `server.js.bak-lunahub`, `public/index.html.bak-lunachat20`,
  `public/admin.html.bak-luna20`.
- System: `~/.config/systemd/user/luna-maintenance.{service,timer}`
  (Timer aktiviert, läuft nach nächster Anmeldung wöchentlich).

## 3./4./5./6. Architektur, Pfade, Konfiguration, Wartung

Siehe `lunaos/README.md` (Module, Datenmodelle, API) und
`LUNA-STORAGE-STANDARD.md` (Pfade, Regeln, Checkliste, Ausnahmen).
Konfiguration: nur `LUNA_DATA_ROOT` (Standard
`/run/media/tim/OTHEROS/Luna`). Wartung: wöchentlich So 04:30 per
systemd-Timer, zusätzlich manuell (`--apply`) und aus LunaOS
(Vorschau/Start + Status + Backups).

## 7./8. Tests und Ergebnisse

- `tests-lunaos.sh`: **47 passed, 0 failed** (isolierter Root/Testport,
  hinterlässt keine Daten): Setup-Validierung, Erststart, Session/CSRF,
  alle Datei-Ops inkl. Papierkorb-Zyklus, Konsole (VFS + Anti-Exec),
  Einstellungen/Profil, Backup+Manifest, Sperr-Zyklus inkl. 423-Guard
  und falschem PW, EXE-Upload→Register→Start=503, CSRF-403, Wartungs-CLI,
  LunaNet-Regression (Status, Registry, 7 Routen).
- `tests-smoke.sh` (LunaNet): **28 passed, 0 failed** – keine Regression.
- Wartung: isoliert (Dry-Run löscht nichts, Apply nur alte Dateien,
  lock/logs geschützt) + live per systemd (result ok).
- Syntax: alle JS-Dateien `node --check`; alle LunaOS-DOM-IDs verifiziert.

## 9. Bekannte Einschränkungen (ehrlich)

- Frontend-Abläufe (Drag-Gefühl, Fenster-Überlappung, Touch) sind
  implementiert und API-verifiziert, aber **kein manueller Browser-
  Abnahmetest** erfolgt – vor Produktivnutzung empfohlen.
- **Ein Profil** pro LunaOS-Installation (persönliches System; weitere
  Profile nur in Phase 2). Trennung konstruktiv vorhanden (VFS-Root je
  Profil-Key) und LunaNet-Konten strikt getrennt (eigene Sessions,
  Cookies, CSRF).
- Wine ist installiert (10.0), Ausführung aber absichtlich deaktiviert
  (Phase 2); UI sagt das überall explizit.
- systemd-Timer aktiviert, greift nach nächster Anmeldung (kein Linger);
  bis dahin manuell oder per LunaOS-Einstellungen auslösbar.
- LunaNet-Bestand bleibt in `~/Dokumente/AI-Memory` (Ausnahme, doku-
  mentiert); nur LunaOS-Neudaten liegen zentral.

## 10. Offene Aufgaben

Browser-Abnahme, ggf. Mehrprofil-Modus, Phase-2-Python (siehe 13).

## 11. Webversion starten

LunaNet starten (`./start-local.sh` o. ä.), dann LunaHub → **LunaOS**
(lila) oder direkt `http://<host>:3000/lunaos/`. Beim Erststart führt
der Assistent durch die Profilanlage (Speicher muss eingehängt sein,
sonst 503-Hinweis).

## 12. Wichtigste Funktionen testen

`./tests-lunaos.sh` (47 Checks, isoliert). Manuell: Profil anlegen →
Datei hochladen/bearbeiten → Papierkorb→Restore → Sperren/Entsperren →
Einstellungen ändern → Reload (Persistenz) → Externe Apps (503 prüfen).

## 13. Nächste Schritte Phase 2 (Python)

Eigenständiges Paket (`lunaos/`-Python): Einstieg, Config
(`LUNA_DATA_ROOT`), Benutzerverwaltung (JSON-Modelle übernehmen),
VFS-Zugriff, App-System, Desktop-Rendering (Qt/Webview), Wine-
Prozesssteuerung über lokalen Runtime-Dienst, Logs/Wartung/Backup
(Regeln übernehmen). Details: `lunaos/README.md`, Abschnitt Phase 2.
