# Zentrales LunaNet-Auth & Sperrbildschirm (Master-Prompt)

Gilt für alle gewöhnlichen LunaNet-Bereiche: Dashboard, LunaChat, alle
gewöhnlichen Apps/Module/Unterseiten (inkl. zukünftiger – Schutz liegt an
der zentralen Middleware, nicht an jedem Modul).

## 1. Geänderte Dateien

- Neu: `lunaos/lunauth.js` (Konten, Sessions, View-Grants, Sperr-Prefs),
  `tests-lunauth.sh` (35 Checks), diese Datei.
- Geändert: `server.js` (18 Routen + `requireLunauth`, Identitätsbindung,
  Registry unverändert), `public/index.html` (Sperrbildschirm, Fetch-
  Patch, Konto- statt Namen-Einstellung, Sperr-Personalisierung),
  `tests-smoke.sh` (isoliert + Auth).
- Wiederverwendet: `lunaos/storage.js` (Mount-Check, safeJoin, Logs,
  Backups), Admin-Sicherheitsmuster (scrypt, Sessions, CSRF, Rate-Limit),
  LunaOS-Sperr-Design (`public/lunaos/` als Referenz), alle Messenger-
  Datenmodelle und -Speicher (keine Migration).

## 2. Anmeldung (zentral)

`POST /api/lunauth/login {user, pass}` → prüft **immer** das Passwort
(scrypt, serverseitig). Antwort: `{user, csrf, view, lock}`. `view` lebt
im Speicher der Seite (plus Sitzungs-Backup für Reloads, siehe unten).
Session-Cookie (`lunauth_sess`, httpOnly, SameSite=Strict, **ohne
Max-Age**) allein genügt für keine Inhalte, überlebt aber Refresh und
Hintergrund; erst der App-Neustart löscht es. `POST /api/lunauth/resume`
stellt bei gültiger Session (Same-Origin) einen frischen Grant ohne
Passwort aus – dadurch bleibt man bei Reload/Hintergrund angemeldet,
nach App-Neustart kommt der Sperrbildschirm. Erster Start:
`POST /api/lunauth/setup` (nur bei 0 Konten). Selbst-Registrierung
jederzeit: `POST /api/lunauth/register {user, pass, pass2, display?}`
(mit eigenem Rate-Limit, Auto-Login per Grant). Weitere Konten:
`POST /api/lunauth/users` (angemeldet). Der Sperrbildschirm hat einen
Umschalter „Anmelden ↔ Konto anlegen“ (Modi login/register, plus
Setup-Modus beim Erststart); Enter in Benutzername springt ins
Passwortfeld, Enter im Passwortfeld löst aus. Neutrale Fehler,
Rate-Limit (5/5 Min pro IP). **Mehrgeräte:** Ein Konto kann parallel
auf mehreren Geräten angemeldet sein (z. B. Desktop + Mobil) – ein
neuer Login kickt andere Sitzungen nicht mehr. Beliebig viele
parallele Sitzungen pro Konto (kein Limit); jede Sitzung hat eigene
Session-/CSRF-/View-Tokens. Fixation-Schutz bleibt: Jeder Login
erzeugt frische Zufalls-Tokens. Logout entwertet nur die eigene
Session + Grant. Keine Passwörter/Tokens in URLs,
localStorage oder Logs.

## 3. Reload-Zwang

Jeder vollständige Dokument-Neuladen verwirft den Grant (Speicher) →
Sperrbildschirm erscheint, Anmeldung wird serverseitig neu geprüft.
Es wird keine Maske über Inhalte gelegt: APIs liefern ohne Grant 401,
die App startet erst nach Grant (kein Polling, keine Inhalte vorher).

## 4. Interne Navigation

Hash-Routing (`#/hub`, `#/chat`, …) lädt kein Dokument neu → Grant im
Speicher bleibt → kein erneuter Login zwischen Dashboard, LunaChat und
Modulen. 401/423 aus einer API führt kontrolliert zurück zum Sperrbild.

## 5. LunaChat-Integration

- Autor/Sender/Leser/Key aus Session (`req.lunauthUser`); Client-Namen
  werden ignoriert (kein Spoofing, keine fremden Konversationen).
- Peer muss Messenger-Identität **oder** LunaNet-Konto sein (frische
  Konten erreichbar, Verlauf/Kontakte erhalten).
- Ban/Kick-, Rate-Limit-, Anhangs- und Längenprüfungen unverändert.
- Kein WebSocket vorhanden (nur Polling, gleiche Auth).
- Notizen/Zeichnungen/Audio/Logs/AI-Chat: angemeldet nutzbar, Daten
  unverändert (gemeinsame Haushalts-Ressourcen, dokumentiert).

## 6. Weitere geschützte Module

Alle gewöhnlichen APIs über `requireLunauth`: messenger*, privates*,
notes, logs, drawing, audio, chat, hubprefs (Key strikt aus Session).
Öffentlich bleiben nur: `/api/status`, `/api/modules`,
`/api/lunauth/{status,setup-needed,setup,login,lockpreview,lockbg}`.

## 7. Ausnahmen (eigene Grenzen, getestet)

- **LunaOS** (`/lunaos/*`, `/api/lunaos/*`): eigener Sperrbildschirm,
  eigene Sessions/Cookies/CSRF – keine Kopplung, keine Auto-Logins.
- **HackNet** (`/hacknet*`, `/api/hacknet/*`): Mechanismen unverändert.
- **AdminPanel** (`/admin.html`, `/api/admin/*`): scrypt-Sessions, Rollen
  und serverseitige Prüfungen unverändert; LunaNet-Login gibt keinen
  Admin-Zugriff. Ausnahme gilt nur für die Sperrbildschirm-Logik.

## 8./9. Personalisierung & Speicherung

Hintergrund (Orbit/Dämmerung/Wald/Schlicht/eigenes Bild), Akzent
(Lila/Blau/Grün), Uhr/Datum an-aus, Live-Vorschau, Zurücksetzen.
Upload: nur PNG/JPEG/GIF/WebP (Magic-Bytes), max 2 MB, sicherer Name
(`bg.<ext>`), Traversal-frei, serviert mit `nosniff`.
Speicherung zentral: `data/LunaNet/lock/<key>.json`,
Bilder `files/users/<key>/lockbg/`. Vorschau/Stil pro Benutzer getrennt;
Stil bewusst öffentlich (Sperrbildschirm erscheint vor Login).

## 10. Tests (tatsächlich ausgeführt)

- `tests-lunauth.sh`: **66/66** (Setup-Validierung, Hash-Nachweis,
  Cookie-ohne-Grant-401, Grant-Lesen, neutrale Fehler, CSRF-403,
  Spoof-Schutz, Grant-Manipulation, Logout-Revoke, Re-Login, Mehrgeräte
  (parallele Grants, beidseitiges Lesen/Schreiben, Logout nur eigenes
  Gerät, alte Sitzung überlebt Neuanmeldung), unbegrenzte parallele
  Sitzungen (12-fach verifiziert, älteste+neueste aktiv), Konten-CRUD,
  case-insensitiver Login, Anzeigename
  (CRUD/Validierung/Verzeichnis), Register (Validierung/Auto-Login/
  Duplikat/Rate-Limit-429), Prefs-Trennung, Prefs-Validierung, Preview,
  Upload-Typ/Größe, Reset, 5 Ausnahmen, Secret-Freiheit der Logs).
- `tests-smoke.sh`: **41/41** (inkl. Cursor-Einbindung) ·
  `tests-lunaos.sh`: **47/47**.
- Live-Verifikation: Autorbindung, Verlaufserhalt, Ausnahmen-HTTP-Codes.

## 11. Einschränkungen

- Bestehende Messenger-Verläufe bleiben ihren Display-Namen zugeordnet;
  wer denselben Namen als Konto wählt, behält Historie/Präsenz nahtlos.
- Gemeinsame Ressourcen (Notizen, Zeichnungen, Audio, Logs) sind
  haushaltsweit sichtbar (wie bisher), jetzt hinter Anmeldung.
- Ohne eingehängtes OTHEROS-Laufwerk: 503 mit Hinweis (kein stilles
  Ausweichen, per Storage-Standard). LunaNet-Bestand in
  `~/Dokumente/AI-Memory` bleibt unangetastet (dokumentierte Ausnahme).
- Kein manueller Browser-Abnahmetest des Sperrbildschirm-Flows erfolgt
  (API-verifiziert; empfohlen).

## 12. Betrieb (wichtig)

- Produktiv läuft LunaNet unter **systemd** (`lunanet.service` +
  `lunanet-tunnel.service`, `Restart=always`). Neustart via
  `systemctl --user restart lunanet.service` – **nicht** nur per
  `manager.sh`, da dessen pkill gegen den Supervisor verliert
  (Port bleibt belegt, Neustart schlägt fehl).
- Tests nutzen isolierte Ports/Roots; Messenger-/Privates-Testposts
  werden am Suite-Ende automatisch aus dem Home-Store entfernt.
