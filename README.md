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

- Stand: Start und Einrichtung sind noch nicht von Grund auf nachgebaut
- `docker compose up -d`
- n8n: `http://127.0.0.1:5678` (beim ersten Start Konto anlegen)
- Mailpit (Posteingang für alle Mails): `http://127.0.0.1:8025`
- Beide nur vom eigenen Rechner erreichbar

## Einrichtung in n8n

- Workflow importieren: `workflows/ticket-agent.json`
- Credentials werden nicht mitexportiert, drei selbst anlegen und in den Knoten neu wählen:
  - Anthropic: Header Auth, Name `x-api-key`, Wert ist der eigene API-Key (für „Claude klassifizieren“ und „Claude Entwurf“)
  - SMTP für Mailpit: Host `mailpit`, Port `1025`. Die Credential enthält bei mir einen Benutzernamen (Mailpit zeigte `amin`). Ob Mailpit beliebige Werte akzeptiert, ist nicht geprüft, das klärt der Nachbau-Test
  - Webhook-Token: Header Auth mit Name `X-Ticket-Token`, Credential-Name `ticket-webhook-token`, Wert ist das Token aus dem Abschnitt „Test“
- Data table `ticket_log` anlegen, Spalten `ticket_id`, `status`, `grund` (Text), im Knoten „Log schreiben“ neu wählen
- Der Zeitstempel der Tabelle ist im Node-Output UTC, die Oberfläche zeigt lokale Zeit
- Knoten mit Credentials prüfen: „Webhook“, beide Claude-Knoten, „Send message and wait for response“, „Versand an Ersteller“, „Log schreiben“

## Test

- Token erzeugen und in die Zwischenablage legen, ohne den Wert anzuzeigen (gilt nur für dieses PowerShell-Fenster), danach in die Webhook-Credential einfügen:

```powershell
$b = [byte[]]::new(32)
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
$env:TICKET_TOKEN = [Convert]::ToBase64String($b)
$env:TICKET_TOKEN | Set-Clipboard
```

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
