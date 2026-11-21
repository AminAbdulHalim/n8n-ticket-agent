# n8n Ticket-Agent

## Was das Projekt ist (und was nicht)

- Prototyp: nimmt Support-Tickets per Webhook an, klassifiziert sie mit Claude, entwirft eine Antwort und lässt einen Menschen freigeben.
- Der Versand geht nur an eine feste Testadresse, Mailpit fängt alle Mails ab, es geht nichts nach außen.
- Kein Produktivsystem: Grenzen und Ungetestetes stehen in `architektur.md`.

## Ablauf

- 1 Webhook nimmt das Formular an (400 bei leeren Pflichtfeldern, 403 ohne gültiges Token)
- 2 Claude klassifiziert (Kategorie, Priorität, Begründung)
- 3 Code-Knoten prüft die Antwort (Stufe 0 bis 3)
- 4 Claude entwirft eine Antwort (nicht bei `unklar`)
- 5 Mensch gibt per E-Mail frei oder lehnt ab (24 h Frist)
- 6 Versand an die Testadresse
- 7 Log-Eintrag in `ticket_log`
- Details, Fehlerverhalten und Status: `architektur.md`

## Voraussetzungen

- Docker (mit Compose)
- Eigener Anthropic-API-Key (nur als Credential in n8n, nie im Repo)
- Modell: `claude-sonnet-5-5` (steht in den Knoten „Claude klassifizieren“ und „Claude Entwurf“)
- PowerShell für den Beispieltest
- Feste Versionen: n8n `2.46.6`, Mailpit `v1.34.0` (in `docker-compose.yml`)

## Start

- Stand: am 20.11.2026 aus einem frischen Klon nachgebaut, mit leerer n8n-Instanz und neuem Konto. Gelaufen ist der Weg über die Test URL (Webhook, Freigabe per Mail, Versand, Log-Zeile). Die Production URL und „Publish“ wurden im Nachbau nicht getestet. Auf dem ursprünglichen Rechner sind beide URLs geprüft
- `docker compose up -d`
- n8n: `http://127.0.0.1:5678`. Beim ersten Start erscheint die Registrierung: Konto anlegen. Danach folgt ein Fenster „Customize n8n to you“ mit Fragen, die nicht Pflicht sind (es geht auch ohne Antworten weiter). Ein Dialog zum Verbinden von n8n mit AI (MCP) ließ sich mit „Skip“ überspringen, der Nachbau lief ohne ihn
- Mailpit (Posteingang für alle Mails): `http://127.0.0.1:8025`
- Beide nur vom eigenen Rechner erreichbar

## Einrichtung in n8n

- Workflow importieren: `workflows/ticket-agent.json`
- Credentials werden nicht mitexportiert, drei selbst anlegen und in den Knoten neu wählen:
  - Anthropic: Header Auth, Name `x-api-key`, Wert ist der eigene API-Key (für „Claude klassifizieren“ und „Claude Entwurf“)
  - SMTP für Mailpit: Host `mailpit`, Port `1025`, SSL/TLS aus, die übrigen Felder bleiben unverändert. Benutzername: beliebig (im Original `amin`, im Nachbau `test`, beides lief). Passwort: beliebiger Text (im Nachbau lief ein selbst gewählter Wert, leer nicht getestet). Beim Speichern meldet n8n einen erfolgreichen Verbindungstest
  - Webhook-Token: Header Auth mit Name `X-Ticket-Token`, Credential-Name `ticket-webhook-token`, Wert ist das Token aus dem Abschnitt „Test“
- Data table `ticket_log` von Hand anlegen (Overview → Data tables → Create data table), Spalten `ticket_id`, `status`, `grund` jeweils als Text. `id`, `createdAt` und `updatedAt` sind von Anfang an in jeder neuen Tabelle enthalten. Die Tabelle ist nicht im Workflow-Export enthalten. Der Knoten „Log schreiben“ zeigt nach dem Import zwar den Namen `ticket_log`, verweist aber auf eine Tabelle, die es in der neuen Instanz nicht gibt: Die Auswahlliste war leer und n8n meldete „Error fetching data table details … status code 404“. Danach die neue Tabelle im Knoten aus der Liste wählen. Die Wertzuordnung (`ticket_id`, `status`, `grund`) steht schon aus dem Import im Knoten
- Der Zeitstempel der Tabelle ist im Node-Output UTC, die Oberfläche zeigt lokale Zeit
- Knoten mit Credentials einzeln öffnen und prüfen: „Webhook“, beide Claude-Knoten, „Send message and wait for response“, „Versand an Ersteller“, „Log schreiben“. n8n trägt beim Öffnen eines Knotens eine vorhandene Credential gleichen Typs automatisch ein. Im Nachbau stand dadurch am Webhook-Knoten zuerst die Anthropic-Credential statt `ticket-webhook-token` (beide Header Auth). Deshalb die Webhook-Credential getrennt anlegen und am Webhook-Knoten ausdrücklich `ticket-webhook-token` wählen

## Test

- Token erzeugen und in die Zwischenablage legen, ohne den Wert anzuzeigen (gilt nur für dieses PowerShell-Fenster), danach in die Webhook-Credential einfügen:

```powershell
$b = [byte[]]::new(32)
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
$env:TICKET_TOKEN = [Convert]::ToBase64String($b)
$env:TICKET_TOKEN | Set-Clipboard
```

- Die Variable `$env:TICKET_TOKEN` gilt nur in diesem PowerShell-Fenster. In einem neuen Fenster ist sie leer, und `Invoke-RestMethod` bricht dann mit „Der Objektverweis wurde nicht auf eine Objektinstanz festgelegt“ ab (gemessen: Länge 0, diese Meldung; ob die Anfrage n8n noch erreicht, nicht geprüft). Wer das Token neu erzeugt, muss es auch in die Credential `ticket-webhook-token` einfügen. Passen beide Werte nicht zusammen, ist es für n8n ein falsches Token und es kommt 403 (403 bei falschem Token ist gemessen, der Fall „neu erzeugtes Token, alte Credential“ selbst nicht gesondert getestet)

- Workflow in n8n auf „Listen for test event“ stellen (Test URL, Pfad `ticket`), dann die Anfrage senden:

```powershell
$body = @{ absender_email = "test@example.com"; betreff = "Drucker defekt"; text = "Mein Drucker druckt nicht mehr." } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:5678/webhook-test/ticket" `
  -Headers @{ "X-Ticket-Token" = $env:TICKET_TOKEN } -ContentType "application/json" -Body $body
```

- Erwartet: 200 mit `{ "status": "angenommen", "ticket_id": … }` (PowerShell zeigt es als Tabelle mit den Spalten `status` und `ticket_id`), 400 bei leerem Feld, 403 ohne oder mit falschem Token (PowerShell meldet dann einen Fehler)
- Danach Freigabemail in Mailpit öffnen und auf „Approve“ oder „Decline“ klicken
- Beide URLs sind geprüft: 403 ohne und mit falschem Token, ein gültiges Token läuft über die Production URL bis zum Versand
- Die Production URL gilt nur, wenn der Workflow veröffentlicht ist (Schalter „Publish“), und braucht kein „Listen for test event“. Der Pfad ist dort `/webhook/ticket` statt `/webhook-test/ticket`

## Demo-Ablauf

Ablauf für eine kurze Vorführung. Die Zeiten sind nicht gemessen, deshalb stehen hier keine. Belege stehen in `tests.md`.

| Nr. | Station | Was gezeigt wird | Beleg |
|---|---|---|---|
| 1 | Was es ist | Prototyp, nur feste Testadresse, Mailpit fängt alle Mails ab, kein Produktivsystem | README |
| 2 | Ticket senden | PowerShell-Anfrage mit Token, Antwort 200 `angenommen` mit `ticket_id` | `tests.md` Nr. 1 |
| 3 | Workflow | Klassifikation, Prüfung und Entwurf im Editor | Workflow-Export |
| 4 | Mensch entscheidet | Freigabemail in Mailpit, „Approve“ klicken | `tests.md` Nr. 1 |
| 5 | Versand und Log | Versandmail an die Testadresse, Log-Zeile `gesendet` in `ticket_log` | `tests.md` Nr. 1 |
| 6 | Ablehnen | „Decline“: Status `entwurf_abgelehnt` (ob dabei eine Mail rausging, ist in Nr. 10 nicht festgehalten) | `tests.md` Nr. 10 |
| 7 | Grenzen | Keine Benachrichtigung bei `manuell_pruefen`, der Freigabelink ist die Berechtigung, Prompt Injection nur abgemildert | `architektur.md` |

## Grenzen

- Der Webhook ist nur durch ein gemeinsames Token geschützt, ohne Mengenbegrenzung
- Der Freigabelink ist die Berechtigung: Wer die Mail hat, kann freigeben
- Prompt Injection ist abgemildert (Tag-Abgrenzung, Freigabe durch einen Menschen), nicht gelöst
- Es gibt keine Benachrichtigung bei `manuell_pruefen`, Tickets stehen nur in `ticket_log`
- Alles Weitere: `architektur.md`, Abschnitte „Grenzen“ und „Geplant, nicht gebaut“

## Offenlegung

- Vorgegeben (Kurs): Aufgaben, Hinweise und Prüflisten, das Gerüst der Architektur-Tabelle mit sieben Schritten
- Nach Hinweisen, gemeinsam mit Claude erarbeitet: Prüf-Code (Stufe 0 bis 3), n8n-Ausdrücke, Entwurfs-Prompt, `architektur.md`, `README.md`
- Selbst: Aufbau in n8n (Knoten, Credentials, Verbindungen), Läufe und Testergebnisse, Klassifikations-Prompt (Kategorien, Prioritätsregeln, VPN-Regel), Commit-Nachrichten
- Nur gegengeprüft: der Workflow-Export (Verbindungen, Felder, Geheimnisse), nicht das Laufzeitverhalten. Die KI hat n8n selbst nie gesehen, nur Dateien, Ausgaben und Screenshots

## Lizenz

MIT, siehe `LICENSE`. Das gilt nur für den Inhalt dieses Repos. n8n, Mailpit und Claude haben eigene Lizenzen und Nutzungsbedingungen.
