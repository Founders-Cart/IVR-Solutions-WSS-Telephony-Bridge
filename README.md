# IVR Solutions – WSS Telephony Bridge (Unified Documentation)

for https://www.ivrsolutions.in

This document defines **one unified interface** for connecting **telephony calls (inbound & outbound)** to **any WebSocket Secure (WSS) client** such as AI voice bots, speech engines, or custom real‑time systems.

IVR Solutions acts as a **real‑time bridge between PSTN/SIP calls and your WebSocket server**.

---

## 1. What This Enables

* Connect **any WSS endpoint** to live phone calls
* Stream **real‑time call audio → your WebSocket**
* Send **audio + commands** back into the live call
* Support **AI bots, human assist, SIP extensions, IVR flows**

**One call = one WebSocket session**

---

## 2. High‑Level Architecture

```
Caller / PSTN
      │
      ▼
IVR Solutions (SIP / Media Engine)
      │   (WSS)
      ▼
Your WebSocket Server (AI Bot / App / Engine)
```

---

## 3. How the WSS URL Is Selected

### A. Inbound Calls (DID → CONFIG_API → WSS)

When a caller dials your DID, IVR Solutions calls your **CONFIG_API** to ask how the call should be handled.

#### CONFIG_API Request (IVR → Your Backend)

```json
{
  "From": "9876543210",
  "To": "18001234567",
  "Direction": "inbound",
  "FlowId": "default_flow",
  "SessionId": "1707204900.123",
  "ServerId": "delhi"
}
```

#### CONFIG_API Response (Your Backend → IVR)

```json
{
  "url": "wss://your-bot.example.com/voice",
  "wsHeaders": {
    "Authorization": "Bearer YOUR_TOKEN"
  },
  "audioFormat": {
    "encoding": "pcm",
    "sampleRate": 8000
  }
}
```

IVR Solutions will now connect the live call to this WSS endpoint.

---

### B. Outbound Calls (API → WSS)

You can directly initiate a call and attach a WebSocket.

#### POST /calls

```json
{
  "from": "7415100898",
  "callTo": "9876543210",
  "wsUrl": "wss://your-bot.example.com/voice",
  "wsHeaders": {
    "Authorization": "Bearer YOUR_TOKEN"
  },
  "audioFormat": {
    "encoding": "pcm",
    "sampleRate": 8000
  }
}
```

---

## 4. WebSocket Session Lifecycle

1. IVR Solutions opens a WSS connection
2. `connected` event is sent
3. `start` event with call metadata
4. Continuous `media` (audio) streaming
5. Optional `dtmf`, `mark`, `clear` events
6. `stop` event when call ends

---

## 5. Events: IVR Solutions → Your WebSocket

### 5.1 Connected

```json
{ "event": "connected" }
```

---

### 5.2 Start

```json
{
  "event": "start",
  "sequence_number": 1,
  "stream_sid": "1707204900.123",
  "start": {
    "stream_sid": "1707204900.123",
    "call_sid": "call_1707204900000_a1b2c3",
    "from": "9876543210",
    "to": "18001234567",
    "media_format": {
      "encoding": "raw/slin",
      "sample_rate": 8000
    }
  }
}
```

---

### 5.3 Media (Audio)

Audio is streamed as **base64‑encoded PCM (16‑bit, mono)**.

```json
{
  "event": "media",
  "sequence_number": 3,
  "stream_sid": "1707204900.123",
  "media": {
    "chunk": 2,
    "timestamp": "10",
    "payload": "<base64-audio>"
  }
}
```

---

### 5.4 DTMF

```json
{
  "event": "dtmf",
  "sequence_number": 5,
  "stream_sid": "1707204900.123",
  "dtmf": {
    "digit": "1",
    "duration": "120"
  }
}
```

---

### 5.5 Mark (Playback Tracking)

```json
{
  "event": "mark",
  "sequence_number": 15,
  "stream_sid": "1707204900.123",
  "mark": { "name": "intro_complete" }
}
```

---

### 5.6 Clear (Barge‑in / Flush Audio)

```json
{ "event": "clear", "stream_sid": "1707204900.123" }
```

---

### 5.7 Stop

```json
{
  "event": "stop",
  "sequence_number": 42,
  "stream_sid": "1707204900.123",
  "stop": {
    "call_sid": "call_1707204900000_a1b2c3",
    "reason": "callended"
  }
}
```

---

## 6. Messages: Your WebSocket → IVR Solutions

### 6.1 Send Audio to Caller

You may send audio as **binary** or **JSON‑wrapped base64**.

```json
{
  "type": "response.audio.delta",
  "delta": "<base64-audio>"
}
```

---

### 6.2 Hangup Call

```json
{ "type": "session.hangup" }
```

---

### 6.3 Send DTMF

```json
{ "type": "session.dtmf", "dtmf": "123#" }
```

---

### 6.4 Clear Queued Audio

```json
{ "type": "audio.clear" }
```

---

## 7. Call Transfers (Bot‑Controlled)

IVR Solutions supports **advanced external phone transfers**, including **single**, **multiple simultaneous**, and **multiple sequential** dialing strategies.

---

### Transfer to External Phone (Single)

```json
{ "type": "session.transfer", "destination": "09876543210" }
```

---

### Transfer to External Phone (Multiple – Simultaneous)

All destination numbers ring **at the same time**. The first answered call is connected.

```json
{
  "type": "session.transfer",
  "destination": ["09876543210", "09876543211", "09876543212"]
}
```

---

### Transfer to External Phone (Multiple – Sequential)

Numbers are tried **one by one** in the given order until answered.

```json
{
  "type": "session.transfer",
  "destination": ["09876543210", "09876543211", "09876543212"],
  "strategy": "sequential"
}
```

---

### Transfer to Another WSS Bot

```json
{ "type": "session.transfer_ws", "url": "wss://new-bot.example.com/voice" }
```

---

### Transfer to IVR Flow / Queue

```json
{ "type": "session.flow_transfer", "flow_id": "sales_flow" }
```

---

### Transfer to SIP Extension

```json
{ "type": "session.transfer_extension", "extension": "101" }
```

---

### Transfer Message Summary Table

| Transfer Type                        | WebSocket Message                                                                          |
| ------------------------------------ | ------------------------------------------------------------------------------------------ |
| External Phone (single)              | `{ "type": "session.transfer", "destination": "9876543210" }`                              |
| External Phone (multi, simultaneous) | `{ "type": "session.transfer", "destination": ["num1","num2","num3"] }`                    |
| External Phone (multi, sequential)   | `{ "type": "session.transfer", "destination": ["num1","num2"], "strategy": "sequential" }` |

---

### WebSocket Message Format (Transfer Examples)

```json
// Single number (works as before)
{ "type": "session.transfer", "destination": "09876543210" }

// Multiple numbers – simultaneous (all ring at once)
{
  "type": "session.transfer",
  "destination": ["09876543210", "09876543211", "09876543212"]
}

// Multiple numbers – sequential (try one by one)
{
  "type": "session.transfer",
  "destination": ["09876543210", "09876543211", "09876543212"],
  "strategy": "sequential"
}
```

## 8. Call Status Callbacks (Optional)

IVR Solutions can POST lifecycle updates to your server:

* `call.initiated`
* `call.ringing`
* `call.in-progress`
* `call.completed`
* `call.failed`
* `call.busy`
* `call.no-answer`

Retries: **up to 5 attempts with exponential backoff**.

---

## 9. Audio Format & Custom Protocol Mapping

You can adapt IVR Solutions to **any WebSocket schema** (OpenAI Realtime, custom STT/TTS engines, etc.) using:

* `messageType`: `binary` or `json`
* `inputTemplate`
* `outputMessageType`
* `outputAudioPath`
* custom `sessionStartMessage`

This makes IVR Solutions a **universal telephony ↔ WebSocket adapter**.

---

## 10. Summary

* Any phone call ↔ Any WebSocket
* Real‑time bidirectional audio
* Bot‑controlled transfers & actions
* Works for AI, humans, IVR, SIP, or hybrids

This single interface powers **AI voice agents, smart IVRs, call centers, and automation workflows**.
