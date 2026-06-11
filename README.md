# 🍉 Fruit Party!

A Fruit Ninja–style browser game for little kids. Fruit floats up slowly, you slice it by
**waving your hand at the camera** (or just touching the screen). No bombs, no lives, no
game over — a 2-year-old waves and fruit explodes. Big celebration every 10 fruits.

Everything is one file: [index.html](index.html). No build step, no install, no server code.

---

## Put it online (GitHub Pages — free, ~5 minutes)

The camera **requires HTTPS**, which GitHub Pages gives you automatically.

1. Go to <https://github.com/new>. Repository name: `fruit-party`. Keep it **Public**. Click **Create repository**.
2. On the new repo page, click **uploading an existing file**, drag `index.html` in, click **Commit changes**.
   *(Or from this folder: `git remote add origin https://github.com/YOURNAME/fruit-party.git && git push -u origin main`)*
3. In the repo: **Settings → Pages** (left sidebar). Under **Branch** pick `main`, folder `/ (root)`, click **Save**.
4. Wait 1–2 minutes. Your game is at:
   **`https://YOURNAME.github.io/fruit-party/`**

Any other static host (Netlify Drop, Cloudflare Pages, Vercel) works the same — drop the
file in, get an HTTPS URL.

## Open it on the iPad

1. Open **Safari** and go to your `https://...github.io/fruit-party/` URL.
2. Tap **📷 Start (with camera)** → tap **Allow** when Safari asks for the camera.
3. Wave! (Optional: Share button → **Add to Home Screen** makes a full-screen app icon.)

First load needs internet (the hand-tracking model, ~8 MB, comes from a CDN — it's cached
after that). Sound starts after the first tap — that's an iOS rule, the Start button counts.

### If something is off

| Symptom | Fix |
|---|---|
| Camera prompt never appears / always denied | iPad **Settings → Apps → Safari → Camera → Ask/Allow**, then reload. Or in Safari tap **aA → Website Settings → Camera → Allow**. |
| "Camera needs a secure (https) page" | You opened it via `http://` or a file. Use the GitHub Pages `https://` URL. |
| No sound | Tap the screen once; check the side switch / volume; iPad Silent mode mutes it. |
| Hands feel laggy | The game lowers its own tracking rate and resolution first, and if the device truly can't keep up it switches to touch by itself. Touch always works. |
| Hand not detected | Sit about an arm's length away, palm facing the camera, decent light. |

## Desktop

Chrome, Safari, or Edge — same URL. Mouse slices just by moving (no clicking needed).
For local development: `python3 -m http.server 8000` in this folder, then
<http://localhost:8000> (`localhost` is allowed to use the camera without HTTPS).

## Playing together on a video call (optional)

Tap **⚙️ → Play together**: both devices enter a nickname and the **same room code**, tap
**Join room**. Each player's name and live score appears top-left on both screens.

How it works, honestly: scores travel through a **free public MQTT relay**
(broker.emqx.io, with two fallbacks) over secure WebSockets — no account and no server,
but it's shared public infrastructure:

- Only the room code, nickname, and score are ever sent. Use first names or nicknames.
- Pick a **unique silly code** (`BANANA-MOON-42`), since anyone using the same code lands
  in the same room.
- If the relay is down, the game says so and plays on solo — multiplayer can never break
  the game.
- To remove the feature entirely, set `ROOM_ENABLED = false` near the bottom of
  [index.html](index.html).

## Settings

⚙️ gear → one slider: **fruit speed** (🐢…🐇), saved on the device. Default is toddler-slow.

## How the code works (quick tour)

All in [index.html](index.html), top to bottom:

- **Sounds** — synthesized with WebAudio (no audio files): pop = pitch-drop square wave +
  filtered noise burst; quick combos pitch up; cheer = little arpeggio.
- **Fruit** — emoji drawn on a canvas (≥120 px). Each fruit picks where its arc should
  peak and how long it should take, and launch speed/gravity are derived from that, so
  the speed slider scales everything consistently.
- **Slicing / collision** — every input (each finger, mouse, each tracked hand) keeps a
  ~0.3 s trail of points. A fruit pops when the **segment between the last two points**
  passes within its radius (so fast swipes can't tunnel through), *or* when the newest
  point rests inside it (so a toddler holding a finger still pops fruit drifting by).
- **Hand tracking** — MediaPipe HandLandmarker (tasks-vision, pinned CDN version) watches
  the 640×480 selfie video and reports 21 landmarks per hand; we use #8, the index
  fingertip. Detection runs at ~15 Hz, and the blade glides toward the newest fingertip
  every frame, so the trail stays smooth between detections. Coordinates are mapped
  through the same object-fit:cover math the video uses, with X flipped for mirroring.
- **Fallback ladder** — camera denied → gradient background + touch; MediaPipe fails or
  times out → camera background + touch; detection averaging too slow → halves its rate,
  then disables itself. Touch/mouse slicing is always on, so the game is never broken.
- **Auto quality** — if fps sags, the canvas renders at a lower resolution and the fruit
  cap drops before anything else suffers.

## Milestone test log

Tested in a desktop browser preview (Chromium). **Not yet run on a physical iPad — do the
2-minute checklist below before game time.**

- **M1 touch game**: synthetic swipe slices fruit (score increments); resting-point slice
  works; celebration banner at every 10; speed slider updates + persists; portrait resize
  correct; no console errors.
- **M2 hand tracking**: permission-denial path shows friendly message and keeps playing;
  with a fake camera stream MediaPipe loads from CDN, reaches "wave your hand" state, and
  stays healthy; coordinate mapping verified (camera center → screen center, mirror
  correct); under artificially slow software-GL the perf guard correctly self-disabled to
  touch. Real-hand-in-real-camera verified only by the math + pipeline, not a human hand.
- **M3 polish**: praise banner text, spawn boop, particle cap under rapid slicing.
- **M4 room**: joined a real public relay from the preview; second client appeared on the
  board with name + score; peer received local score updates; leave cleared everything;
  failure path falls back to a friendly message by construction.

### Known issues / honest caveats

- **Physical-iPad smoke test pending** (needs a real device): camera permission flow,
  actual hand-tracking fps, audio after first tap. Checklist: load URL → Start with
  camera → Allow → wave → fruit pops → tap ⚙️ → slider moves.
- Hand identity can swap when two hands cross — a trail may briefly jump. Cosmetic.
- Very old iPads (pre-2017, no decent WebGL) will land in touch mode by design.
- Public MQTT relays carry no uptime promise; the room may occasionally be unavailable.
- The score resets on reload — by design, it just counts up forever within a session.
