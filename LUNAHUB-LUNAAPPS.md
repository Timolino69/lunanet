# LunaNet 2.0 – LunaHub + LunaApps

Plattform-Upgrade auf dem bestehenden LunaNet-Repository. Kein Rewrite,
keine gelöschten Funktionen, keine gebrochenen API-Verträge.

## 1. Bestand (Phase 1)

- **Backend:** Node.js + Express (`server.js`), JSON-File-Storage in
  `~/Dokumente/AI-Memory/` (Messenger, Privates, Identitäten, Admin,
  Sessions, HackNet, Logs). Kein DBMS, keine Migration nötig.
- **Frontend:** Vanilla HTML/CSS/JS in `public/index.html` (LunaChat),
  `public/admin.html` (Admin-Panel, unverändert), `hacknet/public/`
  (unverändert).
- **APIs (alle erhalten):** `/api/status`, `/api/chat`, `/api/notes`,
  `/api/logs`, `/api/drawing`, `/api/audio`, `/api/messenger`,
  `/api/messenger/status|online|presence`, `/api/privates`,
  `/api/privates/convos|read`, `/api/admin/*`, `/api/hacknet/*`.
- **Auth:** Messenger = frei wählbarer Anzeigename (Ban/Kick via Admin);
  Admin = scrypt-Hash + httpOnly-Session + CSRF + Same-Origin.
- **Design:** eigenes dunkles Luna-Gradient-System, Dark/Light per
  `data-theme`, keine Icon-Lib (offline-fähig).

## 2. Neu: Server (additiv)

Datei: `server.js`, Abschnitt „LUNANET 2.0“. Backups:
`server.js.bak-2.0`, `server.js.bak-lunahub`.

- `GET /api/modules` – Single Source of Truth der Modulmetadaten
  (`{modules, all}`). Schema pro Modul: `id, name, desc, icon, version,
  category, route, status, perms, extern?`.
- `GET /api/hubprefs?user=` / `POST /api/hubprefs` – Favoriten, Zuletzt,
  Widgets, Theme je Messenger-Name (Key = kleingeschriebener Name).
  Datei: `lunanet-hubprefs.json`. Validierung: nur registrierte Modul-IDs,
  Längenlimits, kein Code, serverseitige Prüfung (Frontend-Prüfung allein
  genügt nicht).
- SPA-Fallbacks `GET /hub /chat /apps /tools /settings /ai` (+ `/*`) →
  `public/index.html`. Bestehende URLs (`/`, `/admin.html`, `/hacknet/`,
  alle `/api/*`) unverändert.

## 3. Neu: Frontend-Plattform

Datei: `public/index.html`. Backup: `public/index.html.bak-lunachat20`
(und `.bak-2.0`).

- **Bereich 1 – Hauptnavigation (Rail):** LunaHub, LunaChat (Unread-Badge),
  LunaApps, Luna AI, LunaTools, Theme, Einstellungen. Nur registrierte
  Module als Ziele; HackNet/Admin als externe Links (neuer Tab).
- **Bereich 2 – Modulnavigation (Sidebar):** je Modul eigene Pane
  (Hub-Navi, Chat-Inbox mit Suche + Filter, Apps-Suche + Kategorien,
  Kontext-Navi für AI/Tools/Settings). Ein-/ausblendbar via Layout.
- **Bereich 3 – Hauptinhalt:** Hub-Dashboard, Chat, App-Katalog,
  AI-Chat, Tools, Einstellungen. Klare Hierarchie: Titel, Beschreibung,
  Hauptaktionen, Inhalt. Leere Zustände überall, Lade-Skelette,
  Fehlerbanner statt stiller Fehler.
- **Bereich 4 – Detailleiste:** Status, Online-Liste, Modul-Details
  (Apps), Privat-Postfach (Chat), Schnellzugriff. Ausblendbar, mobil
  als Drawer. Eigenständiges Mobile-Layout mit Bottom-Navigation.
- **Routing:** `#/hub`, `#/chat`, `#/chat/privat?peer=`, `#/ai`,
  `#/apps`, `#/apps/:id`, `#/tools`, `#/tools/:tool`, `#/settings`.
  Legacy-Anker (`#messenger-card`, `#sec-ai`, …) leiten weiter.
  Unbekannte Routen zeigen einen Hinweis statt Absturz.
- **LunaChat:** vollständig erhalten (Gruppe, Privat, Dateien, Suche,
  Zitat, Kontextmenü, Profil-Dialog, AI + Sprache, alle 13 Tools,
  Ban/Kick-Hinweise, Badges, Mitteilungs-Zentrale).

## 4. Admin-Panel

`public/admin.html`: gleiche 2.0-Design-Tokens (Dark/Light-Umschalter,
Fokus-Zustände, 44px-Touch-Ziele), eigenes SVG-Icon-Set für alle 11
Navigationspunkte, Rück-Link zum LunaHub. Alle Admin-Funktionen und
API-Aufrufe unverändert.

## 5. Tests

`./tests-smoke.sh [PORT]` – 28 Prüfungen, erstellt keine Inhalte:
Registry-Vollständigkeit, HubPrefs-Validierung + Roundtrip (mit Cleanup),
Messenger-/Admin-Regression, alle SPA-Fallbacks, Mark-up-Präsenz.
Letzter Stand: **28 passed, 0 failed** (09.10.2026).

## 6. Wiederherstellung

Backups im Projekt: `server.js.bak-2.0`, `server.js.bak-lunahub`,
`public/index.html.bak-2.0`, `public/index.html.bak-lunachat20`,
`public/admin.html.bak-2.0`. Nutzerspeicher (`~/Dokumente/AI-Memory/`)
wurde nie destruktiv angefasst; HubPrefs sind additive Zusatzdatei.
...[truncated 3881 chars]