# restaurant-rag-assistant

A WhatsApp agent for a restaurant that takes table reservations and answers menu
and policy questions, using Claude tool use as the router and a custom RAG
pipeline over the FAQ.

Built in 7 days (August 2026) as a working prototype for a real restaurant. This public
version runs on sample FAQ data and is not deployed.

## What it does

- **Takes reservations** from free-form messages: "mesa para 4 el sábado a las 21"
  becomes a structured booking (date, time, party size, optional name) that n8n
  appends to a Google Sheet.
- **Asks for what's missing** instead of guessing: "quiero reservar una mesa"
  gets a follow-up question, and the booking can arrive across several messages.
- **Answers FAQ questions** (hours, gluten-free options, wifi, parking, payment,
  pets) only from retrieved FAQ fragments; when nothing relevant is found it
  replies that someone from the restaurant will answer, instead of improvising.
- **Resolves relative dates** ("mañana", "el viernes") to `DD/MM/AAAA` against the
  current date, and asks back when the date is genuinely ambiguous.

| The customer writes | The agent |
|---|---|
| "mesa para 4 el sábado a las 21" | calls `crear_reserva`, returns the confirmed booking |
| "tenés wifi?" | calls `consultar_faq`, then answers from the FAQ via RAG |
| "quiero reservar una mesa" | calls no tool: asks for the missing field and waits |

## Tech stack

Python 3.11+ · FastAPI · Pydantic · Claude API (tool use) · Voyage AI embeddings
(`voyage-4`) · NumPy (cosine similarity) · n8n · WhatsApp Cloud API · Google Sheets

## How it works

```
Customer (WhatsApp) → Meta Cloud API → n8n ──→ FastAPI → Claude API (tool use)
                                        │                      │
                                        │                      └→ Voyage AI (RAG)
                                        └──────→ Google Sheets
                                                 (reservations)
```

Full diagram: [`docs/diagrama-arquitectura-secciones-1-3.svg`](docs/diagrama-arquitectura-secciones-1-3.svg).

- n8n receives the Meta webhook, calls FastAPI with `{numero, texto}` and sends
  the reply back through the WhatsApp Cloud API.
- FastAPI exposes a single endpoint, `POST /webhook`, and returns `{respuesta}`
  plus a `reserva` object when a booking was confirmed.
- **FastAPI never touches Google Sheets or Meta.** n8n writes the row. The API
  would work the same behind a web widget or a different spreadsheet.
- Each message gets one Claude call with two tools. The FAQ path adds a second
  call that answers using only the top 3 retrieved fragments.

## Engineering highlights

- **Tool use as the classifier, no keyword matching.** Claude gets
  `crear_reserva` and `consultar_faq` on every message; whichever it calls *is*
  the classification. `consultar_faq` has an empty input schema on purpose: it
  exists only as a signal, and the search runs on the customer's original text.
  Calling no tool is ambiguous (an FAQ question or an incomplete booking look the
  same), so the router checks `crear_reserva` first, then `consultar_faq`
  explicitly, and treats the rest as the "ask for the missing field" path.
- **The similarity threshold was measured, not guessed.** `UMBRAL_SIMILITUD = 0.45`
  comes from `scripts/check_scores.py` (8 questions: 6 rephrased FAQ questions and
  2 off-topic): the worst true match scored 0.5635 and the best false match
  0.3278. The threshold sits at the midpoint, and the comment on the constant in
  `app/rag/search.py` records the measurement.
- **Fail fast on deploy errors, degrade gracefully at runtime.** An empty FAQ file
  or an empty `ANTHROPIC_API_KEY` stops the server at startup (an empty key would
  raise a `TypeError` that escapes the `anthropic.APIError` handler and returns a
  500). Claude and Voyage errors at runtime return distinct fallback messages. A
  dedicated `BusquedaFallidaError` keeps "no relevant results" (an empty list)
  separate from "the search never ran".

Other decisions, briefly:

- The system prompt is rebuilt on every request so "mañana" resolves against the
  real date, not the date the server started.
- Conversation memory is an in-memory dict keyed by phone number, cleared when a
  booking is confirmed so the next booking doesn't inherit the previous party size.

## Project layout

```
app/
├── main.py            # POST /webhook and the 3-way routing
├── claude_client.py   # system prompt, both tool schemas, response parsing
├── conversaciones.py  # in-memory history, keyed by phone number
├── models.py          # incoming payload (Pydantic)
└── rag/
    ├── embeddings.py  # generates and stores the FAQ vectors
    └── search.py      # cosine similarity, top-k, the measured threshold
scripts/
├── check_scores.py    # calibration run behind the 0.45 threshold
└── correr_guiones.py  # runs the test scripts against a live server
n8n/workflow-dia2.json # sanitized export of the n8n workflow
docs/                  # build log, test scripts, architecture diagram
```

## Getting started

Requires an Anthropic API key and a Voyage AI API key (see
[`.env.example`](.env.example)).

```bash
git clone https://github.com/nachixxs/restaurant-rag-assistant.git
cd restaurant-rag-assistant
python -m venv venv
venv\Scripts\activate          # Windows; on macOS/Linux: source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env           # set ANTHROPIC_API_KEY and VOYAGE_API_KEY

python -m app.rag.embeddings   # generates data/faq_embeddings.json
uvicorn app.main:app --reload
```

The API works on its own, without WhatsApp or n8n:

```bash
curl -X POST localhost:8000/webhook \
  -H "Content-Type: application/json" \
  -d '{"numero":"test-1","texto":"mesa para 4 el sábado a las 21"}'
```

The WhatsApp token and phone number ID live in n8n, not in `.env`. The full path
also needs the n8n workflow in `n8n/` published and a Meta app with WhatsApp
configured.

## Tests

18 end-to-end test scripts plus 3 degenerate inputs (empty and whitespace-only
messages), each with input, expected behaviour and actual result, in
[`docs/guiones_testing.md`](docs/guiones_testing.md). `scripts/correr_guiones.py`
runs them against a live server.

- Written the way customers type: lowercase, no accents, typos, emoji, and
  information arriving in pieces (`che tenes algo pa comer si soy celiaco?`, `🍕😋`).
- Verified structurally, not by whether the reply sounds right: exact comparison
  against the fallback constants imported from `app.claude_client`, and presence
  of the `reserva` key to know which tool Claude picked.
- Each script uses its own number so conversation memory can't leak between
  scripts, and messages are spaced 25 s apart to stay under Voyage's free-tier
  rate limit.

## Project status

Working prototype, not production-ready. Findings from the test runs are recorded
with their status (fixed, accepted or open) in
[`docs/guiones_testing.md`](docs/guiones_testing.md). Known limitations:

- **Conversation state is in memory**: a restart drops in-flight conversations.
- **Confirmed bookings can't be modified**: history is cleared on confirmation, so
  "en realidad somos 5" is read as a new booking. Needs a modification tool.
- **Ambiguous messages route inconsistently**: "tienen mesa para cumpleaños, somos
  re grupo grande?" goes to either path across runs.
- **The FAQ is sample data**: `data/faq_embeddings.json` holds six example entries;
  swapping it means editing the list in `app/rag/embeddings.py` and regenerating.
- **Not deployed**: it uses Meta's temporary development token instead of a
  permanent system-user token, runs locally behind an ngrok tunnel, and does not
  verify Meta's `X-Hub-Signature-256` webhook signature.

## Docs

| Document | Contents |
|---|---|
| [`docs/guiones_testing.md`](docs/guiones_testing.md) | the 18 test scripts, results and findings with status |
| [`docs/explicacion-tecnica-dias-1-2.md`](docs/explicacion-tecnica-dias-1-2.md) | technical walkthrough of the FastAPI, ngrok, n8n and Meta wiring |
| [`docs/bitacora-completa-seccion-1.md`](docs/bitacora-completa-seccion-1.md), [`docs/bitacora-completa-seccion-2.md`](docs/bitacora-completa-seccion-2.md) | build log, including what broke and the root cause of each issue |
| [`PRIVACY.md`](PRIVACY.md) | privacy policy registered with Meta |

The docs are in Spanish; they were written during the build.

## License

[MIT](LICENSE)
