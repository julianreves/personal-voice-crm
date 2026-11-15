> **Historical document.** This is the first full design, written before the build. Several decisions were later revised (see [design-decisions.md](design-decisions.md)): transcription moved from self-hosted whisper.cpp to the OpenAI transcription API, and the workflow was split into a thin first slice plus sub-workflows. Where the two disagree, `design-decisions.md` wins.

# Personal Voice CRM -- Full Technical Specification

## Project Overview

A personal CRM system that allows the user to record voice notes via Telegram, which are automatically transcribed, parsed by AI, and stored in a structured Google Sheets database. The system also supports natural language text queries, calendar integration, and proactive daily digest notifications. All orchestration runs on a self-hosted n8n instance on a Mac Mini.

---

## Architecture Summary

```
Telegram Bot (single interface for all interactions)
        |
        v
n8n (self-hosted on Mac Mini -- orchestration layer)
        |
        +--> whisper.cpp (self-hosted on Mac Mini -- local transcription, zero cost)
        +--> OpenAI GPT-4o API (cloud -- parsing, intent classification, response composition)
        +--> Google Sheets API (cloud -- CRM database)
        +--> Google Calendar API (cloud -- event creation and management)
        +--> Telegram Bot API (cloud -- send responses and digests back to user)
```

---

## Monthly Cost

| Component | Cost |
|---|---|
| n8n (self-hosted) | $0 |
| Telegram Bot API | $0 |
| whisper.cpp (local) | $0 |
| Google Sheets | $0 |
| Google Calendar | $0 |
| OpenAI GPT-4o API | $1 to $4 |
| **Total** | **$1 to $4/mo** |

---

## Components in Detail

### 1. Telegram Bot

The single user interface for the entire system. The user interacts with a private Telegram bot by sending voice messages or text messages. The bot is created via Telegram's @BotFather.

**Setup steps:**
1. Open Telegram, search for @BotFather
2. Send `/newbot`
3. Choose a display name (e.g., "My CRM Assistant")
4. Choose a bot username (e.g., `my_crm_bot`)
5. Save the bot token provided by BotFather
6. Note your personal Telegram user ID (can be obtained by messaging @userinfobot)

**Security requirement:** Only the owner's Telegram user ID should be allowed to interact with the bot. All other messages must be silently ignored. This is enforced in n8n as the first node after the Telegram Trigger.

**Interaction types the bot must handle:**

| Input Type | Example | Expected Behavior |
|---|---|---|
| Voice message (new contact) | "Met Sofia at the design summit..." | Transcribe, parse, create new row in Sheets, confirm via Telegram |
| Voice message (update contact) | "Saw Marcus again at Barcelona..." | Transcribe, parse, find existing row, append to interaction log, confirm |
| Voice message (with meeting intent) | "...I want to meet for lunch in two weeks" | All of the above + create Google Calendar event with CRM brief |
| Text query (geo search) | "Who do I know in Sevilla?" | Search Sheets by city, return formatted results |
| Text query (pending actions) | "What follow-ups do I have?" | Filter Sheets for follow-up needed, return list |
| Text query (neglected contacts) | "Who haven't I talked to in 3 months?" | Filter by last seen date, return list |
| Text query (schedule event) | "Schedule coffee with Elena next Thursday" | Find contact, create Calendar event with CRM brief, confirm |
| Text query (update contact) | "Marcus just got promoted to VP" | Find contact, update role field, confirm |
| Text query (search by name) | "What do I know about Marcus?" | Return full contact record |
| Text query (search by tag) | "Who do I know in paid media?" | Filter by tags, return matches |
| Text query (general) | "How many contacts do I have?" | Fetch all rows, count, respond |

---

### 2. whisper.cpp (Local Transcription)

A C++ port of OpenAI's Whisper model running locally on the Mac Mini with Apple Silicon Metal acceleration. Exposed as a local HTTP API that n8n calls.

**Installation:**

```bash
brew install whisper-cpp
whisper-cpp --download-model small
brew install ffmpeg  # Required for audio format conversion
```

**Local API server (Python + Flask):**

```python
from flask import Flask, request, jsonify
import subprocess
import tempfile
import os

app = Flask(__name__)

@app.route('/transcribe', methods=['POST'])
def transcribe():
    audio = request.files['file']
    
    # Save to temp file
    temp_path = tempfile.mktemp(suffix='.ogg')
    audio.save(temp_path)
    
    # Convert to WAV (whisper.cpp requires 16kHz mono WAV)
    wav_path = temp_path.replace('.ogg', '.wav')
    subprocess.run(['ffmpeg', '-i', temp_path, '-ar', '16000', 
                    '-ac', '1', wav_path], capture_output=True)
    
    # Transcribe locally
    result = subprocess.run(
        ['whisper-cpp', '--model', 'small', '--output-txt', 
         '--file', wav_path],
        capture_output=True, text=True
    )
    
    # Read output
    txt_path = wav_path + '.txt'
    with open(txt_path, 'r') as f:
        transcript = f.read().strip()
    
    # Cleanup
    for p in [temp_path, wav_path, txt_path]:
        if os.path.exists(p):
            os.remove(p)
    
    return jsonify({'text': transcript})

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5111)
```

**Endpoint:** `http://localhost:5111/transcribe`
**Method:** POST
**Content-Type:** multipart/form-data
**Field name:** `file`
**Response format:** `{"text": "transcribed text here"}`

**Notes:**
- Telegram voice messages are in OGG Opus format. The server converts them to 16kHz mono WAV via ffmpeg before passing to whisper.cpp.
- The "small" model provides a good balance of speed and accuracy. For higher accuracy at slower speed, use "medium". For faster transcription at lower accuracy, use "base".
- The server must be running whenever n8n needs to process voice notes. Consider running it as a launchd service on the Mac Mini.

---

### 3. OpenAI GPT-4o API

The intelligence layer. Used for three distinct tasks, each with its own system prompt.

**API endpoint:** `https://api.openai.com/v1/chat/completions`
**Model:** `gpt-4o`
**Authentication:** Bearer token in Authorization header

#### Task A: Voice Note Parsing (Contact Extraction)

**System prompt:**

```
You are a personal CRM assistant. You receive voice note transcripts and extract structured contact information.

Today's date is {{TODAY_DATE}}.

Return ONLY valid JSON with these fields:

{
  "full_name": "string or null",
  "date_met": "YYYY-MM-DD, infer from context or use today's date",
  "how_met": "string or null",
  "company": "string or null",
  "role": "string or null",
  "what_they_do": "string or null",
  "city": "string or null -- where this person is based, if mentioned",
  "birthday": "YYYY-MM-DD or null",
  "key_notes": "string -- key details, personality, interests, offers made",
  "follow_up_needed": true or false,
  "follow_up_action": "string or null",
  "reminder_date": "YYYY-MM-DD or null -- when to follow up",
  "tags": ["array", "of", "relevant", "tags"],
  "meeting_intent": true or false,
  "meeting_type": "coffee / lunch / call / dinner / meeting or null",
  "meeting_timeframe": "string -- 'next Wednesday', 'in 2 weeks', or null",
  "meeting_purpose": "string or null",
  "is_update": true or false,
  "confidence": "high / medium / low",
  "raw_transcript": "the original transcript verbatim"
}

Rules:
- Never invent information not present in the transcript.
- If a field is not mentioned, return null.
- If the transcript mentions someone already known (references prior context, says "again", "ran into", "caught up with"), set is_update to true.
- Confidence should be "low" if the name is unclear or ambiguous.
- For meeting_timeframe, preserve the natural language. Do not convert to a date. The downstream system handles date resolution.
- Tags should capture industry, relationship type, topic areas, and location when relevant.
```

**User message content:** The raw transcript from whisper.cpp.

**Expected response:** A JSON object with all fields listed above.

#### Task B: Text Message Intent Classification

**System prompt:**

```
You are a personal CRM assistant receiving a text message from the user. Classify the intent and extract parameters.

Today's date is {{TODAY_DATE}}.

Return ONLY valid JSON:

{
  "intent": "query | schedule | update | general",
  "sub_type": "search_by_city | search_by_name | search_by_tag | pending_followups | neglected_contacts | create_event | update_contact | count_contacts | general_question",
  "parameters": {
    "contact_name": "string or null",
    "city": "string or null",
    "tag": "string or null",
    "event_date": "string or null -- natural language like 'next Thursday'",
    "event_time": "string or null -- like '1pm', '14:00', or null for default",
    "event_type": "coffee | lunch | dinner | call | meeting | null",
    "duration_minutes": "integer or null -- default 60 if not specified",
    "update_field": "string or null",
    "update_value": "string or null",
    "time_range": "string or null -- for queries like 'in the last 3 months'"
  },
  "raw_message": "the original message verbatim"
}

Intent definitions:
- "query": user wants to retrieve or search information from the CRM
- "schedule": user wants to create a calendar event
- "update": user wants to modify an existing contact's information
- "general": anything else (greetings, meta-questions about the system, etc.)

Examples:
- "Who do I know in Sevilla?" -> intent: query, sub_type: search_by_city, city: "Sevilla"
- "Schedule coffee with Marcus next Wednesday" -> intent: schedule, sub_type: create_event, contact_name: "Marcus", event_date: "next Wednesday", event_type: "coffee"
- "Marcus just got promoted to VP" -> intent: update, sub_type: update_contact, contact_name: "Marcus", update_field: "role", update_value: "VP"
- "What follow-ups do I have?" -> intent: query, sub_type: pending_followups
- "Who haven't I talked to in 3 months?" -> intent: query, sub_type: neglected_contacts, time_range: "3 months"
- "Tell me about Elena Morales" -> intent: query, sub_type: search_by_name, contact_name: "Elena Morales"
- "Who works in paid media?" -> intent: query, sub_type: search_by_tag, tag: "paid media"
- "How many contacts do I have?" -> intent: query, sub_type: count_contacts
```

**User message content:** The raw text message from Telegram.

#### Task C: Response Composition

**System prompt:**

```
You are a personal CRM assistant. You receive structured CRM data and compose a natural, concise response for the user via Telegram.

Rules:
- Be brief but informative.
- Use bold for contact names.
- Include the most relevant details: role, company, last interaction, pending actions.
- If there are no results, say so clearly and suggest alternatives (e.g., "No contacts in Sevilla. Want me to search nearby cities?").
- For confirmations (new contact created, event scheduled), summarize what was done and ask if anything needs adjustment.
- Do not use markdown headers. Use simple bold text and line breaks suitable for Telegram.
```

**User message content:** The CRM data (contact records, filter results, etc.) formatted as JSON, plus context about what the user asked for.

#### Task D: Daily Digest Composition

**System prompt:**

```
You are a personal CRM assistant composing a morning briefing for the user via Telegram.

You will receive two lists:
1. Contacts with upcoming birthdays (within the next 7 days)
2. Contacts with pending follow-ups due today or overdue

Compose a brief, actionable digest. For each contact, include:
- Their name (bold)
- The relevant date (birthday or follow-up due date)
- The most recent interaction note for context
- A suggested action if applicable

Keep it concise. This is a daily message the user reads quickly with their morning coffee.
If both lists are empty, send a short "All clear" message.
```

#### Task E: Calendar Event Description Generation

**System prompt:**

```
You are a personal CRM assistant. You receive a contact's full CRM record and generate a formatted event description for Google Calendar.

Use this exact format:

CONTACT BRIEF -- [Full Name]
Role: [role] at [company]
Last seen: [last interaction context and date]
How we met: [how_met]

Recent notes:
[Most recent entries from interaction log]

Pending actions:
[Any follow-up actions]

Tags: [tags]

Keep it clean, scannable, and useful for a quick glance before the meeting.
```

---

### 4. Google Sheets CRM (Database)

A single Google Spreadsheet acting as the CRM database. One sheet named "Contacts".

**Sheet name:** Contacts

**Column schema (exact column headers, in order):**

| Column | Header | Data Type | Description |
|---|---|---|---|
| A | Full Name | Text | Primary identifier. First and last name. |
| B | Date First Met | Date (YYYY-MM-DD) | Date of the first interaction. Set once, never overwritten. |
| C | Last Seen Date | Date (YYYY-MM-DD) | Updated every time a new interaction is logged. |
| D | How Met | Text | Context of the first meeting. Set once, never overwritten. |
| E | Company | Text | Current company or organization. Can be updated. |
| F | Role | Text | Current job title or role. Can be updated. |
| G | What They Do | Text | Description of what they do professionally. |
| H | City | Text | Where this person is based. Can be updated. |
| I | Birthday | Date (YYYY-MM-DD) | Birthday. Can be updated. |
| J | Interaction Log | Text (long) | Append-only running history. Each entry is timestamped. Format: `[YYYY-MM-DD] Note text here.` Entries separated by newlines. Never overwritten, only appended to. |
| K | Follow-up Needed | Text | "TRUE" or "FALSE". Updated on each interaction. |
| L | Follow-up Action | Text | Current pending action. Overwritten when a new follow-up is set. Cleared when follow-up is completed. |
| M | Reminder Date | Date (YYYY-MM-DD) | When the follow-up reminder should trigger. |
| N | Tags | Text | Comma-separated tags. New tags are appended, existing tags preserved. Example: "agency, growth, paid media, DTC" |
| O | Confidence | Text | "high", "medium", or "low". From the GPT-4o parser. |
| P | Raw Transcripts | Text (long) | All raw transcripts appended with timestamps. Format: `[YYYY-MM-DD] Transcript text.` |

**Important behaviors:**
- When CREATING a new contact: fill all columns. Date First Met and Last Seen Date are the same. Interaction Log gets its first entry.
- When UPDATING an existing contact: ONLY update Last Seen Date, Interaction Log (append), Follow-up Needed, Follow-up Action, Reminder Date, and Tags (append new tags). Also update Company, Role, City, Birthday IF the new voice note contains updated information for those fields. NEVER overwrite Date First Met, How Met, or existing Interaction Log entries.
- The Interaction Log append format: take the existing cell content, add a newline, then add the new timestamped entry.
- Tags append format: take the existing cell content, check for duplicates, add new comma-separated tags that are not already present.

**Google Sheets API setup:**
- Create a Google Cloud project
- Enable the Google Sheets API and Google Calendar API
- Create OAuth2 credentials (or a service account)
- Share the spreadsheet with the service account email if using a service account
- In n8n, configure the Google Sheets credential with the OAuth2 or service account details

---

### 5. Google Calendar

Used to create and update calendar events with CRM context in the event description.

**Calendar:** The user's primary Google Calendar (default).

**Event structure:**

| Field | Source |
|---|---|
| Title | "[Meeting type] with [Full Name]" -- e.g., "Coffee with Marcus Chen" or "Lunch with Elena Morales" |
| Start time | Resolved from the natural language timeframe by GPT-4o or from the user's explicit instruction |
| Duration | Default 60 minutes unless specified |
| Description | Generated by GPT-4o Task E (Calendar Event Description Generation) using the contact's full CRM record |
| Location | Not set unless explicitly mentioned |

**When events are created:**
1. Automatically when a voice note contains a meeting intent (meeting_intent = true)
2. On demand when the user sends a text command like "Schedule coffee with Marcus next Thursday"

**Date resolution:** When the meeting_timeframe is natural language (e.g., "next Wednesday", "in two weeks"), GPT-4o resolves it to a specific YYYY-MM-DD date using today's date as reference. If the time is not specified, default to:
- Coffee: 10:00 AM
- Lunch: 1:00 PM
- Dinner: 8:00 PM
- Call: 10:00 AM
- Meeting: 10:00 AM

---

### 6. n8n (Self-Hosted Orchestration)

Runs on the Mac Mini. Two workflows total.

---

## n8n Workflow 1: Main Bot

**Trigger:** Telegram Trigger node (receives all messages sent to the bot)
**Type:** Always active (webhook-based)

### Full Node Sequence:

```
Node 1: Telegram Trigger
  |
Node 2: Code Node (Security Gate)
  |
Node 3: Switch Node (Voice or Text?)
  |
  +-- Voice Path ----------------------------------------+
  |                                                       |
  Node 4: HTTP Request (Download audio from Telegram)     |
  |                                                       |
  Node 5: HTTP Request (Send to local Whisper)            |
  |                                                       |
  Node 6: HTTP Request (Send transcript to GPT-4o         |
  |        Task A: Contact Parsing)                       |
  |                                                       |
  Node 7: JSON Parse (Extract structured fields)          |
  |                                                       |
  Node 8: Google Sheets (Search by Full Name)             |
  |                                                       |
  Node 9: IF Node (Contact exists?)                       |
  |        |                                              |
  |     Yes |          No                                 |
  |        |            |                                 |
  |   Node 10a:    Node 10b:                              |
  |   Google        Google Sheets                         |
  |   Sheets        (Add new row)                         |
  |   (Update row,                                        |
  |    append to                                          |
  |    Interaction                                        |
  |    Log)                                               |
  |        |            |                                 |
  |        +-----+------+                                 |
  |              |                                        |
  |         Node 11: IF Node (Confidence = low?)          |
  |              |        |                               |
  |           Yes |     No |                              |
  |              |        |                               |
  |   Node 11a: Add   Node 12: IF Node                   |
  |   "NEEDS REVIEW"  (Meeting intent?)                   |
  |   to notes            |        |                      |
  |        |           Yes |     No |                     |
  |        |              |        |                      |
  |        |         Node 13:      |                      |
  |        |         Google Sheets |                      |
  |        |         (Fetch full   |                      |
  |        |          record)      |                      |
  |        |              |        |                      |
  |        |         Node 14:      |                      |
  |        |         HTTP Request  |                      |
  |        |         (GPT-4o       |                      |
  |        |          Task E:      |                      |
  |        |          generate     |                      |
  |        |          event desc)  |                      |
  |        |              |        |                      |
  |        |         Node 15:      |                      |
  |        |         Google        |                      |
  |        |         Calendar      |                      |
  |        |         (Create       |                      |
  |        |          event)       |                      |
  |        |              |        |                      |
  |        +-----+--------+-------+                       |
  |              |                                        |
  |         Node 16: HTTP Request                         |
  |         (GPT-4o Task C: compose                       |
  |          confirmation message)                        |
  |              |                                        |
  |         Node 17: Telegram Node                        |
  |         (Send confirmation to user)                   |
  |                                                       |
  +-- Text Path -----------------------------------------+
  |                                                       |
  Node 18: HTTP Request (Send text to GPT-4o              |
  |         Task B: Intent Classification)                |
  |                                                       |
  Node 19: JSON Parse (Extract intent + parameters)       |
  |                                                       |
  Node 20: Switch Node (Intent type?)                     |
  |                                                       |
  +-- intent: query ------------------------------------+ |
  |                                                     | |
  |   Node 21: Switch Node (Sub-type?)                  | |
  |   |                                                 | |
  |   +-- search_by_name:                              | |
  |   |   Node 22a: Google Sheets (Search by name)     | |
  |   |                                                 | |
  |   +-- search_by_city:                              | |
  |   |   Node 22b: Google Sheets (Get all rows)       | |
  |   |   Node 22b2: Code Node (Filter by city)        | |
  |   |                                                 | |
  |   +-- search_by_tag:                               | |
  |   |   Node 22c: Google Sheets (Get all rows)       | |
  |   |   Node 22c2: Code Node (Filter by tag)         | |
  |   |                                                 | |
  |   +-- pending_followups:                           | |
  |   |   Node 22d: Google Sheets (Get all rows)       | |
  |   |   Node 22d2: Code Node (Filter follow-ups)     | |
  |   |                                                 | |
  |   +-- neglected_contacts:                          | |
  |   |   Node 22e: Google Sheets (Get all rows)       | |
  |   |   Node 22e2: Code Node (Filter by last seen)   | |
  |   |                                                 | |
  |   +-- count_contacts:                              | |
  |       Node 22f: Google Sheets (Get all rows)       | |
  |       Node 22f2: Code Node (Count)                 | |
  |   |                                                 | |
  |   All query sub-types converge to:                  | |
  |   Node 23: HTTP Request (GPT-4o Task C:            | |
  |            compose natural language response)       | |
  |   Node 24: Telegram Node (Send response to user)   | |
  |                                                     | |
  +-- intent: schedule ---------------------------------+ |
  |                                                     | |
  |   Node 25: Google Sheets (Search contact by name)   | |
  |   Node 26: IF (Contact found?)                     | |
  |   |     |                                           | |
  |   | Yes |  No: Node 26a: Telegram (Send             | |
  |   |     |       "Contact not found" message)        | |
  |   |     |                                           | |
  |   Node 27: HTTP Request (GPT-4o Task E:            | |
  |            generate event description)              | |
  |   Node 28: Google Calendar (Create event)           | |
  |   Node 29: Telegram (Send confirmation)             | |
  |                                                     | |
  +-- intent: update -----------------------------------+ |
  |                                                     | |
  |   Node 30: Google Sheets (Search contact by name)   | |
  |   Node 31: IF (Contact found?)                     | |
  |   |     |                                           | |
  |   | Yes |  No: Node 31a: Telegram (Send             | |
  |   |     |       "Contact not found" message)        | |
  |   |     |                                           | |
  |   Node 32: Google Sheets (Update specific field)    | |
  |   Node 33: Telegram (Send confirmation)             | |
  |                                                     | |
  +-- intent: general ----------------------------------+ |
  |                                                       |
  |   Node 34: HTTP Request (GPT-4o: answer general      |
  |            question with system context)              |
  |   Node 35: Telegram (Send response)                  |
  |                                                       |
  +-------------------------------------------------------+
```

### Node Configuration Details:

**Node 1 -- Telegram Trigger**
- Credential: Telegram Bot API (bot token from BotFather)
- Updates: Message
- Additional fields: Download any file content attached to the message

**Node 2 -- Code Node (Security Gate)**
```javascript
const allowedUserId = REPLACE_WITH_YOUR_TELEGRAM_USER_ID;
const senderId = $input.first().json.message.from.id;

if (senderId !== allowedUserId) {
  return []; // Silently drop. Workflow stops.
}

return $input.all();
```

**Node 3 -- Switch Node (Voice or Text)**
- Condition 1 (Voice): `{{$json.message.voice}}` is not empty -> route to Voice Path
- Condition 2 (Text): `{{$json.message.text}}` is not empty -> route to Text Path

**Node 4 -- HTTP Request (Download audio from Telegram)**
- Method: GET
- URL: `https://api.telegram.org/bot{{TELEGRAM_BOT_TOKEN}}/getFile?file_id={{$json.message.voice.file_id}}`
- Then a second HTTP Request to download the actual file: `https://api.telegram.org/file/bot{{TELEGRAM_BOT_TOKEN}}/{{file_path}}`
- Response format: Binary/File

**Node 5 -- HTTP Request (Local Whisper Transcription)**
- Method: POST
- URL: `http://localhost:5111/transcribe`
- Content-Type: multipart/form-data
- Body: file = binary data from Node 4
- Response format: JSON

**Node 6 -- HTTP Request (GPT-4o Task A: Contact Parsing)**
- Method: POST
- URL: `https://api.openai.com/v1/chat/completions`
- Authentication: Bearer token (OpenAI API key)
- Headers: Content-Type: application/json
- Body (JSON):
```json
{
  "model": "gpt-4o",
  "temperature": 0.1,
  "response_format": { "type": "json_object" },
  "messages": [
    {
      "role": "system",
      "content": "PASTE TASK A SYSTEM PROMPT HERE (with {{TODAY_DATE}} resolved)"
    },
    {
      "role": "user",
      "content": "{{transcript_text_from_whisper}}"
    }
  ]
}
```

**Node 7 -- JSON Parse**
- Parse the `choices[0].message.content` field from GPT-4o response into individual fields

**Node 8 -- Google Sheets (Search)**
- Operation: Search rows
- Spreadsheet: [CRM Spreadsheet ID]
- Sheet: Contacts
- Search column: Full Name
- Search value: `{{parsed_full_name}}`

**Node 9 -- IF Node**
- Condition: Google Sheets search returned 0 results -> No (create new) / 1+ results -> Yes (update)

**Node 10a -- Google Sheets (Update Existing Contact)**
- Operation: Update row
- Row number: from the search result
- Fields to update:
  - Last Seen Date: today's date
  - Interaction Log: existing value + newline + `[{{TODAY_DATE}}] {{key_notes}}`
  - Follow-up Needed: from parsed data
  - Follow-up Action: from parsed data (if not null)
  - Reminder Date: from parsed data (if not null)
  - Tags: existing value + new tags (deduplicated)
  - Company: from parsed data (only if not null and different from existing)
  - Role: from parsed data (only if not null and different from existing)
  - City: from parsed data (only if not null and different from existing)
  - Birthday: from parsed data (only if not null and existing is empty)
  - Raw Transcripts: existing value + newline + `[{{TODAY_DATE}}] {{raw_transcript}}`

**Node 10b -- Google Sheets (Create New Contact)**
- Operation: Append row
- All fields populated from parsed JSON
- Interaction Log initial value: `[{{TODAY_DATE}}] {{key_notes}}`
- Raw Transcripts initial value: `[{{TODAY_DATE}}] {{raw_transcript}}`

**Node 11 -- IF Node (Low Confidence Check)**
- Condition: confidence equals "low"
- If yes: prepend "[NEEDS REVIEW] " to the key_notes or add to a separate flag

**Node 12 -- IF Node (Meeting Intent)**
- Condition: meeting_intent equals true

**Node 13 -- Google Sheets (Fetch Full Record)**
- Operation: Get row by row number (from the create or update result)
- Returns the complete contact record for event description generation

**Node 14 -- HTTP Request (GPT-4o Task E: Event Description)**
- Same API call pattern as Node 6 but with Task E system prompt
- User message contains the full contact record as JSON

**Node 15 -- Google Calendar (Create Event)**
- Operation: Create event
- Calendar: Primary
- Title: `{{meeting_type}} with {{full_name}}` (capitalized)
- Start date/time: resolved from meeting_timeframe using today's date as reference. GPT-4o should resolve this in its response. If ambiguous, default to 10:00 AM.
- Duration: 60 minutes default
- Description: output from Node 14

**Node 16 -- HTTP Request (GPT-4o Task C: Compose Confirmation)**
- User message contains: what was done (created/updated contact, scheduled event), the parsed data summary
- GPT-4o composes a concise Telegram-friendly confirmation

**Node 17 -- Telegram Node (Send Message)**
- Operation: Send message
- Chat ID: `{{$json.message.chat.id}}` (from the original trigger)
- Text: output from Node 16
- Parse mode: Markdown

**Nodes 18-19 -- Text Path: Intent Classification**
- Same HTTP Request pattern, using Task B system prompt
- User message: the raw text from `{{$json.message.text}}`

**Nodes 22b2, 22c2, 22d2, 22e2 -- Code Nodes for Filtering**

Filter by city:
```javascript
const city = $input.first().json.parameters.city.toLowerCase();
const contacts = $('Google Sheets Node').all();
const matches = contacts.filter(c => 
  c.json.City && c.json.City.toLowerCase().includes(city)
);
return matches.length > 0 ? matches : [{ json: { no_results: true } }];
```

Filter by tag:
```javascript
const tag = $input.first().json.parameters.tag.toLowerCase();
const contacts = $('Google Sheets Node').all();
const matches = contacts.filter(c => 
  c.json.Tags && c.json.Tags.toLowerCase().includes(tag)
);
return matches.length > 0 ? matches : [{ json: { no_results: true } }];
```

Filter pending follow-ups:
```javascript
const today = new Date().toISOString().split('T')[0];
const contacts = $('Google Sheets Node').all();
const matches = contacts.filter(c => 
  c.json.Follow_up_Needed === 'TRUE' && 
  (!c.json.Reminder_Date || c.json.Reminder_Date <= today)
);
return matches.length > 0 ? matches : [{ json: { no_results: true } }];
```

Filter neglected contacts:
```javascript
const params = $input.first().json.parameters;
const months = parseInt(params.time_range) || 3;
const cutoff = new Date();
cutoff.setMonth(cutoff.getMonth() - months);
const cutoffStr = cutoff.toISOString().split('T')[0];

const contacts = $('Google Sheets Node').all();
const matches = contacts.filter(c => 
  c.json.Last_Seen_Date && c.json.Last_Seen_Date < cutoffStr
);
return matches.length > 0 ? matches : [{ json: { no_results: true } }];
```

---

## n8n Workflow 2: Daily Digest

**Trigger:** Cron/Schedule node
**Schedule:** Every day at 8:00 AM (user's local timezone)

### Full Node Sequence:

```
Node 1: Schedule Trigger (daily at 8:00 AM)
  |
Node 2: Google Sheets (Get all rows from Contacts sheet)
  |
Node 3: Code Node (Filter birthdays within 7 days + overdue follow-ups)
  |
Node 4: IF Node (Any matches at all?)
  |        |
  | No     | Yes
  |        |
  | Node 4a: Telegram     Node 5: HTTP Request
  | (Send "All clear      (GPT-4o Task D:
  |  for today")           compose digest)
  |                            |
  |                       Node 6: Telegram Node
  |                       (Send digest message)
```

**Node 3 -- Code Node (Filtering Logic):**
```javascript
const today = new Date();
const sevenDaysOut = new Date(today);
sevenDaysOut.setDate(today.getDate() + 7);
const todayStr = today.toISOString().split('T')[0];

const contacts = $input.all();

const birthdayMatches = contacts.filter(item => {
  if (!item.json.Birthday) return false;
  const bday = new Date(item.json.Birthday);
  bday.setFullYear(today.getFullYear());
  return bday >= today && bday <= sevenDaysOut;
}).map(item => ({
  name: item.json['Full Name'],
  birthday: item.json.Birthday,
  company: item.json.Company,
  role: item.json.Role,
  last_note: item.json['Interaction Log'] ? 
    item.json['Interaction Log'].split('\n').pop() : 'No notes'
}));

const followUpMatches = contacts.filter(item => {
  return item.json['Follow-up Needed'] === 'TRUE' && 
    (!item.json['Reminder Date'] || item.json['Reminder Date'] <= todayStr);
}).map(item => ({
  name: item.json['Full Name'],
  action: item.json['Follow-up Action'],
  reminder_date: item.json['Reminder Date'],
  company: item.json.Company,
  last_note: item.json['Interaction Log'] ? 
    item.json['Interaction Log'].split('\n').pop() : 'No notes'
}));

return [{
  json: {
    birthdayMatches,
    followUpMatches,
    hasMatches: birthdayMatches.length > 0 || followUpMatches.length > 0
  }
}];
```

**Node 5 -- HTTP Request (GPT-4o Task D):**
- User message contains the birthdayMatches and followUpMatches arrays as JSON
- GPT-4o composes a natural-language digest

**Node 6 -- Telegram Node:**
- Operation: Send message
- Chat ID: The user's chat ID with the bot (hardcoded or stored as n8n credential/variable)
- Parse mode: Markdown

---

## Contact Disambiguation Logic

When the Google Sheets search by name returns multiple results (e.g., two people named "Marcus"), the system must handle this gracefully.

**In the voice path (Workflow 1):**
- If multiple matches found, use GPT-4o to compare the parsed context (company, city, tags, topic) against each matching record
- Send the parsed data + all matching records to GPT-4o with this prompt addition:

```
Multiple contacts match the name "[name]". Based on the context in the voice note, 
determine which contact is being referenced. If you cannot determine with confidence, 
return {"disambiguation_needed": true, "candidates": [list of names with companies]}.
```

- If disambiguation fails, send a Telegram message asking the user to clarify:
  "I found 2 contacts named Marcus: Marcus Chen (Ascend Agency) and Marcus Rivera (Netflix). Which one?"

**In the text path:**
- Same logic applies. If the user says "Tell me about Marcus" and there are two, ask for clarification via Telegram.

---

## Error Handling

Every HTTP Request node should have error handling configured:

| Error Type | Handling |
|---|---|
| Whisper server unreachable | Telegram message: "Transcription service is offline. Voice note saved, will retry." Save audio file reference for manual retry. |
| GPT-4o API error (rate limit, timeout) | Retry once after 5 seconds. If still failing, Telegram message: "AI service temporarily unavailable. Try again in a minute." |
| Google Sheets API error | Telegram message: "Could not access the database. Please check Google Sheets permissions." |
| Google Calendar API error | Telegram message: "Could not create calendar event. Contact was saved successfully." |
| Unparseable GPT-4o response | Telegram message: "Could not parse the voice note. Please try again or rephrase." |
| Empty voice note / too short | Telegram message: "Voice note was too short to process. Please record at least a few seconds." |

---

## Environment Variables / n8n Credentials Needed

| Variable | Where Used | How to Configure |
|---|---|---|
| TELEGRAM_BOT_TOKEN | Telegram Trigger + Telegram Send nodes | n8n Telegram credential |
| OWNER_TELEGRAM_USER_ID | Security gate Code node | Hardcoded in the Code node |
| OPENAI_API_KEY | All GPT-4o HTTP Request nodes | n8n Header Auth credential or hardcoded in Authorization header |
| GOOGLE_SHEETS_CREDENTIAL | All Google Sheets nodes | n8n Google Sheets OAuth2 credential |
| GOOGLE_CALENDAR_CREDENTIAL | Google Calendar nodes | n8n Google Calendar OAuth2 credential |
| WHISPER_LOCAL_URL | Whisper HTTP Request node | `http://localhost:5111/transcribe` |
| CRM_SPREADSHEET_ID | All Google Sheets nodes | The ID from the Google Sheets URL |
| CRM_SHEET_NAME | All Google Sheets nodes | "Contacts" |

---

## Build Order (Phases)

| Phase | What to Build | Estimated Time | Dependencies |
|---|---|---|---|
| 1 | Create Google Sheets spreadsheet with exact column schema | 30 minutes | None |
| 2 | Create Telegram bot via BotFather | 5 minutes | None |
| 3 | Set up whisper.cpp + Flask server on Mac Mini | 1 to 2 hours | ffmpeg installed |
| 4 | Workflow 1: Voice path only (Nodes 1 through 17), new contacts only | 2 to 3 hours | Phases 1, 2, 3 |
| 5 | Test Phase 4 with 10 real voice notes | 1 day | Phase 4 |
| 6 | Add update logic to voice path (Node 9 branching, Node 10a) | 1 to 2 hours | Phase 5 validated |
| 7 | Add calendar creation to voice path (Nodes 12 through 15) | 1 to 2 hours | Phase 6 |
| 8 | Add text path to Workflow 1 (Nodes 18 through 35) | 2 to 3 hours | Phase 6 |
| 9 | Build Workflow 2: Daily digest | 1 hour | Phase 6 |
| 10 | Add disambiguation logic | 1 hour | Phase 8 |
| 11 | Add error handling to all HTTP Request nodes | 1 hour | All phases |
| 12 | End-to-end testing of all interaction types | 1 day | All phases |

**Total estimated build time: 3 to 4 days of focused work.**

---

## Testing Checklist

After each phase, verify these scenarios work:

**Phase 5 (Basic voice, new contacts):**
- [ ] Voice note with clear name, company, role -> creates correct row
- [ ] Voice note with minimal info (just a name) -> creates row, other fields null
- [ ] Voice note in Spanish -> transcribes and parses correctly
- [ ] Voice note with background noise -> transcribes acceptably
- [ ] Bot responds with confirmation message

**Phase 6 (Update logic):**
- [ ] Voice note about existing contact -> updates row, does not create duplicate
- [ ] Interaction Log is appended, not overwritten
- [ ] Last Seen Date is updated
- [ ] Original How Met and Date First Met are unchanged

**Phase 7 (Calendar):**
- [ ] Voice note with "let's meet next week" -> creates calendar event
- [ ] Event description contains CRM brief
- [ ] Voice note without meeting intent -> no calendar event created

**Phase 8 (Text queries):**
- [ ] "Who do I know in [city]?" -> returns correct contacts
- [ ] "What follow-ups do I have?" -> returns pending list
- [ ] "Schedule coffee with [name] next [day]" -> creates event
- [ ] "[Name] changed jobs to [new role]" -> updates record
- [ ] Query for nonexistent contact -> graceful "not found" message

**Phase 9 (Daily digest):**
- [ ] Digest fires at 8 AM
- [ ] Birthdays within 7 days are included
- [ ] Overdue follow-ups are included
- [ ] Empty days produce an "all clear" message

---
