# ChurchBridge

**Live English captions for Spanish preaching — built around how preachers
actually speak.**

ChurchBridge listens to a live sermon, understands it as *discourse* rather than
as a stream of sentences, and delivers English captions to the sanctuary screen
and to phones in the pews — without asking the preacher to change a thing.

A working system rather than a prototype: two capture surfaces, a real-time
translation pipeline, an on-device neural noise suppressor implemented from
scratch, and a purpose-built measurement rig to decide between the options.

Built and operated by [Daniel Haines](https://dhaines.dev).

---

## The problem

A bilingual congregation has three bad options.

**A human interpreter.** Excellent when you have one. Most churches don't, not
every week, and not for both services.

**Nobody translates.** Half the room follows along. Half doesn't.

**Off-the-shelf live translation.** This is the option that looks like it should
work, and it is the one that fails most interestingly.

Feed a sermon into a general-purpose speech translator and you get captions that
are technically correct and practically unusable. The reason is that preaching is
not dictation:

- **Preachers speak in fragments.** "And when Paul says — when Paul writes to the
  church at Corinth —" is three false starts before a single complete thought.
  A sentence-at-a-time translator emits three captions, each one nonsense on its
  own.
- **Bilingual preachers code-switch mid-sentence.** A Spanish sermon drops into
  English for a quoted verse and back again. A translator told "the input is
  Spanish" will faithfully mistranslate the English.
- **Meaning arrives late.** Spanish word order routinely withholds the thing that
  makes a sentence make sense until the end. Translate early and you translate
  wrong.
- **Corrections are worse than delay.** Captions that appear, rewrite themselves,
  then jump to a different position on screen are harder to read than captions
  that simply arrived a beat later.

The naive pipeline — transcribe, translate, display — produces exactly these
failures. ChurchBridge exists because the interesting engineering is in
everything *between* those three steps.

## The approach

**Wait for a complete thought, not a complete sentence.** A discourse-aware
buffer holds transcribed text and refuses to release it while it still looks
structurally incomplete — a trailing connector word, an unclosed inverted
question mark, fewer than four words accumulated. Terminal punctuation, a
voice-activity signal, or a fallback timer releases it. The buffer's timers
*extend* when the text looks unfinished, so a preacher who trails off mid-clause
doesn't generate a caption until the clause lands.

**Let a fast path and a smart path run at different speeds.** Machine
translation handles the fast path, so something legible appears quickly. An LLM
then reviews the same sentence asynchronously and returns a better translation
along with structural judgments: is this thought complete, does it need
continuation, is it a fragment, is it scripture or exposition or application.
Immediacy and quality stop competing for the same time budget.

**One gate decides what reaches the screen.** Every sentence carries a
`display_ready` flag, and the server enforces it deterministically — the model
can make the gate stricter, never looser. When a sentence isn't display-ready,
its translation is suppressed until either a merge arrives or an adaptive
timeout releases the best safe fallback.

**When fragments merge, the caption doesn't move.** The hardest readability
problem is the visual jump. ChurchBridge uses head-anchored caption chains: when
the model signals that a fragment belongs with the previous one, the *earliest*
segment keeps its position on screen and later fragments are absorbed into it.
Three fragments become one stable caption, right where the first one already
was. Scripture citations follow the same chains, so a detected verse never ends
up attached to a caption that no longer exists.

## Capture, and a lesson about measurement

Everything downstream is bounded by what the microphone actually hears, and a
live church service is a genuinely hostile room: music and congregational
response overlapping the preacher, PA reinforcement bleeding back into the mic,
and reverberation off hard surfaces. A phone in that room is not a soundboard
feed. Most of the engineering in the iPhone app is about that gap.

The app carries a full streaming implementation of **DeepFilterNet3** — the
neural network in Core ML, with the signal chain around it written in Swift
against Accelerate: STFT analysis and synthesis, an ERB filterbank and its
inverse, overlap-add memory, and normalization state carried across frames so
enhancement stays continuous on a live stream rather than being applied
per-buffer.

**Then the obvious setting turned out to be wrong, and finding that out was the
point.**

Running the model at full strength is the intuitive choice, and by every noise
measurement it looked spectacular — 25 to 44 dB of noise floor simply gone. It
also made the captions worse. At full strength the mask closes over quiet speech
as readily as over a fan; in the worst test runs the recogniser returned nothing
at all. A room can be measurably quieter and less intelligible at the same time.

So the enhanced signal is mixed back at **25%**, with the rest left dry. The
model corrects the signal instead of replacing it, loudness compensation keeps
the mix at the level of the original, and a limiter catches the result. That
value was chosen by listening to captured audio rather than by a metric — see
[what still needs work](#what-still-needs-work) for the honest reason why.

Two further rules follow from the same instinct. **Capture paths are chosen by
capability, never silently:** Apple's voice processing is preferred on real
hardware, with an echo-cancelled path and a raw path behind it, and the app
reports which is active and *why* it fell back rather than quietly degrading and
leaving the room wondering why the captions got worse. And **about twenty-five
live diagnostics** — route, format, sample rate, clipping, RMS, noise floor,
speech activity, AGC and echo-cancellation state, chunk flow — because when
capture degrades mid-service you need to know which link failed, on the spot.

The path there was not clean, and the notes in the repository say so: an initial
tap on the voice-processing I/O graph crashed, a redesign around
`AVAudioSinkNode` crashed differently, and the working design came from
rebuilding around Apple's current sample. The simulator is explicitly treated as
unfit for judging audio quality — build and flow validation only.

Those decisions are made with an instrument rather than a hunch. A separate
project, [`ChurchBridgeAudioBench`](docs/repositories.md), pairs a standalone iOS
benchmark app with a Python controller: it plays sermon audio into a real room,
captures it from a phone in a fixed position, and compares five capture
pipelines under identical acoustic conditions.

## Architecture

```text
  Soundboard mic (browser)          Native iPhone app
            │                              │
            └──────────────┬───────────────┘
                           │  PCM audio (16 kHz)
                           ▼
              FastAPI  /api/stream/v1
                           │
                           ▼
              Streaming STT provider
                  │              │
        interim results     final results
                  │              │
                  ▼              ▼
        preview language   noise cleanup, sentence splitting
        router (fast)              │
                  │                ▼
                  │      SentenceBuffer — discourse-aware gating
                  │        holds until the thought looks complete
                  │                │
                  │                ▼
                  │      Language router
                  │        Spanish → machine translation (fast path)
                  │        English → passthrough
                  │                │
                  │                ├──────────────► broadcast (immediate)
                  │                ▼
                  │      LLM enrichment (async)
                  │        improved translation + discourse structure
                  │        + display_ready + merge_with_previous
                  │                │
                  │      ┌─────────┴─────────┐
                  │  display_ready       not ready
                  │      │                   │
                  │      ▼                   ▼
                  │  translation_update   deferred release
                  │      │                or caption_merge
                  │      ▼                   │
                  │  verse suggestions       │
                  │  (separate async call,   │
                  │   gated by sermon mode)  │
                  ▼      ▼                   │
              Broadcaster ◄──────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
  Sanctuary display     Mobile listeners
  /api/display/v1       /api/listen/v1
```

Also running: bilingual routing at both interim and final level so
code-switching stops being mistranslated; sermon-mode tracking (scripture,
exposition, illustration, application, exhortation, procedural) feeding a
rolling theological context tracker; scripture citation detection hydrated from
a local Bible corpus in both a source and a display version; suggested
cross-references on a separate lightweight call so they never compete with
structural decisions; and delivery to a kiosk-mode sanctuary display and a
mobile listener PWA reachable by QR code.

[`docs/audio-and-translation.md`](docs/audio-and-translation.md) covers the audio
path in detail. The event lifecycle — `feed_commit`, `feed_revision`,
`caption_merge`, alignment payloads, persistence boundaries — lives in the
platform repository.

## What still needs work

A source-available project should be honest about its edges. These are the ones
that matter.

**Benchmark scoring is not trustworthy yet.** A playback timing offset in the
capture rig shifts recorded audio relative to its reference transcript, and word
error rate is acutely sensitive to that — a timing slip is counted as words
missed or invented when nothing was misrecognised. Until it is fixed, tuning
decisions rest on listening rather than measurement, and no accuracy figure this
rig produces should be quoted. Fixing the alignment is the highest-value work in
the audio stack, because everything else in it is waiting on trustworthy
numbers.

**The suppression tuning is a judgement, not a result.** 25% wet was chosen by
ear. It is clearly better than full strength and clearly better than nothing;
whether it beats 30% or 35% is genuinely unknown, and cannot be settled until
the scoring above is repaired.

**Resampling aliases.** Conversion to 16 kHz on the server is linear
interpolation with no anti-aliasing filter, so content above 8 kHz folds back
into the speech band. The effect is modest, it is cheap to fix, and it sits
upstream of every recognition result.

**The 48 kHz assumption is implicit.** The noise suppression model's
configuration fixes 48 kHz and is invoked with whatever the hardware input rate
happens to be. On iPhone hardware these agree. Nothing enforces it.

**The two capture surfaces disagree on principle.** The browser path disables
noise suppression and lets the recogniser cope; the phone runs neural
suppression before sending. Both choices are defensible for their context, but
the same service does not sound the same from both sources.

**Stored transcripts can differ from what the congregation saw.** Session
history records the first committed version of a segment. Later revisions and
caption merges are broadcast live but do not rewrite the stored record.

**No published accuracy numbers.** Not because they are bad — because we do not
yet trust our own measurement of them. That is the first thing on the list
above.

## Repositories

| Repository | What it is |
| --- | --- |
| [churchbridge-platform](https://github.com/hainesdev/churchbridge-platform) | Web client, FastAPI backend, translation pipeline, sanctuary display, mobile listener, deployment |
| [churchbridge-ios](https://github.com/hainesdev/churchbridge-ios) | Native SwiftUI iPhone app: voice capture, on-device noise suppression, capture diagnostics, offline scripture reader |

See [`docs/repositories.md`](docs/repositories.md) for the full map, including
the measurement rig and what is intentionally not published.
[`releases/components.yml`](releases/components.yml) records the reviewed commit
of each component for a named release — a documentation manifest, not a
submodule lockfile, because the platform and the app have independent release
cycles.

## Running it yourself

ChurchBridge needs credentials for a streaming speech provider, a machine
translation provider, and an LLM. Setup and local development instructions live
in the platform repository's README, since they change with the pipeline. The
iOS app builds from its own repository and points at a running platform
instance.

## Licensing

Source-available, not open source, under the
[PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0).

**Any noncommercial purpose is permitted** — explicitly including religious
observance and use by charitable organizations. **A church can run ChurchBridge
for its own congregation, free, forever.** That is deliberate. Commercial use of
any kind requires a separate written license.

Each repository carries a plain-language `LICENSE-FAQ.md`.

## Where it's going

The current system is discourse-aware and bilingual at the sentence level. The
target is speaker-aware, mixed-language aware, and display-aware at the segment
level: diarization shaping merge decisions and routing the way punctuation does
today; code-switching represented as a first-class mixed mode rather than
flattened into "mostly Spanish"; and a display that distinguishes direct English
from translated Spanish, mixed segments, and speaker roles such as leader lines
and congregational responses.

## Contributing

Issues, questions, and field reports from real services are genuinely welcome —
especially from churches running this in a sanctuary. Code contributions require
a contributor license agreement; see [`CONTRIBUTING.md`](CONTRIBUTING.md).

---

Built by [Daniel Haines](https://dhaines.dev) — systems integration, automation,
and the unglamorous work of making real-time things dependable.

Copyright © 2026 Daniel Haines. "ChurchBridge" is a trademark of Daniel Haines.
