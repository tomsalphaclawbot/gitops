# Fallbacks & Error Handling

Configuration recipes for making your voice agents resilient to provider failures, network issues, and unexpected call endings.

---

## Phone Number Fallback Hook

Transfer to a backup number when a call ends due to an error. Add this hook to your phone number configuration:

```yaml
hooks:
  - on: call.ending
    do:
      type: transfer
      destination:
        type: number
        number: "+15551234567"
```

**Note:** A hooks PATCH replaces all existing hooks on the resource. Include all hooks in every update.

---

## Error-Handling Hook on Assistant

Catch error-related call endings and trigger a recovery action (e.g., notify via webhook, retry, or transfer). This hook covers the most common failure categories:

```yaml
hooks:
  - on: call.ending
    filters:
      - type: oneOf
        key: call.endedReason
        oneOf:
          - pipeline-error
          - pipeline
          - providerfault
          - vapifault
          - database
          - call-start
          - call.start
          - call.in-progress.error-assistant-did-not-receive-customer-audio
          - call.in-progress.error-transfer-failed
          - call.in-progress.error-sip-inbound-call-failed-to-connect
          - call.in-progress.error-sip-outbound-call-failed-to-connect
          - call.ringing.error-sip-inbound-call-failed-to-connect
          - call.forwarding.operator-busy
          - phone-call-provider-closed-websocket
          - twilio-failed-to-connect-call
          - twilio-reported-customer-misdialed
          - vonage-rejected
    do:
      - type: tool
        toolId: your-error-handler-tool-id
```

### What each filter matches

| Filter | What it catches |
|--------|----------------|
| `pipeline-error` | All pipeline failures (LLM, voice, transcriber faults) once the call pipeline has started |
| `pipeline` | Pipeline errors using the `call.in-progress.error-pipeline-*` prefix (e.g., no available LLM model) |
| `providerfault` | SIP carrier errors (403/480/503, Twilio/Vonage/Telnyx server faults) |
| `vapifault` | Vapi infrastructure issues (worker errors, missing transports) |
| `call-start` | All call-start provisioning failures (missing assistant, concurrency limits, fraud checks) |
| `call.start` | Same as above but using the dotted naming convention (future additions often use this format — keep both) |
| `database` | Database errors during call start |
| `call.in-progress.error-assistant-did-not-receive-customer-audio` | Customer audio never reached the assistant (audio path failure) |
| `call.in-progress.error-transfer-failed` | A transfer was attempted but failed |
| `call.in-progress.error-sip-*-failed-to-connect` | SIP legs (inbound or outbound) couldn't connect |
| `call.ringing.error-sip-inbound-call-failed-to-connect` | SIP inbound failure before the pipeline started (during ringing) |
| `call.forwarding.operator-busy` | Warm-transfer target reported busy |
| `phone-call-provider-closed-websocket` | Telephony provider dropped the media connection unexpectedly |
| `twilio-failed-to-connect-call` | Twilio couldn't connect the PSTN leg |
| `twilio-reported-customer-misdialed` | Twilio reported the destination number as invalid |
| `vonage-rejected` | Vonage refused to place the call (blocked destination, compliance issue) |

**Tip:** The `oneOf` filter uses substring matching for most entries. `pipeline-error` matches all `pipeline-error-*` variants. `providerfault` matches all `error-providerfault-*` variants. This keeps your config stable as new error reasons are added.

---

## Transcriber Fallback Chain

Configure fallback transcribers so your assistant keeps working if the primary STT provider goes down:

```yaml
transcriber:
  provider: deepgram
  model: nova-3
  language: en
  fallbackPlan:
    transcribers:
      - provider: assembly-ai
        language: en
        formatTurns: true
        confidenceThreshold: 0.4
        disablePartialTranscripts: false
      - provider: azure
        language: en-US
```

**Recommended fallback order:** Deepgram → AssemblyAI → Azure. Each provider has different strengths — having three ensures resilience against any single provider outage.

---

## Voice (TTS) Fallback Chain

Configure fallback voices so your assistant can still speak if the primary TTS provider goes down:

```yaml
voice:
  provider: vapi
  voiceId: Elliot
  fallbackPlan:
    voices:
      - provider: cartesia
        model: sonic-3
        voiceId: bd9120b6-7761-47a6-a446-77ca49132781
      - provider: minimax
        model: speech-02-turbo
        voiceId: vapi_jordan_ivc_voice_v1
```

**Note:** Fallback voices should be tested to ensure they sound acceptable for your use case. A voice mismatch mid-call is jarring but better than silence.

---

## Best Practices

- **Always configure at least one fallback** for transcriber and voice in production. A single-provider setup is a single point of failure.
- **Use the error-handling hook** on every production assistant. Without it, error-caused call endings are invisible.
- **Test your fallback chain** by temporarily using invalid credentials for your primary provider and verifying the fallback kicks in.
- **Keep both `call-start` and `call.start` in your error filter** — Vapi uses both naming conventions and future error reasons may use either.
