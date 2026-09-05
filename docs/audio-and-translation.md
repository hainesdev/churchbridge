# Audio and translation pipeline

How sound becomes captions. This document covers the audio path in detail and
hands off to the discourse pipeline, which is summarised in the
[README](../README.md).

## The wire contract

Both capture surfaces send the same thing over a WebSocket to
`/api/stream/v1`: **base64-encoded Float32 little-endian chunks**, with the
sample rate declared in the session-start message alongside the church id,
sermon topic, and scripture version preferences.

Everything upstream of that boundary is capture-specific. Everything downstream
is shared. That contract is deliberately stable — the native app was built
against it without changing it.

## Browser capture (soundboard)

The soundboard opens the microphone with browser audio processing switched off:

```js
echoCancellation: false,
noiseSuppression: false,
autoGainControl: true,
```

An `AudioWorkletNode` pulls Float32 frames at the audio context's native rate
(48 kHz typically), frames accumulate in a buffer, and a timer flushes them to
the socket.

The reasoning is that browser DSP is tuned for conference calls, not for
recognition — it is aggressive, not configurable, and it removes information the
recogniser could have used. Capture stays as close to raw as the browser allows
and noise handling happens elsewhere.

## Native iPhone capture

The app is built around Apple's voice processing rather than around a raw
microphone tap: `AVAudioSession` in `.playAndRecord` with `.voiceChat` mode, an
`AVAudioEngine` graph, and a tap on the input node sized to roughly 20 ms of
hardware audio.

### The live path

The app ships with one path. `AudioCaptureStrategy.liveDefault` is
**`deepFilterNet3Streaming`**, and the session start routine pins the strategy
to that value before starting capture, so this is what runs in a service:

1. **Apple voice processing** stays in front, handling echo cancellation and
   automatic gain on the hardware input.
2. **Tap** at the hardware sample rate, ~20 ms buffers, on a dedicated
   processing queue so the render callback stays light.
3. **Mono downmix**, handling interleaved and non-interleaved layouts and both
   Float32 and Double sample formats.
4. **DeepFilterNet3** runs at the hardware rate — 48 kHz, matching the model's
   configuration — *before* any resampling.
5. **Wet/dry mix.** The enhanced signal is blended back at **25% wet, 75% dry**.
   Suppression is deliberately gentle; the model corrects the signal rather than
   replacing it. See [How the mix was chosen](#how-the-mix-was-chosen).
6. **Loudness compensation.** Because mixing changes level, a gain of
   `clamp(dryRMS / mixedRMS, 1 … 2.75)` is computed and applied at 95% strength,
   so the output tracks the dry signal's loudness without over-correcting.
7. **Peak limiting** at 0.98 to prevent clipping after compensation.
8. **Resample to 16 kHz** with `AVAudioConverter`.
9. **Chunk, base64-encode, and emit**, with the declared sample rate matching
   the strategy's target.

### How the mix was chosen

Full-strength suppression was tested first, in a room with a box fan running,
using a purpose-built benchmark rig: a separate iPhone app driven over a control
socket by a Python controller, playing sermon audio through speakers and
capturing it from the phone in a fixed position.

At full strength the noise floor dropped 25 to 44 dB — and intelligibility went
with it. The mask closes over quiet speech as readily as over broadband noise,
and in the worst runs the recogniser returned nothing at all. Measurably
quieter, materially worse.

The wet/dry mix was built in response, and the shipped value was selected by
**listening to the captured audio**. Automated scoring was not the deciding
factor, because it could not be trusted: a playback timing offset in the rig
shifts recorded audio relative to its reference transcript, and word error rate
counts that drift as words missed or invented. Repairing that alignment is
outstanding work, and until it lands the tuning rests on human judgement.

### DeepFilterNet3 configuration

`fftSize 960`, `hopSize 480`, `erbBands 32`, `dfBins 96`, `dfOrder 5`,
`dfLookahead 2`, `sampleRate 48000`, `normTau 1.0` — 481 frequency bins, with
running normalisation smoothed by `exp(-hopSize / sampleRate / normTau)` so
normalisation state carries across frames rather than resetting per buffer.

The neural network runs in Core ML (`computeUnits = .all`), taking `feat_erb`
and `feat_spec` and returning `erb_mask` and `df_coefs`. The STFT analysis and
synthesis, the ERB filterbank and its inverse, overlap-add memory, and the
normalisation state are all implemented in Swift against Accelerate.

It requires iOS 18 or newer. If the model or its assets cannot be loaded, the
app surfaces the reason and continues on Apple voice processing alone rather
than failing the session. Model assets are fetched from the platform at runtime
instead of being bundled.

### Other strategies

Four additional strategies exist and are used for measurement and diagnosis
rather than services: `appleVoicePassthrough` (no local resampling — the server
resamples instead), `robustVoiceFilter` (a hand-written DC blocker, high-pass,
and adaptive speech gate that attenuates to 0.22 rather than silence),
`persistentConverter`, and `ephemeralConverter` (two converter lifetime
strategies, kept from debugging converter state). They are reachable from the
benchmark and diagnostic entry points.

Capture mode — Voice Processing, Echo-Cancelled Input, or Raw Debug — is a
separate, persisted setting, defaulting to Voice Processing. The simulator is
forced onto a fallback path, and the reason is surfaced in diagnostics rather
than hidden.

### Diagnostics

The app tracks and displays roughly twenty-five signals: audio route with its
inputs and outputs, the active capture path and any fallback reason, input and
target sample rate, channel count and format, clipping, RMS level, noise floor,
speech activity, voice-processing and AGC state, echo-cancelled input
availability, chunk counts, buffer high-water mark, permission state, and engine
running state.

The purpose is diagnosis during a live service. When captions degrade, the
question "which link failed?" needs an answer immediately, and a path that
silently fell back is worse than one that failed loudly.

## Server ingest

The server decodes the base64 payload and converts Float32 to PCM16 at 16 kHz,
then feeds the streaming recognition provider — Deepgram `nova-3` by default at
`es-US`, with Google Speech supported as an alternative backend.

From there the audio path ends and the discourse pipeline begins: noise cleanup
and sentence splitting, discourse-aware buffering, language routing, machine
translation on the fast path, LLM enrichment, `display_ready` gating,
head-anchored caption merges, and delivery to the sanctuary display and mobile
listeners.

## Known characteristics

**Resampling is linear interpolation.** The server-side Float32-to-PCM16
conversion resamples by linear interpolation with no anti-aliasing pre-filter,
so content above 8 kHz folds back into the band on the way down to 16 kHz. The
effect is modest for speech but it sits upstream of every recognition result.

**The 48 kHz assumption is implicit.** DeepFilterNet3's configuration fixes
48 kHz, and it is invoked with whatever the hardware input rate happens to be.
On iPhone hardware these agree, but nothing enforces the match.

**Benchmark scoring is not yet reliable.** The capture rig's playback timing
offset makes word error rate untrustworthy for comparing nearby configurations.
No accuracy figure from it should be quoted until that is fixed.

**Two surfaces, two philosophies.** The browser path disables noise suppression
and lets the recogniser cope; the native path runs neural suppression before
sending. That difference is deliberate — a phone in a room and a soundboard feed
are different problems — but it does mean the two surfaces do not produce
identical audio for the same service.
