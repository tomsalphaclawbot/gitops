# WebSocket Transport

Rules and best practices for direct audio streaming via Vapi's WebSocket transport.

---

## Audio Format Options

| Format | Sample Rate | Bit Depth | Chunk Size (20ms) | Silence Value | Best For |
|--------|-------------|-----------|-------------------|---------------|----------|
| Mu-Law | 8000 Hz | 8-bit | 160 bytes | `0x7F` | Telephony, bandwidth-constrained |
| PCM | 16000 Hz | 16-bit | 640 bytes | `0x0000` | High quality, local processing |

---

## Critical Audio Rules

1. **Consistent Timing**: Send audio chunks at fixed 20ms intervals using a timer, not event-driven. Inconsistent timing causes VAD and STT issues.
2. **Never Stop Sending**: Fill gaps with silence values — never stop the audio stream. Silence in the stream is expected; absence of data is not.
3. **Correct Silence Values**: Mu-law silence is `0x7F` (not `0x00`, which causes loud pops). PCM silence is `0x0000`.
4. **Buffer Management**: Monitor WebSocket `bufferedAmount` and implement backpressure when the buffer exceeds 32KB.

---

## Control Messages

WebSocket transport supports real-time call management:

| Control | Purpose |
|---------|---------|
| `add-message` | Add context or messages to conversation |
| `mute-assistant` / `unmute-assistant` | Control assistant speech |
| `mute-customer` / `unmute-customer` | Control customer audio processing |
| `say` | Make assistant speak specific content |
| `end-call` | Terminate the call |
| `transfer` | Route to phone number or SIP endpoint |
| `handoff` | Switch to a different assistant |

---

## Connection Best Practices

- Implement automatic reconnection with exponential backoff for network issues
- Queue important control messages during brief disconnections
- Monitor connection health and log all WebSocket errors
- Use a heartbeat/ping mechanism to detect stale connections early

---

## Error Recovery

| Close Code | Meaning | Action |
|-----------|---------|--------|
| 1000 | Normal closure | No action needed |
| 1006 | Abnormal closure | Reconnect immediately |
| 1008 | Policy violation | Check call ID, permissions, or authentication |
| 1011 | Server error | Retry with exponential backoff |
| 1012 | Service restart | Wait 2–5 seconds, then reconnect |
| 1013 | Overloaded | Reconnect with exponential backoff |

For complete implementation details, see the [Vapi WebSocket Transport documentation](https://docs.vapi.ai/calls/websocket-transport).
