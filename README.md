# BetterGenius Project Spec (Design-Only)

## 1) User & Decision

**Who will use this?**
* **Music fans** who want to better understand the meaning of lyrics
* **Artists** who want to engage with their audience and explain their songs
* **Community contributors** (like on Genius) who enjoy annotating and sharing interpretations
* **Casual listeners** who just want quick lyric access

**What decision does your prediction/design inform?**
* Your design helps decide **how users consume and interact with music knowledge**:
   * For listeners, it informs the decision of *whether to dive deeper into a song's meaning*
   * For artists, it informs *how to present context around their lyrics*
   * For contributors, it informs *what annotations or insights to add and how to collaborate*

## 2) Target & Horizon

**Target:** Confidence score (0-1) for each analytical insight generated about lyric meaning and themes

**Horizon:** Analysis complete within 2-3 minutes of lyric upload

## 3) Features (No Leakage)

**Available at prediction time:**
* Lyric text content (word patterns, sentiment, structure, literary devices)
* Artist metadata (personal life, genre, era, geography) 
* Cultural/historical context markers from release date
* Song metadata (album, collaborators, production details)
* Literary device patterns (metaphors, repetition, rhyme schemes)
* Language and regional dialect indicators

**Excluded to avoid leakage:**
* User feedback on interpretations
* Post-release cultural events or memes
* Streaming popularity data
* Future artist interviews explaining the song
* Community voting/rating on existing annotations

## 4) Baseline → Model Plan

**Baseline (can implement immediately):**
Rule-based system matching keywords to theme templates:
* Love/relationships: "heart", "forever", "you and I" → romance analysis
* Loss/sadness: "gone", "miss", "tears" → grief analysis  
* Rebellion: "fight", "free", "break" → protest analysis
* Simple sentiment scoring and template explanations for detected themes

**Model Plan:**
Transformer-based text classification model fine-tuned on lyric analysis tasks
* **Why it's better:** Can capture nuanced context, metaphorical language, and multi-layered meanings that keyword matching misses
* **Hypothesis:** Pre-trained language models understand literary devices and can identify thematic patterns across different musical genres and time periods

## 5) Metrics, SLA, and Cost

**Metrics:**
* **Primary:** User relevance rating >4.0/5.0 for generated interpretations
* **Secondary:** Theme classification accuracy (F1-score >0.75)
* **Coverage:** >80% of uploaded lyrics successfully analyzed

**Why these fit:**
* User rating directly measures value delivered to end users
* Classification accuracy ensures technical quality
* Coverage ensures system reliability across diverse music catalog

**SLA:**
* **p95 latency:** < 180 seconds for complete analysis
* **Cost envelope:** < $0.50 per analysis (including compute and API calls)

## 6) API Sketch

### 6.1 Endpoints

| Method | Path | Purpose | Auth? |
|--------|------|---------|-------|
| POST | /v1/analyze | Submit lyrics for analysis, return insights | Bearer |
| GET | /v1/analysis/{id} | Retrieve analysis results | Bearer |
| GET | /v1/health | Liveness check | None |
| GET | /v1/status/{id} | Check analysis progress | Bearer |

**Notes:** 
* Versioning: `/v1` for initial release
* Rate limits: 10 analyses per minute per user
* Idempotency: Use `Analysis-Id` header for duplicate prevention

### 6.2 Request/Response Examples

**Request (POST /v1/analyze):**
```json
{
  "lyrics": "Hello darkness, my old friend\nI've come to talk with you again...",
  "artist": "Simon & Garfunkel",
  "song_title": "The Sound of Silence",
  "release_year": 1964,
  "genre": "folk rock",
  "analysis_id": "uuid-12345"
}
```

**Response (202 - Analysis Started):**
```json
{
  "analysis_id": "uuid-12345",
  "status": "processing",
  "estimated_completion": "2024-09-19T15:03:00Z"
}
```

**Response (GET /v1/analysis/uuid-12345 - Complete):**
```json
{
  "analysis_id": "uuid-12345",
  "status": "complete",
  "insights": [
    {
      "theme": "isolation",
      "confidence": 0.92,
      "explanation": "The personification of darkness as a friend suggests profound loneliness...",
      "evidence": ["Hello darkness, my old friend", "talking to myself"]
    },
    {
      "theme": "social_commentary", 
      "confidence": 0.87,
      "explanation": "References to neon signs and subway walls indicate urban alienation...",
      "evidence": ["neon light", "subway walls"]
    }
  ],
  "processing_time_ms": 45000,
  "cost_cents": 23
}
```

**Error (422 - Invalid Payload):**
```json
{
  "error": "invalid_payload",
  "message": "Missing required field",
  "fields": ["lyrics"]
}
```

**Error (429 - Rate Limited):**
```json
{
  "error": "rate_limited", 
  "message": "Analysis limit exceeded",
  "retry_after": 60
}
```

### 6.3 Auth Scheme

**Authorization:** `Bearer <JWT_token>` in request header
* JWT tokens issued by AWS Cognito User Pool
* Token validation via Cognito public keys
* Rate limiting tied to Cognito user ID (`sub` claim)
* Required claims: `sub` (user ID), `email_verified`, `cognito:groups` (optional for role-based access)
* Token expiration: 1 hour (configurable in Cognito)
* Refresh tokens handled by client applications