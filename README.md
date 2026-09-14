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

---

### Knowing What Happened to a Transfer

Sending `session.transfer` is not the end of the story — a destination may answer, reject,
be busy, or never pick up. IVR Solutions reports the outcome in three places:

1. a `call.transferring` callback when the transfer starts,
2. the final `call.completed` callback, which carries the result,
3. the [Call Status API](#9-call-status--transfer-history-api), which additionally lists
   **every** number that was rung.

#### `transfer_status` values

| Value | Meaning |
| --- | --- |
| `initiating` | Transfer started, destinations are ringing |
| `answered` | A human answered and is connected to the caller |
| `completed` | The transferred conversation finished normally |
| `failed` | No destination answered — see `transfer_hangup_cause` |
| `caller_abandoned` | The caller hung up while the destination was still ringing |
| `handed_off` | Handed to an IVR flow or SIP extension; the outcome is no longer tracked here |

#### Two recordings

A transferred call produces two different recordings, and only one of them contains the
human agent:

| Field | Covers |
| --- | --- |
| `recording_url` | The bot conversation. Ends at the moment of transfer. |
| `full_recording_url` | The **whole** call, including the conversation with the human agent. |

If you are reviewing what an agent said, use `full_recording_url`.


## 8. Call Status Callbacks (Optional)

IVR Solutions POSTs lifecycle updates to your `status_callback_url` (or `callback_url`).

### Events

| Event | When |
| --- | --- |
| `call.initiated` | Call accepted and being placed |
| `call.ringing` | Destination is ringing |
| `call.in-progress` | Call answered, media flowing |
| `call.transferring` | A transfer has started (see section 7) |
| `call.completed` | Call ended normally |
| `call.failed` | Call could not be completed |
| `call.busy` | Destination busy |
| `call.no-answer` | Destination did not answer |
| `call.canceled` | Call cancelled before answer |
| `call.recording` | A recording is ready |

> **Transferred calls now send `call.completed`.** Earlier releases stopped at
> `call.transferring` and never sent a terminal event for a call that was transferred. If
> your integration treats `call.transferring` as the end of a call, expect a following
> `call.completed` that carries the transfer result and the true total `duration`.

### Payload

Every event carries the base fields; the rest appear only when they have a value.

```json
{
  "event": "call.completed",
  "call_id": "call_1707204900000_a1b2c3",
  "from": "7415100898",
  "to": "9876543210",
  "status": "completed",
  "direction": "outbound",
  "duration": 214,
  "timestamp": "2026-02-06T09:31:12.004Z",
  "custom_params": {},

  "hangup_cause": "16",
  "hangup_reason": "Normal call clearing",
  "hangup_cause_txt": "Normal, unspecified",
  "disconnected_by": "caller",
  "recording_url": "https://recordings.example.com/call_1707204900000_a1b2c3.mp3",
  "full_recording_url": "https://recordings.example.com/17072049001234.wav",

  "transfer_to": "09876543210,09876543211",
  "customer_no": "9876543210",
  "transfer_status": "completed",
  "transfer_answered_by": "09876543211",
  "transfer_answered_at": "2026-02-06T09:29:41.882Z",
  "transfer_duration": 143
}
```

| Field | Notes |
| --- | --- |
| `duration` | Whole call in seconds, including time with a human agent |
| `transfer_to` | Every destination that was attempted, comma separated |
| `transfer_answered_by` | The one that actually answered |
| `transfer_duration` | Seconds the caller spent talking to the agent, distinct from `duration` |
| `transfer_hangup_cause` | Why the transfer failed, when nobody answered |
| `full_recording_url` | Whole-call recording, including the agent |

### Delivery

* Retries: up to **5 attempts** with exponential backoff (1s, 2s, 4s, 8s, 16s)
* Timeout: 10s per attempt
* Each request is signed — verify it before trusting the body:

| Header | Value |
| --- | --- |
| `X-Signature` | HMAC-SHA256 of the raw JSON body |
| `X-Signature-Algorithm` | `HMAC-SHA256` |
| `X-Call-Id` | The `call_id` |
| `X-Timestamp` | Matches `timestamp` in the body |

---

## 9. Call Status & Transfer History API

### `GET /calls/{call_id}`

Returns the current state of a call. For a transferred call it also lists **every**
destination that was rung, in order.

```json
{
  "call_id": "call_1707204900000_a1b2c3",
  "status": "completed",
  "direction": "inbound",
  "from": "9876543210",
  "to": "18001234567",
  "duration": 214,
  "recording_url": "https://recordings.example.com/call_1707204900000_a1b2c3.mp3",
  "full_recording_url": "https://recordings.example.com/17072049001234.wav",
  "telephony_call_id": "1707204900.1234",

  "transfer_to": "09876543210,09876543211",
  "transfer_status": "completed",
  "transfer_answered_by": "09876543211",
  "transfer_duration": 143,

  "transfer_legs": [
    {
      "transfer_seq": 1,
      "strategy": "simultaneous",
      "leg_no": 1,
      "legs_total": 2,
      "destination": "09876543210",
      "answered": false,
      "status": "rejected",
      "hangup_cause": "21",
      "hangup_cause_txt": "Call Rejected",
      "ring_seconds": 3,
      "talk_seconds": 0,
      "telephony_call_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
    },
    {
      "transfer_seq": 1,
      "strategy": "simultaneous",
      "leg_no": 2,
      "legs_total": 2,
      "destination": "09876543211",
      "answered": true,
      "status": "completed",
      "ring_seconds": 7,
      "talk_seconds": 143,
      "answered_at": "2026-02-06T09:29:41.882Z",
      "telephony_call_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
    }
  ]
}
```

`transfer_legs` is present only for calls that were transferred.

### `GET /calls/{call_id}/transfer-legs`

The legs on their own, with the winner lifted to the top level — useful for polling a
transfer while it is still in progress.

```json
{
  "call_id": "call_1707204900000_a1b2c3",
  "count": 2,
  "answered_by": "09876543211",
  "answered_telephony_call_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "transfer_legs": [ ... ]
}
```

### Leg fields

| Field | Meaning |
| --- | --- |
| `strategy` | `simultaneous` (all at once) or `sequential` (one by one) |
| `leg_no` / `legs_total` | This destination's place in the attempt order |
| `destination` | The number that was rung |
| `answered` | Whether this destination picked up |
| `status` | `ringing`, `answered`, `completed`, `canceled`, `busy`, `rejected`, `no_answer`, `timeout`, `failed` |
| `canceled` | Another destination answered first (simultaneous only) |
| `ring_seconds` | How long it rang before answering or failing |
| `talk_seconds` | How long this agent talked |
| `hangup_cause_txt` | Q.850 reason text when the destination did not answer |
| `telephony_call_id` | Identifier for this leg, for correlation with call records |
| `transfer_seq` | Which transfer, if a call was transferred more than once |

---

## 10. Audio Format & Custom Protocol Mapping

You can adapt IVR Solutions to **any WebSocket schema** (OpenAI Realtime, custom STT/TTS engines, etc.) using:

* `messageType`: `binary` or `json`
* `inputTemplate`
* `outputMessageType`
* `outputAudioPath`
* custom `sessionStartMessage`

This makes IVR Solutions a **universal telephony ↔ WebSocket adapter**.

---

## 11. Summary

* Any phone call ↔ Any WebSocket
* Real‑time bidirectional audio
* Bot‑controlled transfers & actions
* Works for AI, humans, IVR, SIP, or hybrids

This single interface powers **AI voice agents, smart IVRs, call centers, and automation workflows**.
