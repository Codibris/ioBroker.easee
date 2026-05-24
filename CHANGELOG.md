# Changelog

Alle relevanten Änderungen dieses Forks werden hier dokumentiert.

Das Format folgt [Keep a Changelog](https://keepachangelog.com/de/1.1.0/),
die Versionsnummern folgen [Semantic Versioning](https://semver.org/lang/de/)
mit einem Fork-Suffix `-codibris.<n>`.

## [Unreleased]

Keine offenen Änderungen.

## [1.0.11-codibris.3] – 2026-05-24

Behebt einen seit Adapter-Anbeginn vorhandenen Einheiten-Fehler in der
Token-Ablauflogik. Symptom: bei aktivem `logtype: true` schreibt der
Adapter pro Poll-Tick eine `Token has expired - refresh`-Zeile ins
Log. Mit dem `polltime: 300`-Default aus `codibris.2` wurde das
besonders auffällig.

### Behoben
- **Token-Refresh-Thrashing** (`main.js:417` + `main.js:445`).
  Die Easee-API liefert `expiresIn` nach OAuth-Standard in **Sekunden**
  (typisch: 86400 = 24 h). Der Adapter behandelte den Wert bisher als
  Millisekunden und rechnete:
  ```js
  expireTime = Date.now() + (response.data.expiresIn - 500);
  ```
  Bei `expiresIn = 86400` ergibt das `Date.now() + 85900 ms` ≈
  85 Sekunden Token-Lebensdauer statt 24 Stunden. Konsequenz: bei jedem
  `readAllStates()`-Tick galt `expireTime <= Date.now()` und der
  Adapter feuerte einen Refresh-Call gegen `/api/accounts/refresh_token`.
  Korrigiert zu:
  ```js
  expireTime = Date.now() + (response.data.expiresIn - 30) * 1000;
  ```
  Mit `expiresIn = 86400` lebt der Token jetzt 23 h 59 m 30 s, wie vom
  API-Contract vorgesehen, mit 30 s Safety-Margin vor dem realen
  Ablauf.

### Auswirkung im Produktivbetrieb

| Kennzahl | vorher | nachher |
|---|---|---|
| Refresh-Token-Calls / Stunde | 12 (alle 5 Min bei polltime 300 s) | ≈ 0.04 (alle 24 h) |
| Log-Spam `Token has expired - refresh` | dauerhaft | nur einmal täglich |

## [1.0.11-codibris.2] – 2026-05-24

Follow-up zu `1.0.11-codibris.1`: schließt einen Schema-Inkonsistenz-Bug,
der aus dem Upstream geerbt wurde, und passt den Polltime-Default an
die neue SignalR-Realität an.

### Geändert
- **GUI-Constraint für `polltime`: `min: 360` → `min: 30`, `max: 86400` → `max: 3600`**
  (`admin/jsonConfig.json`). Das Admin-Formular hat bisher jeden Wert
  unter 360 s als "Zu klein" markiert – auch den frisch geschriebenen
  `io-package.json`-Default. Die neue Range deckt realistische
  Anwendungsfälle (30 s schneller Control-Loop bis 1 h Heartbeat) ab.
- **Help-Text** im Schema weist darauf hin, dass mit aktiviertem
  SignalR 300–600 s ausreichend sind, weil Live-Updates per
  WebSocket-Push kommen.
- **Default `polltime` von 60 s auf 300 s** (`main.js:15`,
  `io-package.json` native, `package.json` version). Da SignalR jetzt
  per Default aktiv ist und State-Push liefert, ist der REST-Poll nur
  noch nötig für Initial-Sync, Config-Refresh und als
  Reconnect-Fallback. 60 s war ein Übergangswert aus der Phase, in der
  SignalR noch deaktiviert war.

### Hintergrund
Sichtbar wurde der Schema-Bug erst nach dem `signalR: true`-Default
aus `codibris.1`: das frische Einrichten einer Instanz schreibt jetzt
`polltime: 60` (kommend von `1.0.11-codibris.1`), das GUI lehnt diesen
Wert beim ersten Öffnen der Konfiguration ab. Mit `codibris.2` sehen
neue Instanzen `polltime: 300`, was im neuen GUI-Range liegt; bestehende
Instanzen behalten ihren konfigurierten Wert und können jetzt jeden Wert
zwischen 30 s und 3600 s wählen.

## [1.0.11-codibris.1] – 2026-05-24

Erste Bugfix-Release des Codibris-Forks. Adressiert die drei
wichtigsten Stabilitätsprobleme des seit 2023 unmaintainten Upstreams
[Newan/ioBroker.easee](https://github.com/Newan/ioBroker.easee) bei
Node.js 22+ Setups mit mehreren Wallboxen.

### Hinzugefügt
- Diese `CHANGELOG.md` für nachvollziehbare Versionshistorie des Forks.
- Eintrag im "News"-Block der `io-package.json` (`1.0.11-codibris.1`)
  mit DE/EN-Beschreibung, sodass die ioBroker-Admin-UI die Änderungen
  beim nächsten Update korrekt anzeigt.

### Geändert
- **Default `polltime` von 30 s auf 60 s**
  (`main.js:15`, `io-package.json` native, `package.json` version).
  Bei 3 Wallboxen × 4 REST-Calls pro Zyklus war 30 s in der Praxis zu
  aggressiv für die Easee-Cloud und Multi-Wallbox-Setups.
- **`minPollTimeEnergy` von 120 s auf 1800 s** (`main.js:17`).
  Der Endpunkt `/api/sessions/charger/<id>/monthly` ist der
  Haupt-Verursacher von HTTP-429-Antworten. Die zurückgegebenen
  Monats-Statistiken ändern sich kaum stündlich, daher ist ein
  Refresh alle 30 Min vollkommen ausreichend.
- **`signalR` Default von `false` auf `true`**
  (`io-package.json` native). Die fixierte WebSocket-Variante
  (siehe nächster Punkt) ist stabil und liefert Live-Updates
  ohne Polling-Last.
- **README.md** komplett überarbeitet für den Fork-Kontext:
  Fork-Hinweis mit Link auf Upstream, Patches als Problem→Fix-Tabelle,
  Installations-Snippet via `iobroker url`, `chargerOpMode`-Codes als
  Tabelle.

### Behoben
- **HTTP 429 vom Session-Endpunkt** ([Upstream #86](https://github.com/Newan/ioBroker.easee/issues/86)).
  In einem 3-Wallbox-Setup wurden vor dem Patch ~1900 HTTP-429-Fehler
  pro 24 h beobachtet. Mit `minPollTimeEnergy = 1800` reduziert sich
  die Frequenz der `/sessions/charger/<id>/monthly`-Calls um den
  Faktor 15.
- **SignalR-Crash unter Node.js 22+** ([Upstream #120](https://github.com/Newan/ioBroker.easee/issues/120)).
  `@microsoft/signalr@8.x` wirft beim initialen `negotiate()`-Call
  `TypeError: Cannot read properties of undefined (reading 'secure')`.
  Folge: `UNCAUGHT_EXCEPTION`, ioBroker stoppt nach mehreren
  Restart-Versuchen die Instanz.
  **Workaround:** `skipNegotiation: true` +
  `transport: signalR.HttpTransportType.WebSockets`
  in `startSignal()` (`main.js:40-49`). Der Easee-Hub
  `streams.easee.com` unterstützt WebSockets nativ, der
  SSE/Long-Polling-Fallback wird nicht benötigt.

### Verifikation im Produktivbetrieb

Setup: 3 Wallboxen (Master + 2 Secondary an 11 kW Circuit),
Node.js 24.16.0, js-controller 7.1.0.

| Kennzahl | 1.0.10 (Upstream) | 1.0.11-codibris.1 |
|---|---|---|
| HTTP 429 / 24 h | ~1900 | 0 (seit Patch-Deploy) |
| SignalR-Start | Crash, Restart-Loop | stabil, alle 3 Wallboxen via WebSocket registriert |
| State-Update-Latenz | bis 30 s (Polling) | < 1 s (WebSocket-Push) |
| Easee-API-Calls / h (Steady-State) | ~480 | ~60 |

## Vorheriger Upstream-Stand

Diese Release basiert auf [Newan/ioBroker.easee@68af2a1](https://github.com/Newan/ioBroker.easee/commit/68af2a1)
("Merge pull request #87 from NCIceWolf/master"), dem letzten
Master-Commit des Upstream-Repos. Für die Historie davor siehe den
"Changelog"-Abschnitt in der `README.md`.

[Unreleased]: https://github.com/Codibris/ioBroker.easee/compare/v1.0.11-codibris.3...master
[1.0.11-codibris.3]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.3
[1.0.11-codibris.2]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.2
[1.0.11-codibris.1]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.1
