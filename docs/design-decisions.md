# Design decisions

The original specification ([original-specification.md](original-specification.md)) described the whole system up front. A review pass before building revised it. These decisions override the original where they differ.

## Locked decisions

1. **Transcription = OpenAI `gpt-4o-transcribe` API, not self-hosted whisper.cpp.** This deletes an always-on Flask + launchd + ffmpeg service and its silent-failure modes. The cost is a few cents per note. The API accepts Telegram's `.oga/.ogg` voice files directly, so no audio conversion step is needed.
2. **The interaction log stays a single append-only cell per contact.** Splitting it into a separate Interactions sheet was considered and rejected: update volume is low and the 50,000-character cell limit will not be reached.
3. **Date math is done in code (n8n Code node / expressions), never by the LLM.** Language models are unreliable at calendar arithmetic ("next Wednesday" resolved from the wrong anchor date).
4. **Reliability over cost.** Every capture replies with a confirmation, and a dedicated error-notify workflow pings the owner on any failure, so nothing fails silently. If capture is high-friction or unreliable the tool does not get used, so a few dollars of API spend is the right trade.
5. **Main workflow + sub-workflows.** Reusable pieces (`transcribe`, `parse-contact`, `compose-reply`, `error-notify`) become sub-workflows as the slices land.
6. **Store = Google Sheets**, a single `Contacts` sheet (columns in [../schema/contacts-columns.csv](../schema/contacts-columns.csv); the workflow maps by header name).
7. **Host = n8n on an always-on Mac mini.**
8. **Confirmation messages are a deterministic template, not an LLM call**, for speed and reliability.

## Bugs found in the original specification

Found by reading the spec against how n8n actually works, before any of it was built:

- The Google Sheets column keys were referenced inconsistently between Workflow 1 (underscore keys such as `Follow_up_Needed`, wrong) and Workflow 2 (bracketed headers such as `['Follow-up Needed']`, right).
- The birthday-window matching in the daily digest had edge cases: time of day could exclude a birthday that is *today*, and `setFullYear` breaks across the December → January boundary.
- Letting the LLM resolve relative dates (see decision 3).

## Roadmap (slices, in order)

1. **error-notify sub-workflow**: a reusable workflow that messages the owner on any node failure. Built early so nothing fails silently.
2. **Update existing contact**: search by name; if found, append to the interaction log and raw transcripts, update last-seen date, follow-up fields and tags (deduplicated); update company, role, city or birthday only when new information is present; never overwrite the first-met date.
3. **Calendar events**: when the parsed note has meeting intent (or on a text "schedule…" command), resolve the date in code, generate an event description from the contact record, and create a calendar event with a CRM brief.
4. **Text queries**: intent classification, then search by name / city / tag, pending follow-ups, neglected contacts and counts, answered in natural language.
5. **Daily digest**: an 8 a.m. schedule that sends upcoming birthdays (next 7 days) and overdue follow-ups, with the birthday-window bugs above fixed.
