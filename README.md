# ChurchBridge

**Live English captions for Spanish preaching — built around how preachers
actually speak.**

ChurchBridge listens to a live sermon, understands it as *discourse* rather than
as a stream of sentences, and delivers English captions to the sanctuary screen
and to phones in the pews — without asking the preacher to change a thing.

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

**Wait for a complete thought, not a complete sentence.**

A discourse-aware buffer holds transcribed text and refuses to release it while
it still looks structurally incomplete — a trailing connector word, an unclosed
inverted question mark, fewer than four words accumulated. Terminal punctuation,
a voice-activity signal, or a fallback timer releases it. The buffer's timers
*extend* when the text looks unfinished, so a preacher who trails off mid-clause
doesn't generate a caption until the clause lands.

**Let a fast path and a smart path run at different speeds.**

Machine translation handles the fast path, so something legible appears quickly.
An LLM then reviews the same sentence asynchronously and returns a better
translation along with structural judgments: is this thought complete, does it
need continuation, is it a fragment, is it a quote introduction, is it scripture
or exposition or application. Immediacy and quality stop competing for the same
time budget.

**One gate decides what reaches the screen.**

Every sentence carries a `display_ready` flag, and the server enforces it
deterministically — the model can make the gate stricter, never looser. When a
sentence isn't display-ready, its translation is suppressed until either a merge
arrives or an adaptive timeout releases the best safe fallback.

**When fragments merge, the caption doesn't move.**

The hardest readability problem is the visual jump. ChurchBridge uses
head-anchored caption chains: when the model signals that a fragment belongs with
the previous one, the *earliest* segment keeps its position on screen and later
fragments are absorbed into it. Three fragments become one stable caption, right
where the first one already was.

**Scripture is handled as scripture, not as text that happens to be quoted.**

When a preacher cites a passage, the citation is detected inside the same
enrichment turn that judges the sentence's structure — it shares that context, so
it costs nothing extra. The reference is then looked up in a local Bible corpus
and rendered *twice*: in the source version the preacher is quoting from, and in
the display version the congregation reads. A Spanish citation from Reina-Valera
appears on screen in English, as the actual passage, while the preacher is still
on it.

Separately, ChurchBridge suggests one to three related cross-references per
display-ready sentence. That runs as its own lightweight call *after* the
structural decision has already settled, and it is deliberately not allowed to
delay buffering, gating, or merge repair — suggestions are the least urgent thing
on screen and are treated that way. Sermon mode gates them: no cross-references
during procedural announcements, none for fragments or sentences still pending.

Both follow the caption merge chain. When fragments are absorbed into an earlier
caption, verse events are re-pointed at the surviving segment, so a citation never
ends up attached to a caption that no longer exists.

**Capture is the hardest part, and it is treated that way.**

Everything downstream is bounded by what the microphone actually hears, and a
live church service is a genuinely hostile room: music and congregational
response overlapping the preacher, PA reinforcement bleeding back into the mic,
and reverberation off hard surfaces. A phone in that room is not a soundboard
feed. Most of the engineering in the iPhone app is about that gap.

**Noise suppression runs on the device, before audio leaves it.** The app carries
a full streaming implementation of DeepFilterNet3 — the neural network in Core
ML, with the signal chain around it written in Swift against Accelerate: STFT
analysis and synthesis, an ERB filterbank and its inverse, overlap-add memory,
and normalization state carried across frames so enhancement stays continuous on
a live stream rather than being applied per-buffer. Model assets are fetched from
the platform at runtime instead of being baked into the app.

**The obvious setting was wrong, and finding that out was the point.**
Running the model at full strength is the intuitive choice, and by every noise
measurement it looked spectacular — 25 to 44 dB of noise floor simply gone. It
also made the captions worse. At full strength the mask closes over quiet speech
as readily as over a fan; in the worst test runs the recogniser returned nothing
at all. A room can be measurably quieter and less intelligible at the same time.

So the enhanced signal is mixed back at **25%**, with the rest left dry. The
model corrects the signal instead of replacing it, loudness compensation keeps
the mix at the level of the original, and a limiter catches the result. That
value was chosen by listening to captured audio rather than by a metric — for an
honest account of why, see [what still needs work](#what-still-needs-work).

**Three capture paths, chosen by capability, never silently.** Apple's voice
processing is preferred on real hardware; there is an echo-cancelled input path
and a raw path behind it. The app reports which one is actually active and *why*
it fell back, rather than quietly degrading and leaving the room to wonder why
the captions got worse.

**About twenty-five live diagnostics** — audio route and its inputs and outputs,
input and target sample rate, channel count and format, clipping, RMS level,
noise floor, speech activity, voice-processing and AGC state, echo-cancelled
input availability, chunk counts, engine state. When capture degrades mid-service
you want to know which link failed, on the spot.

The path there was not clean, and the notes in the repository say so: an initial
tap on the voice-processing I/O graph crashed, a redesign around `AVAudioSinkNode`
crashed differently, and the working design came from rebuilding around Apple's
current sample. The simulator is explicitly treated as unfit for judging audio
quality — build and flow validation only. Capture decisions get made on real
hardware in a real room.

## What works today

This is a working system, not a proposal.

- **Live capture** from a soundboard through the browser, or from a native
  iPhone app.
- **On-device neural noise suppression** — a streaming DeepFilterNet3
  implementation in Core ML with a Swift/Accelerate STFT and ERB filterbank
  front end, with model assets served from the platform.
- **Capability-selected capture paths** — Apple voice processing, echo-cancelled
  input, or raw — with the active path and any fallback reason surfaced in the
  app.
- **Deep capture diagnostics** — route, format, sample rate, clipping, RMS,
  noise floor, speech activity, AGC and echo-cancellation state, chunk flow.
- **Streaming speech recognition** with a provider-selected backend (Deepgram
  `nova-*` by default; Google Speech supported).
- **Bilingual handling** — Spanish routed through translation, English passed
  straight through, at both interim-preview and final-sentence level, so
  code-switching stops being mistranslated.
- **Discourse-aware buffering**, with continuation signals from the LLM feeding
  back into future boundary decisions.
- **LLM enrichment** producing improved translations, discourse tags, completion
  and continuation signals, source quality, register, sermon mode, and verse
  detection.
- **Head-anchored caption merges** with adaptive deferred release.
- **Sermon state tracking** — scripture, exposition, illustration, application,
  exhortation, procedural — feeding a rolling theological context tracker.
- **Scripture citation detection**, hydrated from a local Bible corpus in both a
  source version and a display version, with per-church version selection.
- **Scripture cross-references** suggested per display-ready sentence, on a
  separate lightweight call so they never compete with structural decisions for
  model attention.
- **A scripture side panel** separating passages the preacher actually cited from
  related references the system suggests.
- **Offline scripture on device** — the iPhone app bundles its own Bible
  database, so lookups do not depend on the network.
- **Two delivery surfaces** — a sanctuary display in kiosk mode, and a mobile
  listener PWA reachable by QR code.
- **A benchmark harness** for provider and model comparison, pipeline
  degradation testing, and scorecard tracking.

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

[`docs/audio-and-translation.md`](docs/audio-and-translation.md) covers the audio
path in detail: the wire contract, both capture surfaces, the live on-device
signal chain, and server ingest. The detailed event lifecycle — `feed_commit`,
`feed_revision`, `caption_merge`, alignment payloads, persistence boundaries —
lives in the platform repository.

## Repositories

| Repository | What it is |
| --- | --- |
| [churchbridge-platform](https://github.com/hainesdev/churchbridge-platform) | Web client, FastAPI backend, translation pipeline, sanctuary display, mobile listener, deployment |
| [churchbridge-ios](https://github.com/hainesdev/churchbridge-ios) | Native SwiftUI iPhone app: voice capture, on-device DeepFilterNet3 noise suppression, capture diagnostics, offline scripture reader |

See [`docs/repositories.md`](docs/repositories.md) for the full map, including
what is intentionally not published.

[`releases/components.yml`](releases/components.yml) records the reviewed commit
of each component for a named product release. It is a documentation manifest,
not a submodule lockfile — the platform and the iOS app have independent release
cycles and are deliberately not coupled.

## Running it yourself

ChurchBridge needs credentials for a streaming speech provider, a machine
translation provider, and an LLM. Setup, configuration, and local development
instructions live in the platform repository's README — start there rather than
here, since they change with the pipeline.

The iOS app builds from its own repository in Xcode and points at a running
platform instance.

## Licensing

ChurchBridge is **source-available**, not open source, under the
[PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0).

**Any noncommercial purpose is permitted** — and that explicitly includes
religious observance and use by charitable organizations. **A church can run
ChurchBridge for its own congregation, free, forever.** That is deliberate.

Commercial use of any kind — selling it, charging for hosting, installation,
support, or integration, or building it into a for-profit product — requires a
separate written license.

Each repository carries a plain-language `LICENSE-FAQ.md`. For commercial
licensing, hosted ChurchBridge, or deployment help, see the contact details in
the platform repository.

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
suppression before sending. Both choices are deliberate and defensible for their
context, but the same service does not sound the same from both sources.

**Stored transcripts can differ from what the congregation saw.** Session
history records the first committed version of a segment. Later revisions and
caption merges are broadcast live but do not rewrite the stored record, so an
archived transcript can disagree with the repaired caption that was actually on
screen.

**No published accuracy numbers.** Not because they are bad — because we do not
yet trust our own measurement of them. That is the first thing on the list
above.

## Where it's going

The current system is discourse-aware and bilingual at the sentence level. The
target system is speaker-aware, mixed-language aware, and display-aware at the
segment level:

- **Speaker boundaries as structural signals.** Diarization should shape merge
  decisions, translation routing, and rendering the same way punctuation and
  discourse tags do today.
- **Code-switching as a first-class mode.** A segment containing both languages
  should be represented as mixed, not flattened into "mostly Spanish."
- **A display that distinguishes original from interpreted speech** — direct
  English, translated Spanish, mixed-language segments, and speaker roles such as
  leader lines and congregational responses.

## Contributing

Issues, questions, and field reports from real services are genuinely welcome —
especially from churches running this in a sanctuary.

Code contributions require a contributor license agreement, because the project
is kept under single copyright ownership. See [`CONTRIBUTING.md`](CONTRIBUTING.md)
before opening a pull request.

---

Copyright © 2026 Daniel Haines. "ChurchBridge" is a trademark of Daniel Haines.
