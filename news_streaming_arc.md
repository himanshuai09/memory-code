
```mermaid
flowchart TD
    subgraph S1["1. Broadcast Audio Capture & Ingestion"]
        TV["Live TV / Audio Broadcast\n(CNBC-TV18 / ET NOW / NDTV Profit)"]
        REC["Audio Recorder / FFmpeg Segmenter\n(30-minute chunks / 1800s)"]
        SPOOL[("Local Audio Spool /\nAzure Blob storage")]
        R_CHUNKS["Redis Stream\n(news_audio_chunks)"]
        
        TV -->|MPEG-TS or HLS Stream| REC
        REC -->|Save audio file| SPOOL
        REC -->|Push metadata| R_CHUNKS
    end

    subgraph S2["2. Worker Dispatch & Routing"]
        WORKER["NewsTranscriberWorker\n(Consumer Group: transcriber_group)"]
        ROUTER{"Provider Router\n(TRANSCRIPTION_PROVIDER)"}
        
        R_CHUNKS -->|XREADGROUP| WORKER
        WORKER --> ROUTER
    end

    subgraph S3["3. Unified n8n Trigger & Qwen3.5-35B Pipeline"]
        N8N_CLIENT["N8nTranscriptionClient\n(live_listening_and_transcribing)"]
        N8N_WEBHOOK["n8n Webhook Trigger\n(:5678/webhook/audio-transcribe-summarize)"]
        
        subgraph K5_CLUSTER["K5 GPU / Microservice Cluster"]
            STT_ENGINE["Speech-to-Text Engine\n(Whisper Turbo / Transcriber)"]
            QWEN_MODEL["Qwen3.5-35B Model\n(Financial Summarization)"]
            STT_ENGINE -->|Raw Transcript| QWEN_MODEL
        end
        
        ROUTER -->|Provider is n8n| N8N_CLIENT
        N8N_CLIENT -->|HTTP POST audio file| N8N_WEBHOOK
        N8N_WEBHOOK -->|Trigger workflow| STT_ENGINE
        QWEN_MODEL -->|Structured JSON items| N8N_WEBHOOK
        N8N_WEBHOOK -->|HTTP 200 Response| N8N_CLIENT
        end

    subgraph S4["4. Standalone Fallback Providers"]
        GEMINI_CLIENT["GeminiTranscribeClient\n(google/gemini-3-flash-preview)"]
        RUNPOD_CLIENT["RunPodWhisperClient\n(large-v3-turbo on CUDA)"]
        
        ROUTER -.->|Provider is gemini| GEMINI_CLIENT
        ROUTER -.->|Provider is whisper| RUNPOD_CLIENT
    end


    subgraph S6["6. Storage, Streaming & Client Consumption"]
        DB_TRANSCRIPTS[("PostgreSQL\nnews_transcripts")]
        DB_NEWS_ITEMS[("PostgreSQL\nnews_items")]
        REDIS_TRANSCRIPT_STREAM["Redis Stream\nnews_transcript_stream"]
        REDIS_NEWS_EVENTS["Redis Stream\nnews_stream_events"]
        SSE_ENDPOINT["FastAPI SSE Endpoints\n/api/v1/news/audio/transcripts/stream\n/api/v1/news/stream"]
        FRONTEND["InvestKode Frontend\n(Real-Time Audio News Feed)"]

        N8N_CLIENT -->|Transcript text & segments| DB_TRANSCRIPTS
        N8N_CLIENT -->|Categorized items| DB_NEWS_ITEMS
        DB_TRANSCRIPTS -.-> REDIS_TRANSCRIPT_STREAM
        DB_NEWS_ITEMS -.-> REDIS_NEWS_EVENTS
        REDIS_TRANSCRIPT_STREAM --> SSE_ENDPOINT
        REDIS_NEWS_EVENTS --> SSE_ENDPOINT
        SSE_ENDPOINT -->|Server-Sent Events| FRONTEND
    end

    style S3 fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style K5_CLUSTER fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style S6 fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
```

---

## 2. Sequential Runtime Flow Diagram

The following sequence diagram traces the complete lifecycle of a 30-minute audio segment:

```mermaid
sequenceDiagram
    autonumber
    actor TV as TV Broadcast
    participant REC as FFmpeg / Segmenter
    participant R_Q as Redis (news_audio_chunks)
    participant WORKER as NewsTranscriberWorker
    participant CLIENT as N8nTranscriptionClient
    participant N8N as n8n Webhook (:5678)
    participant K5 as K5 (STT + Qwen3.5-35B)
    participant PG as PostgreSQL
    participant R_EV as Redis (news_stream_events)
    participant SSE as FastAPI SSE
    actor UI as InvestKode UI

    TV->>REC: Continuous Audio Stream (CNBC / ET NOW)
    REC->>REC: Accumulate 30-min chunk (1800s)
    REC->>R_Q: XADD news_audio_chunks (chunk_path, channel, duration)
    
    R_Q->>WORKER: XREADGROUP new chunk entry
    WORKER->>WORKER: Read audio bytes / load file
    WORKER->>CLIENT: transcribe_and_summarize(audio_bytes, channel="CNBC")
    
    CLIENT->>N8N: POST /webhook/audio-transcribe-summarize (file + channel)
    N8N->>K5: Forward audio payload to service
    Note over K5: 1. Speech-to-Text generates transcript<br/>2. Qwen3.5-35B extracts high-impact items
    K5-->>N8N: Return transcript, segments, items
    N8N-->>CLIENT: HTTP 200 OK (Structured JSON response)
    
    CLIENT-->>WORKER: Result dict (text, segments, items)
    
    rect rgb(240, 248, 255)
        Note over WORKER,PG: Persistence & Broadcasting Phase
        WORKER->>PG: INSERT INTO news_transcripts (text, confidence, segments_json)
        WORKER->>PG: INSERT INTO news_items (category, content, extraction_type='audio')
        WORKER->>R_EV: XADD news_stream_events (Qwen3.5-35B items)
        WORKER->>R_EV: PUBLISH news_transcript_CNBC (Raw transcript payload)
        R_EV->>SSE: Yield stream event
        SSE->>UI: Live push to React terminal via SSE
    end
```

---

## 3. Data Contract: What n8n & Qwen3.5-35B Exchange

### Request to n8n Webhook:
* **URL:** `http://20.219.4.10:5678/webhook/audio-transcribe-summarize`
* **Method:** `POST`
* **Headers:** `Authorization: Bearer <N8N_TRIGGER_API_KEY>` (if configured)
* **Form Data / Body:**
  ```json
  {
    "channel": "CNBC",
    "source": "CNBC",
    "audio_blob_url": "https://<account>.blob.core.windows.net/news-audio/...",
    "file": "<multipart audio/mpeg or audio/wav bytes>"
  }
  ```

### Response from n8n & Qwen3.5-35B:
```json
{
  "transcript": "Good morning and welcome to Squawk Box. Tech stocks rallied today...",
  "duration": 1800.0,
  "model": "qwen3.5-35b",
  "segments": [
    {"start": 0.0, "end": 4.2, "text": "Good morning and welcome to Squawk Box."},
    {"start": 4.5, "end": 12.0, "text": "Tech stocks rallied today led by semiconductor earnings."}
  ],
  "items": [
    {
      "category": "company",
      "content": "Nvidia reported record quarterly revenue driven by data center demand."
    },
    {
      "category": "macro",
      "content": "Federal Reserve officials signaled interest rate cuts could begin next quarter."
    },
    {
      "category": "breaking_news",
      "content": "Brent Crude surged past $85/bbl following Middle East supply concerns."
    }
  ]
}
```

---

## 4. Provider Routing Logic

```mermaid
flowchart TD
    START(["Audio Chunk Ready"]) --> CHK{"Check TRANSCRIPTION_PROVIDER"}
    
    CHK -->|Provider is n8n or qwen| N8N_PATH["Invoke N8nTranscriptionClient"]
    CHK -->|Provider is gemini| GEM_PATH["Invoke GeminiTranscribeClient\n(google/gemini-3-flash-preview)"]
    CHK -->|Provider is whisper| RUN_PATH["Invoke RunPodWhisperClient\n(large-v3-turbo GPU)"]
    CHK -->|Provider is azure| AZ_PATH["Invoke AzureSpeechClient"]
    
    N8N_PATH --> N8N_RES["Returns: Transcript + Qwen3.5-35B Items"]
    GEM_PATH --> TXT_ONLY["Returns: Raw Transcript Only"]
    RUN_PATH --> TXT_ONLY
    AZ_PATH --> TXT_ONLY
    
    N8N_RES --> HAS_ITEMS{"Has extracted items"}
    HAS_ITEMS -->|Yes| SAVE_ITEMS["Save directly to news_items\n(No LLM Gateway call needed)"]
    
    TXT_ONLY --> CHK_GW{"ENABLE_LLM_GATEWAY_SUMMARIZATION"}
    CHK_GW -->|false (default)| SKIP_GW["Skip Summarization\n(Save transcript only)"]
    CHK_GW -->|true| CALL_GW["Call LLM Gateway\n(packages/llm_client.py)"]
    
    SAVE_ITEMS --> DISPATCH["Emit to Redis Stream & Push SSE"]
    SKIP_GW --> DISPATCH
    CALL_GW --> DISPATCH
```

---

## 5. Configuration Reference

In [`configs/concall-ui.production.env`](file:///e:/investcodeai/investcode/configs/concall-ui.production.env):

```env
# ── Transcription Provider Selection ─────────────────────────────────────────
# Options: 'n8n' (unified transcription + Qwen3.5-35B), 'gemini', 'whisper', 'azure'
TRANSCRIPTION_PROVIDER=gemini

# ── n8n Trigger & Qwen3.5-35B Service Settings ───────────────────────────────
N8N_TRANSCRIPTION_TRIGGER_URL=http://20.219.4.10:5678/webhook/audio-transcribe-summarize
N8N_TRIGGER_API_KEY=
N8N_TRIGGER_TIMEOUT_SECONDS=180

# ── Direct LLM Gateway Bypass ────────────────────────────────────────────────
# Set to false to disable direct LLM Gateway calls for audio news summarization
ENABLE_LLM_GATEWAY_SUMMARIZATION=false
```
