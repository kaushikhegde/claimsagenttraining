# Voice Insurance Claims Agent — Design Spec

**Date:** 2026-04-02
**Status:** Approved

## Overview

A Node.js CLI voice assistant for insurance claim intake. The agent conducts a real-time voice conversation with claimants (or internal insurance agents), gathers claim details through intelligent branching questions, extracts structured data, and persists it to PostgreSQL.

Serves two personas:
1. **Internal tool** — insurance company agents process claims hands-free
2. **Customer-facing bot** — claimants interact directly with the AI to file claims

## Architecture

**Approach: Gemini Live API + Post-Processing Agent**

Three phases per session:

```
Voice Conversation (Gemini Live API)
  → Claim Extraction (Gemini Text API)
  → Data Persistence (PostgreSQL)
```

### Phase 1: Voice Session Manager

- Captures mic audio via `node-record-lpcm16` (PCM 16-bit, 16kHz mono)
- Streams to Gemini 2.0 Flash Live API over WebSocket (`wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent`) (bidirectional)
- Receives audio response chunks, plays via `speaker` npm package
- Gemini handles VAD (voice activity detection), interruptions, and turn-taking natively
- Accumulates transcript from `serverContent` events
- End-of-conversation: Gemini wraps up naturally, or user says "end call" / "that's all"

**System Prompt Strategy:**
- Role: AI insurance claims assistant
- Disclaimer: Identifies as AI, cannot provide legal/medical advice
- Conversation flow: Start with "What happened?", classify claim type via Gemini, then follow structured follow-ups for that type
- Supported claim categories: auto_accident, workplace_injury, slip_and_fall, medical_malpractice, property_damage, general_injury
- Required fields to gather (varies by type): name, contact, date, location, description, policy number, witnesses, medical provider, severity
- Guardrails: Stay on topic, never give legal/medical advice, suggest professional help

### Phase 2: Claim Extraction Agent

After the voice conversation ends, the full transcript is sent to Gemini 2.0 Flash (text API) for structured extraction.

**Claim JSON Schema:**

```json
{
  "claimType": "auto_accident | workplace_injury | slip_and_fall | medical_malpractice | property_damage | general_injury",
  "status": "draft | complete | needs_followup",
  "missingFields": [],
  "claimant": {
    "fullName": "",
    "phone": "",
    "email": "",
    "policyNumber": ""
  },
  "incident": {
    "date": "",
    "location": "",
    "description": "",
    "severity": "minor | moderate | severe | critical"
  },
  "details": {
    "vehicleInfo": {},
    "employerInfo": {},
    "premisesInfo": {},
    "providerInfo": {},
    "propertyInfo": {}
  },
  "witnesses": [{ "name": "", "contact": "" }],
  "medicalProvider": { "name": "", "contact": "" },
  "additionalNotes": ""
}
```

- Only the relevant `details` sub-object is populated based on `claimType`
- `status` is `needs_followup` if required fields are missing
- `missingFields` lists what couldn't be gathered

### Phase 3: Data Persistence

**PostgreSQL — 3 tables:**

**`claims`:**
- id (UUID PK), claim_type (ENUM), status (ENUM), claimant_name, claimant_phone, claimant_email, policy_number, incident_date (DATE), incident_location, incident_description, severity (ENUM), type_specific_details (JSONB), witnesses (JSONB), medical_provider (JSONB), additional_notes, missing_fields (JSONB), created_at (TIMESTAMPTZ)

**`transcripts` (audit):**
- id (UUID PK), claim_id (FK → claims), full_transcript (TEXT), duration_seconds (INT), created_at (TIMESTAMPTZ)

**`conversation_logs` (compliance):**
- id (UUID PK), claim_id (FK → claims), role (VARCHAR — "agent" or "claimant"), message (TEXT), timestamp (TIMESTAMPTZ)

**Tech:** `pg` npm package, `pg.Pool` for connection pooling, plain SQL migration files (no ORM).

## Project Structure

```
backend/
├── src/
│   ├── index.js              # CLI entry point
│   ├── config/
│   │   └── index.js          # Env vars, Gemini key, DB config
│   ├── voice/
│   │   ├── session.js        # Voice Session Manager
│   │   ├── prompts.js        # System prompt for Gemini Live API
│   │   └── audio.js          # Mic input & speaker output helpers
│   ├── extraction/
│   │   ├── agent.js          # Claim Extraction Agent
│   │   ├── prompts.js        # Extraction prompt template
│   │   └── schema.js         # Claim JSON schema + validation
│   ├── db/
│   │   ├── pool.js           # PG connection pool
│   │   ├── claims.js         # Claim CRUD operations
│   │   ├── transcripts.js    # Transcript storage
│   │   └── logs.js           # Conversation log storage
│   └── utils/
│       └── logger.js         # Simple console logger
├── migrations/
│   └── 001_initial.sql       # Create tables
├── .env.example              # Template for env vars
├── package.json
└── README.md
```

## Dependencies

| Package | Purpose |
|---------|---------|
| `@google/generative-ai` | Gemini 2.0 Flash text API (extraction) |
| `ws` | WebSocket client for Gemini Live API |
| `node-record-lpcm16` | Mic capture (PCM audio) |
| `speaker` | Audio playback |
| `pg` | PostgreSQL client |
| `dotenv` | Environment variables |
| `uuid` | Claim ID generation |

**System requirement:** `sox` installed (`brew install sox` on macOS).

## CLI Flow

1. Show disclaimer: "This is an AI assistant. It cannot provide legal or medical advice."
2. Connect to Gemini Live API via WebSocket
3. Open mic — agent greets: "Hello, I'm an AI claims assistant. Can you tell me what happened?"
4. Conversation streams bidirectionally
5. User says "that's all" or agent wraps up naturally
6. CLI shows "Processing your claim..."
7. Extraction agent processes transcript → structured JSON
8. Save to PostgreSQL (claim + transcript + conversation log)
9. Print claim summary + claim ID
10. Exit

## Error Handling

- **WebSocket disconnect:** Auto-reconnect once; if fails → save partial transcript, notify user
- **Mic/speaker failure:** Graceful error with troubleshooting hints (check sox, permissions)
- **Gemini API errors:** Retry with exponential backoff (max 3 attempts), then exit with saved state
- **DB connection failure:** Save claim JSON to local `.json` fallback file, log warning

## Guardrails & Compliance

- Agent identifies itself as AI at conversation start
- Never provides legal advice — "I'd recommend consulting a legal professional"
- Never provides medical advice — "Please consult your healthcare provider"
- Stays on topic — redirects off-topic questions back to claim
- No personal opinions on claim validity or fault
- Full transcript stored for audit
- Individual conversation turns logged with timestamps

## Language Support

- English only for MVP
- Architecture designed for easy multilingual support later (language config in system prompt, locale-aware extraction prompts)

## Testing Strategy

- Unit tests for extraction agent (mock transcripts → validate JSON output)
- Unit tests for DB layer (claim CRUD)
- Integration test for full flow using mock audio stream
- Manual testing for voice quality
