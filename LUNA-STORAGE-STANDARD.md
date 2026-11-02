# Luna Ecosystem Storage Standard (verbindlich, Phase 1)

Gilt für alle Luna-Projekte (LunaNet, LunaChat, LunaHub, LunaApps, LunaOS,
LunaDrive, LunaNotes, LunaPlan, LunaID, LunaNotify, LunaControl, LunaConnect,
LunaAI, …), alle Backends, alle Python-Anwendungen und Hintergrunddienste.

## 1. Basispfad

```
LUNA_DATA_ROOT=/run/media/tim/OTHEROS/Luna        (Standard)
```

- Module lesen den Pfad ausschließlich aus der Umgebungsvariable
  `LUNA_DATA_ROOT` (Fallback: Standard). Niemals an mehreren Stellen
  hartcodieren.
- Node: `lunaos/storage.js` (`dataRoot()`). Python (Phase 2) muss dieselbe
  Variable und dieselben Regeln implementieren.

## 2. Verzeichnisstruktur (Ausgangspunkt)

```
Luna/
├── apps/{LunaNet,LunaOS,shared}/
├── accounts/{users,profiles,auth}/
├── data/{LunaNet,LunaChat,LunaOS,shared}/
├── files/{users,shared}/
├── databases/
├── backups/{automatic,manual}/
├── logs/{LunaNet,LunaOS,shared}/
├── temp/{LunaNet,LunaOS,shared}/
├── cache/{LunaNet,LunaOS,shared}/
├── config/
└── maintenance/{scripts,reports,state}/
```

Verzeichnisse werden nur angelegt, wenn das Laufwerk verifiziert ist
(`driveStatus()`).

## 3. Mount- und Speichersicherheit

Vor jeder Schreib-/Migrations-/Backup-/Löschoperation prüft
`driveStatus()`:

1. Mountpoint `/run/media/tim/OTHEROS` steht in `/proc/mounts`.
2. Eingehängt mit `rw` (nicht `ro`).
3. Genug freier Speicher (Reserve: 50 MB).
4. Echter Schreibtest (Probe-Datei + Löschen) – kein blindes Vertrauen.
5. Verzeichnisstruktur vorhanden/anlegbar.

Fehlt das Laufwerk: verständlicher Fehler (HTTP 503), **kein**
stilles Ausweichen, **keine** Bereinigung, **keine** Migration,
keine Wiederholungs-Endlosschleife. Angepasste Roots (Tests,
Deployments) werden ohne Mount-Check, aber mit Beschreibbarkeits-
und Platzprüfung behandelt.

Dateisystem: OTHEROS ist `vfat` – keine Unix-Permissions. Deshalb:
atomare Writes (tmp-Datei + fsync + rename), kein chmod-Vertrauen,
keine Symlinks in verwalteten Bereichen (werden ignoriert/nicht gefolgt).

## 4. Sichere Dateioperationen (`storage.js`)

- `safeJoin(root, …)`: löst auf und wirft bei Verlassen von `root`
  (Path-Traversal-Schutz). Pflicht für alle benutzerbeeinflussten Pfade.
- `checkName(name)`: Datei-/Ordnernamen ohne Trennzeichen, max. 120 Zeichen.
- `atomicWriteJson()`: tmp + fsync + rename (crash-sicher, auch auf vfat).
- `appendLog(modul, name, level, msg)`: zentrale Logs mit Zeitstempel,
  Rotation (1 MB, 5 Generationen). Keine Passwörter/Tokens/Schlüssel.
- `tempFile()`, `backupPath()`: eindeutige, kollisionssichere Pfade.

## 5. Bereinigung (temp/cache, wöchentlich)

- Nur innerhalb `<root>/temp/**` und `<root>/cache/**` (Scope-Check vor
  jedem Löschen, Symlinks werden nie gefolgt).
- Nur reguläre Dateien mit Alter > 7 Tage (mtime).
- Schutz: `*.lock`/`*.pid`, unlesbare/unklare Dateien, alles außerhalb
  der Scopes (Konten, Profile, DBs, Nachrichten, Configs, Logs, Backups).
- Dry-Run ist Standard (`luna-maintenance.js`, ohne `--apply`).
- Idempotent, crasht keine Dienste. Berichte:
  `maintenance/reports/cleanup-<zeit>.json`,
  Status: `maintenance/state/last-cleanup.json`.
- Zeitplan: systemd-User-Timer `luna-maintenance.timer` (So 04:30,
  `lunaos/systemd/`). Bei fehlendem Laufwerk wird übersprungen.
  Manuell: `node maintenance/luna-maintenance.js [--apply]`.

## 6. Logs und Backups

- Logs: `<root>/logs/<Modul>/<name>.log` (+ `.1`…`.5`), Zeitstempel,
  Stufen, max. 1 MB, keine Secrets. Werden **nie** bereinigt.
- Backups: `<root>/backups/{automatic,manual}/<zeit>-<name>/` mit
  `manifest.json` (Zeitstempel, Version, Teile, SHA-256-Prüfsummen).
  Werden **nie** bereinigt, nie unkontrolliert überschrieben.
  Hinweis: gleiches Laufwerk = kein Schutz vor Hardwaredefekt.

## 7. Neue Module (Checkliste)

1. `LUNA_DATA_ROOT` verwenden, nie eigene Pfade erfinden.
2. Vor Schreiben: `driveStatus()` (bzw. Sprach-Äquivalent).
3. Benutzereingaben: `safeJoin` + `checkName`.
4. Temp/Cache unter `temp/<Modul>` / `cache/<Modul>` ablegen.
5. Logs nach `<root>/logs/<Modul>/`, rotiert, ohne Secrets.
6. Backups nach `<root>/backups/…`, mit Manifest + Prüfsummen.
7. Destruktives nur in isolierten Testverzeichnissen testen.

## 8. Dokumentierte Ausnahmen

- **LunaNet-Bestand** (`~/Dokumente/AI-Memory/`: Messenger, Privates,
  Identitäten, Admin, Sessions, HubPrefs, HackNet): bleibt am alten Ort.
  Grund: laufender öffentlicher Betrieb, Abwärtskompatibilität bestehender
  Sitzungen/Daten. Eine Migration erfolgt nur geplant, mit Sicherung und
  dokumentiert – nicht stillschweigend.
- **Programm-/Systemdateien**: Node-Module, Binaries, systemd-Units,
  Quellcode – keine Luna-Nutzdaten, daher ausgenommen.
