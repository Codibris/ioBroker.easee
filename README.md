![Logo](admin/easee.png)
# ioBroker.easee – Codibris Fork

[![GitHub release](https://img.shields.io/github/v/release/Codibris/ioBroker.easee?include_prereleases)](https://github.com/Codibris/ioBroker.easee/releases)
[![Last commit](https://img.shields.io/github/last-commit/Codibris/ioBroker.easee)](https://github.com/Codibris/ioBroker.easee/commits)
[![License](https://img.shields.io/github/license/Codibris/ioBroker.easee)](LICENSE)

> **Fork-Hinweis:** Dieser Fork liefert Bugfixes für den nicht mehr gepflegten [Newan/ioBroker.easee](https://github.com/Newan/ioBroker.easee)-Adapter
> (letztes Upstream-Release **1.0.10 vom Juli 2023**, 23 offene Issues).
> Schwerpunkt: HTTP-429-Rate-Limit-Mitigation und SignalR-Reparatur unter Node.js 22+.
> Vollständige Versionshistorie und Verifikations-Messdaten in [CHANGELOG.md](CHANGELOG.md).

## Easee Wallbox Adapter für ioBroker

Verbindet ioBroker mit der Easee Wallbox API (Cloud) und liefert Status- und Konfigurationsdaten
sowie Steuerbefehle für eine oder mehrere Wallboxen. Updates kommen wahlweise per REST-Polling
oder – bevorzugt – per SignalR-WebSocket-Push.

## Installation

```bash
iobroker url 'https://github.com/Codibris/ioBroker.easee/tarball/master' easee
```

Bei mehreren Wallboxen am gemeinsamen Stromkreis empfohlene Konfiguration:

| Setting | Default in diesem Fork | Empfehlung |
|---|---|---|
| `polltime` | 60 s | 60 s (REST-Fallback) |
| `signalR` | true | true – Live-Updates via WebSocket |

## Unterschiede zum Upstream (Newan/ioBroker.easee 1.0.10)

| Problem im Upstream | Fix in diesem Fork |
|---|---|
| **HTTP 429 vom `/api/sessions/charger/<id>/monthly`-Endpoint** ([#86](https://github.com/Newan/ioBroker.easee/issues/86)) – bei 3 Wallboxen ~1900 Fehler/Tag | `minPollTimeEnergy` von 120 s auf **1800 s** (`main.js:17`). Monats-Statistiken ändern sich kaum stündlich. |
| **`polltime` Default 30 s** zu aggressiv für Multi-Wallbox-Setups | Default auf **60 s** angehoben (`main.js:15`, `io-package.json`). |
| **`@microsoft/signalr` crash unter Node.js 22+** ([#120](https://github.com/Newan/ioBroker.easee/issues/120)) – `Cannot read properties of undefined (reading 'secure')` beim Negotiation-Step → Adapter killt sich im Restart-Loop | `skipNegotiation: true` + `transport: WebSockets` in `startSignal()` (`main.js:40-49`). Easee-Hub `streams.easee.com` unterstützt WebSockets nativ. |
| `signalR: false` als Default | `signalR: true` als Default – die fixierte WebSocket-Variante ist jetzt stabil. |

## Hilfreiches

`chargerOpMode`-Codes:

| Code | Bedeutung |
|---|---|
| 0 | Offline |
| 1 | Disconnected |
| 2 | AwaitingStart |
| 3 | Charging |
| 4 | Completed |
| 5 | Error |
| 6 | ReadyToCharge |

`dynamicCircuitCurrentPX` – alle drei Phasen müssen innerhalb von 500 ms per Skript gesetzt werden, sonst wird die jeweilige Phase auf 0 gesetzt.


## Changelog
<!--
  Placeholder for the next version (at the beginning of the line):
  ### **WORK IN PROGRESS**
-->
### 1.0.11-codibris.1 (2026-05-24) – Fork by Codibris
* SignalR (`signalR: true`) als Default aktiv – Live-Updates via WebSocket statt 30s-REST-Polling
* Polltime-Default von 30s auf 60s (für Multi-Wallbox-Setups schonender für die Easee-API)
* `minPollTimeEnergy` von 120s auf 1800s – Session-Monatsdaten ändern sich kaum, Faktor 15 weniger Calls auf `/api/sessions/charger/<id>/monthly` (Hauptverursacher von HTTP 429-Fehlern)

### 1.0.10 (2023-07-27)
* (Newan) fix version number

### 1.0.9 (2023-07-27)
* (walburgf)  changed API URL from api.easee.cloud to api.easee.com
* (walburgf)  created addition parameter in admin config to reduce/steer logging information for user
* (walburgf)  modified internationalization to use jsonConfig.json. this needs at least ioBroker.admin version 5
* (walburgf)  added dependency to admin >=v5.1.28

### 1.0.8 (2023-07-02)
* (Newan)  small fixes

### 1.0.7
* (Newan) Changed login URL

### 1.0.6
* (Newan) Changed that smart charging is editable

### 1.0.5
* (marwin79) More Features supported and convert values to expected datatypes

### 1.0.4
* (Newan) dynamicCircuitCurrentPX writeable (set all Phases in 500ms) to limit ampere

### 1.0.3
* (Newan) Adapter crash fixed an other bugfixes

### 1.0.1
* (Newan) Add circuitMaxCurrentPX to limit current ampere

### 1.0.0
* (Newan) Stable Version with SignalR

## Donation
[![](https://www.paypalobjects.com/de_DE/DE/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=L55UBQJKJEUJL)

## License
MIT License

Copyright (c) 2025 Newan <iobroker@newan.de>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
