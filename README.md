
# WebSocket Message Specification

This document is the **authoritative specification** for WebSocket-based audio streaming,
AI session control, and call lifecycle handling between the telephony platform and AI clients.

---

## 1. Architecture Overview

Each inbound or outbound call establishes **one WebSocket session**.

**Message directions:**
- **Platform → AI**: Events (media, metadata, lifecycle)
- **AI → Platform**: Commands (transfer, hangup, control)

**High-level flow:**
1. Call created (`call.initiated`)
2. WebSocket connected
3. `start` event sent with call metadata
4. `media` / `dtmf` / `mark` streamed
5. AI may issue control commands (transfer, hangup)
6. `stop` event sent
7. Terminal call event emitted

---

## PART A — Platform → AI (WebSocket Events)

### 2. Connected Event

Sent immediately after WebSocket connection is established.

```json
{ "event": "connected" }
```

---

### 3. Start Event

Sent **once per call**. Contains call identity, caller/DID, and media format.

#### 3.1 Default Platform Payload

```json
{
  "event": "start",
  "sequence_number": 1,
  "stream_sid": "<stream sid>",
  "start": {
    "stream_sid": "<channel-id>",
    "call_sid": "<call-id>",
    "from": "9876543210",
    "to": "18001234567",
    "media_format": {
      "encoding": "raw/slin",
      "sample_rate": 8000
    }
  }
}
```

**Fields**
- `from` — Caller number (ANI)
- `to` — Dialed number (DID)
- `call_sid` — Unique call identifier
- `stream_sid` — WebSocket stream identifier

#### 3.2 Custom sessionStartMessage

```json
{
  "sessionStartMessage": {
    "type": "session.start",
    "from": "{{caller}}",
    "to": "{{did}}",
    "sessionId": "{{sessionId}}"
  }
}
```

Available variables: `{{caller}}`, `{{did}}`, `{{sessionId}}`

---

### 4. Media Event

Carries raw PCM audio chunks.

```json
{
  "event": "media",
  "sequence_number": 3,
  "stream_sid": "<stream sid>",
  "media": {
    "chunk": 2,
    "timestamp": "10",
    "payload": "<base64 audio>"
  }
}
```

**Audio Format**
- Encoding: raw/slin (16-bit PCM, little-endian)
- Sample rate: 8000 Hz
- Channels: Mono

---

### 5. DTMF Event

```json
{
  "event": "dtmf",
  "sequence_number": 1,
  "stream_sid": "<stream sid>",
  "dtmf": {
    "duration": "<ms>",
    "digit": "<digit>"
  }
}
```

---

### 6. Mark Event

Used to track playback completion.

```json
{
  "event": "mark",
  "sequence_number": 15,
  "stream_sid": "<stream sid>",
  "mark": { "name": "<label>" }
}
```

---

### 7. Clear Event

Clears queued audio (barge-in support).

```json
{
  "event": "clear",
  "stream_sid": "<stream sid>"
}
```

---

### 8. Stop Event

Indicates end of WebSocket stream.

```json
{
  "event": "stop",
  "sequence_number": 10,
  "stream_sid": "<stream sid>",
  "stop": {
    "call_sid": "<call-id>",
    "account_sid": "<account-id>",
    "reason": "stopped | callended"
  }
}
```

---

## PART B — AI → Platform (Session Control Commands)

### 9. Call Transfer — WebSocket API

During an active call, the AI may issue **exactly one transfer command**.

#### Transfer Types

| # | Type | Description | Message |
|---|------|------------|--------|
| 1 | External Phone | Mobile / PSTN | `session.transfer` |
| 2 | Another WebSocket (Direct) | Zero interruption | `session.transfer_ws` |
| 3 | Another WebSocket (Flow) | Brief re-setup | `session.flow_transfer` |
| 4 | SIP Extension | Agent phone | `session.transfer_extension` |
| 5 | IVR Flow | Queue / Voicemail | `session.flow_transfer` |

---

### 9.1 Transfer to External Phone

```json
{
  "type": "session.transfer",
  "destination": "9876543210"
}
```

---

### 9.2 Transfer to Another WebSocket (Direct)

```json
{
  "type": "session.transfer_ws",
  "url": "wss://new-bot.example.com/voice"
}
```

---

### 9.3 Transfer via Flow

```json
{
  "type": "session.flow_transfer",
  "flow_id": "sales_ai_flow"
}
```

---

### 9.4 Transfer to SIP Extension

```json
{
  "type": "session.transfer_extension",
  "extension": "101"
}
```

---

### 10. Other Session Commands

#### Hangup

```json
{ "type": "session.hangup" }
```

#### Send DTMF

```json
{ "type": "session.dtmf", "dtmf": "123#" }
```

#### Clear Audio

```json
{ "type": "audio.clear" }
```

---

## PART C — Call Lifecycle Events

| Event | Meaning |
|------|--------|
| call.initiated | Outbound call accepted |
| call.ringing | Destination ringing |
| call.in-progress | Call answered |
| call.completed | Normal hangup |
| call.busy | SIP 17 / 21 |
| call.no-answer | SIP 18 / 19 |
| call.failed | Network / carrier failure |
| call.canceled | Canceled before answer |

**Rules**
- One terminal event per call
- Transfer ends AI session immediately
- Caller ID preserved
- Recording continues across transfers
