# n8n Workflows – KI-Readiness Assessment

## Architektur

```
Workflow 1 – Collect (automatisch, per Webhook)
  [Webhook] → [Code: Validieren & Sanitize] → [Code: Speichern] → [Respond 200]

Workflow 2 – Analyze (manuell ausgelöst)
  [Manual Trigger] → [Code: Filtern & Prompt] → [HTTP: Claude API] → [Teams]
```

---

## Konfiguration

Lege auf dem n8n-Server ein Datenverzeichnis an und passe den Pfad in den Code-Nodes an:

```bash
DATA_DIR=/your/path/to/data     # z. B. /home/n8n/data
mkdir -p $DATA_DIR
echo '[]' > $DATA_DIR/responses.json
# surveys.json initial befüllen (siehe Schema unten)
```

In jedem Code-Node sind die Pfade als Variablen ganz oben definiert – **nur dort anpassen**:

```javascript
const surveysPath   = '/data/surveys.json';   // ← anpassen
const responsesPath = '/data/responses.json'; // ← anpassen
```

---

## Datenschemata

### `surveys.json` – Whitelist der aktiven Surveys

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "template": "ai-readiness-assessment-0",
    "description": "KI-Workshop Februar 2026",
    "comments": "Erstes Pilotunternehmen",
    "created": "2026-02-01",
    "respondents": [
      "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "3e4a1b2c-5d6e-7f8a-9b0c-1d2e3f4a5b6c"
    ]
  }
]
```

- `id` – UUID des Surveys (kommt als `?survey=` URL-Parameter)
- `template` – Name der HTML-Vorlage (ohne `.html`), z. B. `ai-readiness-assessment-0`; bestimmt die URL `/<template>.html?survey=<id>`
- `respondents` – optionale Whitelist; wenn vorhanden, wird `respondent_id` dagegen geprüft
- Wenn `respondents` fehlt oder leer: jeder mit gültigem `survey_id` darf antworten

### `responses.json` – Gespeicherte Antworten

Liegt auf dem n8n-Server, z. B. `/home/n8n/data/responses.json`.

```json
[
  {
    "survey_id": "550e8400-e29b-41d4-a716-446655440000",
    "respondent_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "submitted_at": "2026-02-24T09:15:00.000Z",
    "unternehmen": "Beispiel GmbH",
    "name": "Max Mustermann",
    "funktion": "Geschäftsführer",
    "datum": "24.02.2026",
    "scores": {
      "gesamt": 42,
      "dimensionen": {
        "Markt & Strategie": 55,
        "Use Cases": 38,
        "Menschen & Kompetenzen": 31,
        "Technologie & Architektur": 44
      }
    },
    "answers": {
      "f1": { "value": 50, "text": "Ansatzweise – KI ist Thema..." },
      "f2": { "value": 25, "text": "Kaum – wir haben..." }
    }
  }
]
```

- Re-Submissions (gleiche `survey_id` + `respondent_id`) **überschreiben** den bestehenden Eintrag.
- Wenn `respondent_id` leer ist, wird jede Einreichung als neuer Datensatz angehängt.

---

## Workflow 1 – Collect

### Node 1: Webhook

| Feld | Wert |
|---|---|
| Method | POST |
| Path | `/ki-assessment` |
| Authentication | None (UUID-Entropie als Access Control) |
| Response Mode | Using 'Respond to Webhook' Node |

### Node 2: Code – Validieren & Sanitize

```javascript
const body = $input.first().json.body ?? $input.first().json;

// Hilfsfunktion: String bereinigen und kürzen
function sanitize(val, maxLen = 200) {
  if (typeof val !== 'string') return '';
  return val.trim().replace(/[<>"'`]/g, '').slice(0, maxLen);
}

// UUID-Format prüfen
function isUUID(val) {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(val);
}

const survey_id = sanitize(body.survey_id, 36);
const respondent_id = sanitize(body.respondent_id, 36);

if (!isUUID(survey_id)) {
  throw new Error('INVALID_SURVEY_ID');
}

// surveys.json laden und survey_id prüfen
const fs = require('fs');
const surveysPath = '/data/surveys.json'; // ← Pfad anpassen
const surveys = JSON.parse(fs.readFileSync(surveysPath, 'utf8'));
const survey = surveys.find(s => s.id === survey_id);

if (!survey) {
  throw new Error('UNKNOWN_SURVEY_ID');
}

// Respondent-Whitelist prüfen (falls vorhanden)
if (survey.respondents && survey.respondents.length > 0) {
  if (!isUUID(respondent_id) || !survey.respondents.includes(respondent_id)) {
    throw new Error('INVALID_RESPONDENT_ID');
  }
}

// Felder sanitizen
const cleaned = {
  survey_id,
  respondent_id: isUUID(respondent_id) ? respondent_id : '',
  submitted_at: new Date().toISOString(),
  unternehmen: sanitize(body.unternehmen),
  name: sanitize(body.name),
  funktion: sanitize(body.funktion),
  datum: sanitize(body.datum, 20),
  scores: body.scores,   // aus JS berechnet, kein User-Input
  answers: Object.fromEntries(
    Object.entries(body.answers ?? {}).map(([k, v]) => [
      sanitize(k, 10),
      { value: Number(v.value), text: sanitize(v.text, 300) }
    ])
  ),
};

return [{ json: cleaned }];
```

### Node 3: Code – Speichern in responses.json

```javascript
const fs = require('fs');
const responsesPath = '/data/responses.json'; // ← Pfad anpassen

const entry = $input.first().json;

// Bestehende Daten laden (oder leeres Array)
let responses = [];
if (fs.existsSync(responsesPath)) {
  responses = JSON.parse(fs.readFileSync(responsesPath, 'utf8'));
}

// Re-Submission: bestehenden Eintrag ersetzen (nur wenn respondent_id gesetzt)
if (entry.respondent_id) {
  const idx = responses.findIndex(
    r => r.survey_id === entry.survey_id && r.respondent_id === entry.respondent_id
  );
  if (idx >= 0) {
    responses[idx] = entry;
  } else {
    responses.push(entry);
  }
} else {
  responses.push(entry);
}

fs.writeFileSync(responsesPath, JSON.stringify(responses, null, 2), 'utf8');

return [{ json: { success: true, survey_id: entry.survey_id } }];
```

### Node 4: Respond to Webhook

| Feld | Wert |
|---|---|
| Response Code | 200 |
| Response Body | `{ "ok": true }` |

> Bei einem Fehler in Node 2 oder 3 antwortet n8n automatisch mit 500. Das Frontend zeigt dann eine Fehlermeldung.

---

## Workflow 2 – Analyze

### Node 1: Manual Trigger

Kein Parameter nötig. Workflow wird manuell in n8n gestartet.

> **Vor dem Start:** Im Code-Node (Node 2) die Variable `SURVEY_ID` auf die gewünschte Survey-UUID setzen.

### Node 2: Code – Filtern & Prompt aufbauen

```javascript
const fs = require('fs');

// ─── HIER ANPASSEN ──────────────────────────────────────────
const SURVEY_ID = '550e8400-e29b-41d4-a716-446655440000';
// ────────────────────────────────────────────────────────────

const responsesPath = '/data/responses.json'; // ← Pfad anpassen
const responses = JSON.parse(fs.readFileSync(responsesPath, 'utf8'));

// Alle Einträge für diesen Survey
const entries = responses.filter(r => r.survey_id === SURVEY_ID);
if (entries.length === 0) throw new Error('Keine Antworten für diese Survey-ID gefunden.');

// Für jede Antwort einen Prompt-Block und ein Output-Item erzeugen
return entries.map(data => {
  const dimLines = Object.entries(data.scores.dimensionen)
    .map(([dim, score]) => `- ${dim}: ${score}/100`)
    .join('\n');

  const answerLines = Object.entries(data.answers)
    .map(([key, val]) => `- ${key.toUpperCase()}: ${val.text} (${val.value}/100)`)
    .join('\n');

  const prompt = `Du bist ein erfahrener KI-Berater für KMUs. Analysiere das folgende KI-Readiness Assessment und erstelle eine prägnante, konstruktive Reflexion.

UNTERNEHMEN: ${data.unternehmen}
ANSPRECHPARTNER: ${data.name}${data.funktion ? `, ${data.funktion}` : ''}
DATUM: ${data.datum}

GESAMTSCORE: ${data.scores.gesamt}/100
SCORES NACH DIMENSION:
${dimLines}

EINZELNE ANTWORTEN:
${answerLines}

Erstelle eine strukturierte Reflexion mit exakt diesem Format – prägnant, direkt, ohne Floskeln:

**GESAMTBILD (2–3 Sätze)**
[Kurze Charakterisierung der aktuellen KI-Reife]

**STÄRKEN**
• [Stärke 1]
• [Stärke 2]
• [ggf. Stärke 3]

**HAUPTLÜCKEN**
• [Lücke 1]
• [Lücke 2]
• [ggf. Lücke 3]

**TOP 3 EMPFEHLUNGEN FÜR DEN WORKSHOP**
1. [Konkrete Empfehlung mit Begründung]
2. [Konkrete Empfehlung mit Begründung]
3. [Konkrete Empfehlung mit Begründung]

Wichtig: Sei spezifisch und beziehe dich auf die tatsächlichen Antworten. Vermeide generische Aussagen.`;

  return {
    json: {
      ...data,
      dimLines,
      answerLines,
      claudePrompt: prompt,
    }
  };
});
```

### Node 3: HTTP Request – Claude API

| Feld | Wert |
|---|---|
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |

**Headers:**
```
x-api-key: {{ $vars.ANTHROPIC_API_KEY }}
anthropic-version: 2023-06-01
content-type: application/json
```

**Body (JSON):**
```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "{{ $json.claudePrompt }}"
    }
  ]
}
```

> Der Node iteriert über alle Items aus Node 2 (ein Item pro Teilnehmer). n8n führt den HTTP Request für jedes Item einzeln aus.

### Node 4: Microsoft Teams

Nutzt den bestehenden Teams-Node in n8n.

**Channel:** z. B. `KI-Workshop – Assessments`

**Message (Markdown):**

```
## KI-Readiness Assessment – {{ $('Code – Filtern & Prompt').item.json.unternehmen }}

**Datum:** {{ $('Code – Filtern & Prompt').item.json.datum }}
**Ansprechpartner:** {{ $('Code – Filtern & Prompt').item.json.name }}{{ $('Code – Filtern & Prompt').item.json.funktion ? ', ' + $('Code – Filtern & Prompt').item.json.funktion : '' }}

---

**Gesamtscore: {{ $('Code – Filtern & Prompt').item.json.scores.gesamt }}/100**

{{ $('Code – Filtern & Prompt').item.json.dimLines }}

---

**KI-Reflexion:**

{{ $json.content[0].text }}
```

> `$json.content[0].text` enthält die Claude-Antwort aus Node 3.

---

## Deployment-Checkliste

### n8n Server
- [ ] Datenverzeichnis anlegen und Schreibrechte für n8n-Prozess setzen (Pfad in Code-Nodes eintragen)
- [ ] `surveys.json` mit erster Survey-UUID anlegen
- [ ] `responses.json` als leeres Array `[]` anlegen
- [ ] `ANTHROPIC_API_KEY` als n8n Variable hinterlegen (Settings → Variables)
- [ ] Teams-Credential in n8n konfigurieren (falls noch nicht geschehen)
- [ ] Workflow 1 importieren, Webhook-URL notieren, Workflow aktivieren
- [ ] Workflow 2 importieren (bleibt inaktiv – wird manuell ausgelöst)

### GitHub Pages
- [ ] Repo: `https://github.com/bridge-brain/survey-workflows`
- [ ] HTML-Vorlagen und `CNAME` ins Repo pushen
- [ ] GitHub Pages aktivieren: Settings → Pages → Branch: `main`, Folder: `/ (root)`
- [ ] DNS: CNAME-Eintrag `survey` → `bridge-brain.github.io` beim DNS-Provider setzen
- [ ] Nach DNS-Propagation (~15 min): `https://survey.bridge-brain.ai` aufrufen

### Erster Survey
- [ ] Workflow 3 importieren und aktivieren, Formular-URL notieren
- [ ] Im Formular: Beschreibung, Kommentar und Vorlage ausfüllen → Survey wird automatisch angelegt
- [ ] Test-Link aus der Formular-Antwort kopieren: `https://survey.bridge-brain.ai/<vorlage>.html?survey=<UUID>`
- [ ] Formular ausfüllen und absenden
- [ ] `responses.json` auf dem Server prüfen
- [ ] Workflow 2 manuell mit der Survey-UUID triggern
- [ ] Teams-Nachricht prüfen
