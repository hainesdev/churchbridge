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

**Capture is treated as an engineering problem, not a given.**

Sanctuary audio is hostile — PA bleed, room reverberation, a soundboard feed of
unknown quality. Alongside browser capture from the soundboard, a native iPhone
app handles production-quality voice capture using iOS audio processing, with
live diagnostics for route, sample rate, clipping, speech activity, and
reconnection state.

## What works today

This is a working system, not a proposal.

- **Live capture** from a soundboard through the browser, or from a native
  iPhone app.
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
- **Scripture cross-references** suggested per display-ready sentence, on a
  separate lightweight call so they never compete with structural decisions for
  model attention.
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
                  ▼      ▼                   │
              Broadcaster ◄──────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
  Sanctuary display     Mobile listeners
  /api/display/v1       /api/listen/v1
```

The detailed event lifecycle — `feed_commit`, `feed_revision`, `caption_merge`,
alignment payloads, persistence boundaries — lives in the platform repository.

## Repositories

| Repository | What it is |
| --- | --- |
| [churchbridge-platform](https://github.com/hainesdev/churchbridge-platform) | Web client, FastAPI backend, translation pipeline, sanctuary display, mobile listener, deployment |
| [churchbridge-ios](https://github.com/hainesdev/churchbridge-ios) | Native SwiftUI iPhone app for production-quality voice capture |

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
