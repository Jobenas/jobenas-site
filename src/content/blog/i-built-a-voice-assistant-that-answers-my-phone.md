---
title: "I Built a Voice Assistant That Answers My Phone — The $0 Stack"
subtitle: "SIP, RTP, and the telephony debugging that AI tutorials skip"
description: "How I built a personal voice assistant that registers as a SIP extension on my PBX, handles phone calls with speech recognition and natural TTS, and costs nothing to run."
pubDate: 2026-04-15
heroImage: "../../assets/blog-placeholder-3.jpg"
---

Every existing voice assistant option assumes you either want a toy or an enterprise contract. Google Home answers trivia. Alexa controls lights. Twilio's voice API starts at fractions of a cent per minute, which compounds fast when you're running a persistent assistant that handles calls throughout the day.

I wanted something different: an AI that registers as a SIP extension on my PBX, answers incoming calls, understands speech, reasons about the question, and responds with a natural voice. Total recurring cost: zero.

Here's how it works and what broke along the way.

## The pipeline

The call path is five components chained together:

```
Phone → 3CX PBX → SIP/TCP → Oracle Server
  → RTP audio capture → Groq Whisper STT (~1.3s)
  → Copilot LLM with tool augmentation (~2.5s)
  → Google Chirp3-HD TTS (~1.5s)
  → RTP audio → Phone
```

Total round-trip from the moment I stop speaking to when the response starts playing back: roughly five seconds. Not conversational-speed, but usable — about the latency of talking to someone who's looking something up before answering.

Every component in this chain runs at zero cost. Groq's free tier handles 14,400 STT requests per day. Google Cloud's TTS has a generous free tier. The LLM runs through a copilot proxy that routes to Claude Sonnet 4.6. The PBX is 3CX Free, a hosted instance that supports two extensions at no charge.

## Why SIP and not a webhook

Most voice AI tutorials start with Twilio or a similar telephony API. You connect a webhook, Twilio streams audio to your server, you process it and stream audio back. Simple architecture, per-minute billing.

I already had a 3CX instance running for personal use. The PBX handles call routing, voicemail, and the mobile app. The cheapest integration would be registering a second extension — Oracle as extension 25011, sitting next to my own 25010 — and having it answer calls directly over SIP.

No telephony API. No per-minute charges. No third-party dependency for the call itself.

## The first attempt: pyVoIP

Python has a SIP library called pyVoIP. It handles registration, call detection, and audio — exactly what I needed. The initial code was clean:

```python
from pyVoIP.VoIP import VoIPPhone

phone = VoIPPhone(SIP_SERVER, SIP_PORT, SIP_USER, SIP_PASS)
phone.start()
# Register callback for incoming calls
```

Registration silently failed. No error, no rejection — just a phone that never appeared as online in the 3CX dashboard.

The problem: pyVoIP only supports UDP for SIP transport. 3CX's hosted instances require TCP. This isn't documented prominently on either side. SIP over UDP works for LAN deployments where the PBX sits on the same network. For hosted instances reachable over the internet, TCP is the expected transport, and UDP REGISTER packets get dropped with no response.

This is the kind of failure that wastes hours. The library doesn't raise an exception. The PBX doesn't send a rejection. You just sit there watching logs that say "sent REGISTER" with nothing coming back.

## The second attempt: baresip

Baresip is a modular SIP user agent written in C. It handles TCP, it handles every codec imaginable, it runs headless. On paper, perfect.

In practice: it crashes in daemon mode on a headless server. The `aufile` audio module is designed for one-shot playback — you can play a WAV file into a call, but you can't feed a continuous bidirectional audio stream through it. The architecture assumes an actual audio device exists. On a server with no sound card, you're fighting the tool instead of building with it.

## Writing a custom SIP client

After two dead ends, I wrote the SIP client from scratch. Not because I wanted to — because the alternatives made assumptions that didn't match the deployment environment.

The custom client is ~500 lines of Python. It handles three things: TCP SIP registration with digest authentication, INVITE/BYE call flow, and UDP RTP for audio.

SIP over TCP is straightforward once you understand the handshake. You send a REGISTER, get a 407 Proxy Authentication Required with a realm and nonce, compute the digest response, and re-register. The PBX sends back 200 OK and your extension goes online.

```python
class SIPClient:
    def connect(self):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((SIP_SERVER, SIP_PORT))

    def register(self):
        # Send REGISTER → get 407 → compute digest → retry
        # On 200 OK: self.registered = True
```

The call flow is similarly mechanical. An incoming INVITE contains the caller's SDP — their RTP IP and port. You respond with 200 OK containing your own SDP. Audio starts flowing over UDP RTP in both directions. When the call ends, either side sends BYE.

The audio format is G.711 PCMU (u-law) at 8000 Hz — the universal SIP codec. Each RTP packet carries 20ms of audio as 160 bytes. Python's `audioop` module handles the encoding and decoding between PCMU and linear PCM.

## The registration debugging story

Getting the SIP client to register took longer than writing it.

The first obstacle was `AllowLanOnly`. 3CX's default security policy restricts SIP registration to devices on the same LAN as the PBX. For a hosted instance in a DigitalOcean droplet, "the same LAN" means nothing — every device is remote. This setting had to be disabled via the 3CX API. There's no toggle in the web UI.

The second obstacle was keep-alive timing. SIP registrations expire. The PBX expects periodic re-registration to confirm the extension is still alive. Miss a deadline and the extension drops offline. The re-registration interval has to sit inside the main SIP listen loop, not in a separate thread, because sharing a TCP socket between threads introduces a race condition: the listen thread might be mid-read when the keep-alive thread tries to write, corrupting both messages.

The solution is a non-blocking loop that checks elapsed time on each iteration:

```python
while self._running:
    if time.time() - last_register > 120:
        self.register()
        last_register = time.time()

    data = self.sock.recv(8192)  # non-blocking with timeout
    # Handle INVITE, BYE, ACK...
```

Every 120 seconds, the client re-registers. Between re-registrations, it listens for incoming calls. No threads, no race conditions, no socket corruption.

## The audio pipeline

When a call connects, a handler thread starts the voice pipeline:

1. **Record**: Capture RTP packets, decode PCMU to PCM, buffer until 0.8 seconds of silence (RMS below 300)
2. **Transcribe**: Ship the WAV buffer to Groq's Whisper endpoint. The `whisper-large-v3-turbo` model returns text in ~1.3 seconds. It auto-detects language.
3. **Think**: Send the transcription to the LLM with inline tool augmentation. The agent can check calendar, email, and system status mid-conversation. Response in ~2.5 seconds.
4. **Speak**: Send the LLM's text to Google Chirp3-HD TTS. The voice ("Leda") handles both English and Spanish with natural prosody. Audio back in ~1.5 seconds.
5. **Play**: Encode the TTS audio to PCMU and stream it as RTP packets at the correct 20ms pacing.

The silence detection is critical. Too sensitive and it cuts off mid-sentence. Too lenient and the caller waits too long after finishing a sentence. The current thresholds — 0.8 seconds of silence, RMS below 300, minimum 0.3 seconds of speech — handle normal conversational patterns without clipping.

## TTS: why Piper didn't make it

The initial TTS was Piper — a fast, local, offline text-to-speech engine. It runs on CPU, produces audio in milliseconds, and costs nothing.

The problem is prosody. Piper's voices sound like a GPS giving directions. Each sentence has the same intonation contour regardless of content. For reading back a weather forecast, it's fine. For a phone conversation where the caller expects something approaching human, it's immediately obvious you're talking to a machine.

Google's Chirp3-HD model produces audio that sounds like someone actually saying the words. The latency is higher (~1.5 seconds vs milliseconds for Piper), but the quality difference is large enough that it changes whether the system feels usable or feels like a gimmick.

The language handling is a bonus. The same "Leda" voice speaks both English and Spanish without needing to switch models. Piper requires separate model files for each language, and the Spanish models are noticeably worse than the English ones.

## Tool augmentation

The voice agent isn't just a chatbot on a phone line. It has access to the same tools as the text-based assistant:

- **Calendar**: "Do I have anything this afternoon?" pulls today's events from Google Calendar
- **Email**: "Any important emails?" checks unread across three accounts
- **System status**: "Is everything running?" checks service health, memory, disk

The tool matching is intentionally crude — keyword detection, not intent classification. If the caller says "calendar" or "schedule" or "meeting" or "reunión," the calendar tool fires. This is fast and deterministic. An LLM-based intent classifier would add latency for minimal accuracy improvement on a voice interface where commands are naturally short.

## What it costs

| Component | Cost |
|-----------|------|
| 3CX Free PBX | $0 (2 extensions included) |
| Groq Whisper STT | $0 (free tier: 14,400 req/day) |
| Copilot LLM proxy | $0 |
| Google Chirp3-HD TTS | $0 (free tier) |
| Server (existing home server) | $0 marginal |
| **Total** | **$0/month** |

The entire stack runs on a home server with 4GB RAM and 2 cores. The voice agent uses negligible resources when idle and handles one concurrent call — which is all I need.

## What I learned

The SIP debugging was the expensive part. The voice pipeline — STT, LLM, TTS — worked on the first try. Each component has a clean API, returns predictable results, and fails loudly when something goes wrong.

The SIP layer is the opposite. Failures are silent. Libraries make transport assumptions they don't document. PBX security settings don't appear in the UI. Socket concurrency bugs manifest as intermittent corruption that only happens under load.

The broader lesson: when building voice AI, the AI is the easy part. The telephony integration — SIP registration, RTP audio, codec negotiation, NAT traversal — is where the complexity lives. Every tutorial that starts with "just use Twilio" is skipping the hard part, not solving it.

The voice agent has been running for two days now. It answers calls, checks my calendar, summarizes my email, and tells me if my infrastructure is healthy — all from a phone call while I'm driving or cooking. Five-second latency, zero monthly cost, and no dependency on any company's pricing decision.
