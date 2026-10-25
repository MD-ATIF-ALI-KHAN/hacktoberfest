# Third Eye

**Sees further than a cane reaches.**

A camera assistant that tells a blind user what's ahead, judges whether the path is
walkable, and says what to do about it — out loud, powered by **Gemma** multimodal
vision through the Google AI Studio API.

---

## The problem

A white cane tells you about the ground within about **1.2 metres**, and only where the
tip actually sweeps. It finds nothing at waist, chest or head height — an open van door,
a cafe A-frame, scaffolding, a low branch. And it tells you nothing about what is coming
until you are already touching it.

Blind travellers navigate expertly within that 1.2 m. The gap is everything beyond it.

## What it does

Point the camera forward and press one button. Every reading returns three things:

```
ASSESS: STOP
SEE:    metal pole ahead, two steps, chest height
DO:     stop, move left to pass
```

- **A verdict** — `CLEAR`, `CAUTION` or `STOP`, spoken first
- **What it sees** — telegraphic, most urgent thing in the first three words, with
  direction, rough distance in steps, and height band
- **What to do** — a concrete physical instruction

You can also hold the mic and ask a free-form spoken question about whatever the camera
sees, or tap a preset: *Read the text · What's in my path · Find the door · Steps or stairs*.

## Why Gemma

What a vision-language model does that an object detector cannot is answer **open-ended
questions about a scene it was never trained on**:

> "Is there a step at that doorway?"
> "What platform does that sign say?"
> "Which of these doors is the entrance?"

A detector returns a class label and a bounding box. Gemma returns an answer. That gap is
the entire premise of this project.

## Architecture

```
       ┌──────────────┐          ┌────────────────────┐
       │    Camera    │          │    Hold to ask     │
       │ getUserMedia │          │ SpeechRecognition  │
       └──────┬───────┘          └─────────┬──────────┘
              │                            │
              ▼                            │
   ┌────────────────────────┐              │
   │  Portrait crop  9:16   │              │
   │ 384px · q0.6 · ~15 KB  │              │
   └───────────┬────────────┘              │
               │         ┌─────────────────┘
               ▼         ▼
   ┌──────────────────────────────┐
   │  Request: image + prompt     │
   │  three-line output contract  │
   └──────────────┬───────────────┘
                  │  POST streamGenerateContent (SSE)
                  ▼
   ╔══════════════════════════════╗
   ║      GEMMA  vision           ║
   ║      Google AI Studio        ║
   ╚══════════════┬═══════════════╝
                  │  token stream
                  ▼
   ┌──────────────────────────────┐
   │  Line parser                 │
   │  ASSESS → SEE → DO           │
   └──────────────┬───────────────┘
                  ▼
   ┌──────────────────────────────┐
   │  Traffic guard               │
   │  blocks crossing verdicts    │
   └──────────────┬───────────────┘
        ┌─────────┼─────────┬──────────────┐
        ▼         ▼         ▼              ▼
   ┌─────────┐ ┌────────┐ ┌────────┐ ┌───────────┐
   │ Verdict │ │ Answer │ │ DO bar │ │  Readings │
   │  pill   │ │  text  │ │        │ │    log    │
   └────┬────┘ └───┬────┘ └───┬────┘ └───────────┘
        └──────────┼──────────┘
                   ▼
   ┌──────────────────────────────┐
   │  Web Speech TTS              │
   │  verdict spoken FIRST        │
   └──────────────┬───────────────┘
                  ▼
          "Stop."   in ~1 second
```

Two details carry most of the design:

- **The verdict is the first line of the response**, and the stream is parsed line by
  line — so the one word that matters is spoken while the explanation is still generating.
- **The traffic guard sits after the model, not inside it.** A prompt is a request; this
  needs to be a requirement.

Model selection is resolved at runtime: the app calls `ListModels` with your key, ranks
Gemma vision models smallest-first, and picks the fastest available — so a changed or
retired model ID cannot break it.

## Latency is the product

A person who is walking cannot wait. The first working build took **46 seconds**. Five
changes brought it down:

1. **Streamed over SSE**, parsed line by line. `ASSESS` is the first and shortest line, so
   the user hears **"Stop."** about a second before the explanation arrives. *Time to
   first word* is the metric, not time to a finished sentence.
2. **Smallest capable Gemma vision model first.** A 4B model saying "kerb ahead, two
   steps" in 2 s beats a 31B model saying it beautifully in 40 s.
3. **384px frames at JPEG q0.6** — image tokens dominate prefill cost.
4. **48-token output cap**, prompt constrained to 15 telegraphic words.
5. **A new press aborts the request in flight**, so nothing queues behind a stale call.

The telemetry panel reports *first word*, *total* and *image KB* separately, so the
bottleneck is visible rather than guessed at.

## Judging the path, and the one thing it refuses

Third Eye **does** advise. For fixed things — obstacles, steps, kerbs, drop-offs, gaps,
doors — it gives a verdict and a concrete instruction, because those things don't move, a
two-second delay costs nothing, and that guidance is exactly what a sighted guide would
give.

The one judgement it refuses is **traffic**. It cannot see a car outside the frame, and in
the seconds between capture and speech that car has already arrived. So any road, kerb,
crossing or vehicle in view forces `CAUTION` or `STOP`, and the advice becomes *"wait, you
need a person to help you cross."* Phrases like "no cars", "traffic is clear" and "safe to
cross" are blocked in the prompt **and** again by an output filter in code — because a
prompt is a request, and this needs to be a requirement.

It also never claims something is absent, since the failure mode of a vision model is
missing things.

**This is an assistive supplement, not a mobility aid.** A cane or guide dog remains
primary. It is not a medical device.

## Design decisions

- **Vertical 9:16 camera panel**, and capture crops to match the preview exactly — what
  you see is what the model gets. That is also the right crop for walking: ground to head
  height in the path, no wasted pixels at the sides.
- **Replayable readings log.** A spoken answer is gone the moment it finishes; each
  reading stays on screen with its latency, and tapping it speaks again with no second
  API call.
- **Set in [Atkinson Hyperlegible](https://www.brailleinstitute.org/freefont)**, the
  typeface the Braille Institute designed for low-vision readers.
- **Operable without sight**: one large primary button, keyboard `Space`, tap-anywhere on
  the camera, live regions for screen readers, speech rate adjustable to 3×.

## Built with

Plain HTML, CSS and JavaScript — no framework, no build step, one file. Gemma multimodal
vision via the Google AI Studio API (`streamGenerateContent`, SSE). Web Speech API for
text-to-speech and push-to-talk recognition. `getUserMedia` + canvas for capture.

## Run it

```bash
git clone https://github.com/MD-ATIF-ALI-KHAN/hacktoberfest.git
cd hacktoberfest
python -m http.server 8000
```

Open <http://localhost:8000> and paste an API key from
[aistudio.google.com/apikey](https://aistudio.google.com/apikey) under **Setup**.

> One key covers both families — Google AI Studio issues a single key that serves **Gemma
> and Gemini**. There is no separate Gemma key, only a different model name.

The camera needs a secure context, so use `localhost`, or tunnel for phone testing:

```bash
cloudflared tunnel --url http://localhost:8000
```

## Honest limitations

- A cloud round trip cannot catch a fast-moving hazard. A production version needs an
  on-device reflex layer beneath this one — depth plus a detector, under 100 ms, driving
  haptics — with Gemma on the semantic layer where it is unbeatable.
- Motion blur while walking degrades accuracy; frames are not yet sharpness-gated.
- Not yet tested with blind users. Every decision about audio pacing and verbosity is a
  hypothesis until it is.

## Links

- **Repository:** https://github.com/MD-ATIF-ALI-KHAN/hacktoberfest
- **Licence:** Apache-2.0
