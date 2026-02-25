# KI-Readiness Assessment

Vorab-Assessment für KI-Workshops mit KMUs. Teilnehmer füllen das Formular im Browser aus – die Antworten werden per n8n-Webhook gespeichert und können anschließend mit einem Claude-Prompt zu einer strukturierten Reflexion verarbeitet werden.

## Architektur

```
[Teilnehmer]
     │
     ▼
GitHub Pages (index.html)
     │  POST JSON
     ▼
n8n Webhook (Workflow 1: Collect)
     │  Validieren → Speichern in responses.json
     ▼
responses.json  ◄──── surveys.json (Whitelist)
     │
     │  manuell ausgelöst
     ▼
n8n (Workflow 2: Analyze)
     │  Claude API
     ▼
Microsoft Teams
```

## URL-Schema

```
https://ai-assessment.bridge-brain.ai?survey=<UUID>&respondent=<UUID>
```

| Parameter | Pflicht | Beschreibung |
|---|---|---|
| `survey` | Ja | UUID des Surveys – muss in `surveys.json` eingetragen sein |
| `respondent` | Nein | UUID des Teilnehmers – für Whitelist und Re-Submission-Deduplizierung |

Ohne gültigen `survey`-Parameter wird eine Fehlermeldung angezeigt.

## Assessment-Inhalt

**16 Fragen in 4 Dimensionen:**

| Dimension | Fragen |
|---|---|
| Markt & Strategie | F1–F4 |
| Use Cases | F5–F8 |
| Menschen & Kompetenzen | F9–F12 |
| Technologie & Architektur | F13–F16 |

**Scoring:** Gewichteter Durchschnitt, Skala 0–100.
Fragen sind als M (messbar, Gewicht 1.0), R (Reflexion, Gewicht 0.3) oder M+R (Gewicht 0.6/0.8) typisiert.

**Ergebnisbereiche:**

| Score | Einordnung |
|---|---|
| 0–25 | KI-Readiness gering |
| 26–50 | Erste Schritte erkennbar |
| 51–75 | Solide Basis vorhanden |
| 76–100 | Fortgeschrittene Reife |

## Dateien

| Datei | Beschreibung |
|---|---|
| `index.html` | Das Formular (Single-Page, kein Build-Schritt) |
| `CNAME` | Custom Domain für GitHub Pages |
| `n8n-workflow.md` | Backend-Dokumentation: Datenschemata, Workflow-Nodes, Deployment-Checkliste |

## Setup

### 1. GitHub Pages

1. Repo forken oder klonen
2. Settings → Pages → Branch: `main`, Folder: `/ (root)` aktivieren
3. DNS: CNAME-Eintrag `ai-assessment` → `<github-username>.github.io` setzen
4. In `index.html` die Variable `WEBHOOK_URL` auf die eigene n8n-Webhook-URL setzen

### 2. n8n Backend

Siehe [n8n-workflow.md](n8n-workflow.md) für vollständige Anleitung inklusive:
- `surveys.json` und `responses.json` Schema
- Workflow 1 (Collect): Validierung, Sanitization, Speicherung
- Workflow 2 (Analyze): Manueller Trigger, Claude API, Teams-Nachricht
- Deployment-Checkliste

### 3. Ersten Survey anlegen

1. UUID generieren (z. B. [uuidgenerator.net](https://www.uuidgenerator.net/))
2. Eintrag in `surveys.json` auf dem n8n-Server anlegen
3. Link teilen: `https://ai-assessment.bridge-brain.ai?survey=<UUID>`

## Datenschutz

- Alle Daten verbleiben auf dem self-hosted n8n-Server (Deutschland)
- Keine Cookies, kein Tracking, kein externes CDN
- Zugriffskontrolle über UUID-Entropie (kein Login erforderlich)
- Optional: Respondent-Whitelist in `surveys.json`
