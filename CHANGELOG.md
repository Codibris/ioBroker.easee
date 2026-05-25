# Changelog

Alle relevanten Änderungen dieses Forks werden hier dokumentiert.

Das Format folgt [Keep a Changelog](https://keepachangelog.com/de/1.1.0/),
die Versionsnummern folgen [Semantic Versioning](https://semver.org/lang/de/)
mit einem Fork-Suffix `-codibris.<n>`.

## [Unreleased]

Keine offenen Änderungen.

## [1.0.11-codibris.5] – 2026-05-25

Addressiert einen in der Praxis beobachteten **SignalR-Zombie-Bug**:
Die WebSocket-Verbindung zu `streams.easee.com` kann aufhören
Telemetrie zu liefern, ohne dass `connection.onclose()` triggert. Der
Adapter glaubt SignalR sei in Ordnung, State-Updates kommen aber nur
noch über den 300 s REST-Poll. Symptom: Wallbox wechselt real auf
"Charging", `chargerOpMode` zeigt minutenlang weiter "AwaitingStart".

### Hinzugefügt
- **SignalR-Heartbeat-Watchdog** (`main.js:36–134`, `:onUnload`).
  Jeder `ProductUpdate`-Callback stempelt `this.lastSignalRActivity`.
  Ein `setInterval(60 s)` prüft, ob die letzte Aktivität älter als
  **120 s** ist; falls ja → `connection.stop()`, was `onclose()`
  triggert, was den existierenden Exponential-Backoff-Reconnect
  startet. 120 s sind so gewählt, dass eine im Standby befindliche
  Wallbox (die periodisch Telemetrie pusht) keine false positives
  erzeugt. Im Unload wird der Watchdog sauber via `clearInterval`
  gestoppt.
- **11 neue SignalR-Observation-IDs in `lib/enum.js`** (2024+
  Firmware):
  - `219` → `status.fatalErrorCode`
  - `220–222` → `status.lteRSRP / lteSINR / lteRSRQ`
  - `223` → `status.chargingSessionStart`
  - `230–232` → `status.eqAvailableCurrentP1/P2/P3`
  - `240` → `status.diagnosticsString`
  - `241` → `config.wiFiMACAddress`
  - `250` → `status.connectedToCloud` (nützlicher Health-Flag)
  - `251` → `status.cloudDisconnectReason`
  Quelle: pyeasee (Python-Lib hinter der Home-Assistant
  easee_hass-Integration) + offizielle Easee-Doku. IDs `233` und
  `234` bleiben undokumentiert.

### Beobachtetes Verhalten
- Vor dem Patch: Charger-Status hing nach echtem Anstecken bis zu
  5 Min auf "AwaitingStart", weil SignalR still gestorben war und
  nur der REST-Tick noch updates lieferte.
- Nach dem Patch: Watchdog erkennt die Stille spätestens 120 s nach
  dem letzten Event, erzwingt Reconnect, Easee pusht beim
  Re-Subscribe sofort den aktuellen State – Latenz wieder im
  Sekundenbereich.

## [1.0.11-codibris.4] – 2026-05-24

Größeres Härtungspaket basierend auf einem Code-Review unseres Forks
und einer Sichtung der offenen Upstream-Issues. Acht eigenständige
Bugfixes in vier Commits.

### Behoben

- **Race condition + verlorenes Polling** (`main.js:171–250`).
  `readAllStates()` nutzte `tmpAllChargers.forEach(async charger => …)`,
  das die per-Charger-Promises nicht awaitet hat. Konsequenzen: bei 3
  Wallboxen liefen die HTTP-Calls überlappend, ein Throw wurde zur
  **Unhandled Promise Rejection**, und der `setTimeout` für den
  nächsten Tick war im `if (tmpAllChargers != undefined)`-Block.
  Ein einziger Fehler in `getAllCharger()` konnte das Polling für
  immer stoppen. Jetzt: `for…of` mit per-Charger try/catch, äußerer
  try/catch/finally, `setTimeout` im `finally`.

- **SignalR-Reconnect-Loop ohne Backoff** (`main.js:39–134`).
  Bisher rief `connection.onclose` sofort wieder `this.startSignal()`
  auf — ohne Pause. Bei totem Endpunkt waren das tausende
  Negotiation-Versuche pro Minute. Jetzt: exponentielles Backoff
  (1 s → 2 s → 4 s → … max 60 s), Reset bei erfolgreichem Reconnect.
  Aktive Connection in `this.signalConnection`, Shutdown-Flag
  `this.signalRUnloaded`, sauberer `connection.stop()` in `onUnload`.

- **`refreshToken()` ohne 4xx-Fallback** (`main.js:501–539`).
  Bei `invalid_grant` (Passwortänderung, Token-Revoke) blieb der
  Adapter mit veraltetem Token zurück und produzierte endlose 401er.
  Jetzt: bei HTTP 4xx automatischer Fallback auf `login()` mit den
  gespeicherten Credentials. `expireTime = Date.now()` bei jedem
  Misserfolg, damit der nächste Tick den Refresh/Login wiederholt.

- **API-GETs ohne Retry und mit unleserlichen Fehlern**
  (`main.js:541–605`). Die fünf REST-Endpoints
  (`getAllCharger`, `getChargerState`, `getChargerConfig`,
  `getChargerSite`, `getChargerSession`) hatten jeweils ein eigenes
  `.catch()` mit `this.log.error(error)` (loggt das rohe Axios-
  Error-Objekt) und generischem `throw new Error('… - stop refresh')`.
  Refaktoriert in einen Helper `_apiGet(path, context)`:
  - **5xx**: ein Retry nach 1 s (transiente Cloud-Fehler).
  - **429**: kein Retry (würde Rate-Limit verschärfen), Warn-Log.
  - **401**: `expireTime = Date.now()`, nächster Tick re-authentifiziert.
  - Lesbare Error-Messages mit Endpoint-Kontext
    (`getChargerState(EH551234): HTTP 500 - …`).

- **`circuitMaxCurrentP2` zeigte `P3`-Wert** (`main.js:395`).
  Copy-Paste-Bug aus dem Upstream: P2-State wurde mit
  `charger_config.circuitMaxCurrentP3` befüllt.

- **`subscribeStates` für P3 war auskommentiert** (`main.js:1340`).
  User-Änderungen an `circuitMaxCurrentP3` im Admin-UI wurden
  stillschweigend ignoriert.

- **Token im Debug-Log** (`main.js:420`, `:448`).
  `login()` und `refreshToken()` haben den vollen `response.data`
  via `JSON.stringify` in den Debug-Log geschrieben — also
  `accessToken` und `refreshToken` im Klartext. Jetzt nur noch
  `expiresIn` und Token-Länge.

- **`arrCharger`-Array wurde bei Re-Init nicht geleert**
  (`main.js:136`). `this.arrCharger = []` hat eine Instance-
  Property gesetzt, der restliche Code arbeitet aber mit der
  module-level `const arrCharger`. Fix: `arrCharger.length = 0`.

- **`chargerOpMode` als Zahl statt Label** (`main.js:757`).
  Upstream-Issue #85: `chargerOpMode = 7` ist
  `AwaitingAuthentication`, war aber nicht im Object-Schema. Jetzt
  enthält das State-Objekt eine `common.states`-Map für die Werte
  0–8.

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

[Unreleased]: https://github.com/Codibris/ioBroker.easee/compare/v1.0.11-codibris.5...master
[1.0.11-codibris.5]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.5
[1.0.11-codibris.4]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.4
[1.0.11-codibris.3]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.3
[1.0.11-codibris.2]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.2
[1.0.11-codibris.1]: https://github.com/Codibris/ioBroker.easee/releases/tag/v1.0.11-codibris.1
