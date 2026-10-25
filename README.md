# Third Eye

**Sees further than a cane reaches.** A camera-based assistant that describes the
path ahead to a blind or low-vision user, out loud, powered by **Gemma** multimodal
vision through the Google AI Studio API.

A white cane tells you about the ground within about 1.2 m. Third Eye extends that
reach: point the camera forward, press one button, and hear what is ahead — obstacles,
steps, kerbs, vehicles, doorways, and any text in view. You can also hold the mic and
ask a free-form spoken question about whatever the camera sees.

---

## Why Gemma

The thing a vision-language model does that a detection model cannot is answer
**open-ended questions about a scene it was never trained on**:

> "Is there a step at that doorway?"
> "What platform does that sign say?"
> "Which of these doors is the entrance?"

An object detector returns a class label and a box. Gemma returns an answer. That gap
is the entire premise of this project.

## Run it

No build step. The camera requires a secure context, so `localhost` or HTTPS:

```bash
git clone https://github.com/MD-ATIF-ALI-KHAN/hacktoberfest.git
cd hacktoberfest
python -m http.server 8000
```

Open <http://localhost:8000>, then under **Setup** paste an API key from
[aistudio.google.com/apikey](https://aistudio.google.com/apikey).

> One key covers both families. Google AI Studio issues a single key that serves
> **Gemma and Gemini** — there is no separate Gemma key, only a different model name.
> The browser-side key here is fine for a demo but must never ship in a real app;
> proxy it through a server.

To use it on a phone, tunnel the local server so it gets HTTPS:

```bash
cloudflared tunnel --url http://localhost:8000
```

## Controls

| Action | Result |
|---|---|
| **Describe what's ahead** / tap the camera / `Space` | One-shot description of the scene |
| **Hold to ask** | Push-to-talk; ask anything about the current view |
| Quick chips | Read the text · What's in my path · Find the door · Steps or stairs |
| Recent readings | Tap any past reading to hear it again — no extra API call |
| Camera panel | Switch between cameras; **Rescan** picks up a Wi-Fi/virtual camera |

## How it is built for latency

Latency is the whole problem. A person walking cannot wait. The first build took
**46 seconds**; these five changes attack that:

1. **Streamed over SSE** — `streamGenerateContent?alt=sse`, and speech begins at the
   first clause. *Time to first word* is the number that matters, not time to a
   finished sentence.
2. **Smallest capable vision model first** — a 4B model saying "kerb ahead, two steps"
   in 2 s beats a 31B model saying it beautifully in 40 s.
3. **384px image at JPEG q0.6** — image tokens dominate prefill cost.
4. **48 output token cap**, prompt constrained to 15 telegraphic words with the most
   urgent thing in the first three.
5. **A new press aborts the in-flight request**, so nothing queues behind a stale call.

The telemetry panel reports *first word*, *total*, and *image KB* separately, so the
bottleneck is visible rather than guessed at.

The camera preview is a 9:16 portrait column, and capture crops to match it exactly —
what you see is what the model gets. That is also the right crop for walking: ground to
head height in the path, no wasted pixels at the sides.

## Safety stance

A hallucinated "the path is clear" could put someone in traffic. So:

- **The app never asserts safety.** "Safe to cross", "you can go", "path is clear" and
  similar are blocked both by prompt instruction and by a regex filter on the output.
- **It describes, it does not advise.** Output is "kerb ahead, two steps", never "go left".
- **It never claims something is absent**, because the failure mode of a vision model is
  missing things.
- **Road crossings are out of scope.** It will not give a go/no-go on traffic.

**This is an assistive supplement, not a mobility aid.** A cane or guide dog remains
primary. It is not a medical device.

## Honest limitations

- A cloud round trip cannot catch a fast-moving hazard. A real product needs an
  on-device reflex layer (depth + detector, <100 ms, haptics) underneath this one;
  Gemma belongs on the semantic layer where it is unbeatable, not the reflex layer.
- Motion blur while walking degrades accuracy; frames are not yet sharpness-gated.
- Typography is [Atkinson Hyperlegible](https://www.brailleinstitute.org/freefont),
  designed by the Braille Institute for low-vision readers.
- Not yet tested with blind users. Everything about the audio pacing and verbosity is
  a guess until it is.

## Licence

Apache-2.0
