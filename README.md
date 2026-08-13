# lift-log
# Lift Log

A small offline-first workout tracker that does one thing properly: **it tells you when to add weight.**

No account, no server, no analytics, no build step. Six static files. Add it to your phone's
home screen and it runs like an app.

---

## Why it exists

Most beginner training programs fail for the same boring reason — nobody writes their numbers
down, so nobody knows whether to add weight this week. Every other feature in a fitness app is
downstream of that one habit.

So this app is built around a single rule, called **double progression**:

> When you hit the top of the rep range on every set, add weight next session.

Log your sets. If you hit the top of the range across the board, that exercise turns yellow next
time with the new weight already filled in. That's the whole idea.

---

## What's in it

**Today** — daily checklist, step / water / protein meters, and a seven-day completion strip.
The strip is the point: individual days don't matter, the pattern does.

**Train** — alternating full-body A/B sessions. Each exercise expands to show a form diagram
(grey figure is the start, dark is the finish), three form cues, and swap options for when the
machine you wanted is occupied. Automatic 90-second rest timer with a haptic buzz at the end.

**Progress** — working weight over time for each lift, plus a five-rung progression ladder for
getting your first pull-up.

**Routines** — a morning and a wind-down list, and the Apple Shortcuts recipes below.

Also: full JSON export and import, so your data is yours and portable.

---

## Install

It needs to be served over HTTPS — iOS won't install a home-screen app from `file://`.

**Host it** (any of these, all free):
- GitHub Pages — Settings → Pages → Deploy from branch, `main`, `/ (root)`
- Cloudflare Pages or Netlify — drag the folder onto the dashboard

**Install it on iPhone:** open the URL **in Safari** (not Chrome — Chrome on iOS can't install
PWAs) → Share → **Add to Home Screen**.

**On Android:** Chrome will offer "Install app" in the menu.

**Locally, to hack on it:**

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app — markup, CSS and JS in one file, no dependencies |
| `manifest.webmanifest` | Name, icons, standalone display mode |
| `sw.js` | Service worker; caches the shell so it works offline |
| `icon-180/192/512.png` | Home screen icons |

Roughly 48 KB total. It loads instantly and works with no signal.

---

## Making it yours

The program is data, not code. Everything lives in objects at the top of the `<script>` block
in `index.html`.

```js
{
  id: "legpress",          // unique — history is keyed on this, don't change it later
  dia: "legpress",         // which diagram to draw
  name: "Leg press",
  sets: 3, lo: 10, hi: 12, // hit `hi` on every set → it tells you to add weight
  inc: 5,                  // kg added when you earn it
  unit: "kg",              // "kg" | "sec" (timed holds) | "bw" (bodyweight reps)
  cues: ["...", "...", "..."],
  swaps: [{ id, name, dia, why }]   // alternatives when equipment is taken
}
```

Things you'll probably want to edit:

- `PROGRAM.A` / `PROGRAM.B` — the two workouts
- `CORE` — the finisher
- `DEFAULT_TASKS`, `MORNING`, `BEDTIME` — the daily and routine lists
- `WARMUP` — the pre-session checklist
- `WATER_TARGET`, `STEP_TARGET`, `TARGET` (total sessions)
- The "Northbridge notes" block in `vTrain()` — gym-specific advice you'll want to replace

**After editing, bump `const CACHE = "liftlog-v2"` in `sw.js`.** Otherwise the service worker
keeps serving the old cached version to anyone who already installed it.

### Diagrams

The form diagrams are stick figures generated from joint coordinates, not images. A pose is
fourteen numbers — head, neck, hip, knee, ankle, elbow, hand — and each exercise draws a ghost
figure at the start position and a solid one at the finish:

```js
goblet: () => ground()
  + fig(P([96,28, 96,42, 96,80, 96,102, 96,124, 106,58, 106,50]), 1)   // standing (ghost)
  + fig(P([86,46, 88,58, 78,88, 104,100, 98,124, 98,72, 98,64]))       // bottom position
  + bell(98,64) + arrow(130,52,130,86)
```

The viewBox is 200×140. Helpers available: `ground()`, `bar()`, `pad()`, `bell()`, `arrow()`.
Adding an exercise diagram means adding one entry to the `DIA` object.

---

## Apple Health, alarms and notifications

Worth being straight about this, because it's the first thing people ask.

| Want | Status |
|---|---|
| Read step count from Apple Health | **Not possible for a web app.** HealthKit is native-only. Workaround below — it works well. |
| Trigger on an alarm | Not directly. Shortcuts Automation can do it. |
| Reliable daily reminders | Use the iOS Reminders app. Web Push on iOS isn't dependable enough to build a habit on. |
| Sync with MyFitnessPal / Cronometer | No open consumer API on either. Protein is logged manually. |

None of that is a shortcoming of this app — it's the iOS sandbox. A native app with a paid
developer account could do Health and local notifications properly. Everything else is
covered for free by Shortcuts.

### The Shortcuts bridge

The app reads URL parameters, so any Shortcut that opens a URL can write into it.

| Parameter | Effect |
|---|---|
| `?steps=8234` | Sets today's step count |
| `?water=1` | Adds one cup |
| `?protein=120` | Sets today's protein total (grams) |
| `?check=creatine` | Ticks a checklist item by its id |
| `?tab=today` | Opens on a given tab |

Parameters are stripped from the URL once read, so refreshing won't double-count.

**Import steps from Health:** Shortcuts → new Shortcut → **Find Health Samples** (Steps,
Statistic: Sum, Range: Today) → **Open URLs** → `YOUR-URL?steps=` + the Health variable. Set a
Time of Day automation for the evening and forget about it.

**On alarm:** Automations → **Alarm** → *When alarm is stopped* → Open URL `YOUR-URL?tab=today`.

**One-tap water:** a Shortcut that opens `YOUR-URL?water=1`, added to your home screen or
triggered with Siri.

---

## Your data

Stored in `localStorage` on your device. Nothing leaves your phone — there's no server to send
it to and no third-party scripts on the page.

The trade-off: **iOS can evict web app storage after long periods of non-use**, and it's gone if
you delete the app or clear Safari data. Use **Back up data** on the Routines tab every few
weeks; it downloads a JSON file that **Restore** reads back.

---

## Deliberate omissions

- **No calorie tracking.** Calories need a real food database to be anything other than
  guesswork. Use a dedicated app. Mixing food logging into a training log tends to turn it into
  a weight-watching app, and session count is the number you actually control.
- **No daily weigh-ins.** Bodyweight is asked once, only to compute a protein target.
- **No streaks that punish you.** The seven-day strip shows what happened; it doesn't reset to
  zero and shame you for a missed Tuesday.

---

## Not medical or coaching advice

This is a personal project, written for a beginner's general fitness program. It isn't
professional coaching and it isn't medical advice. If you're new to lifting, get someone
qualified to watch your first few sessions — squats and Romanian deadlifts in particular. If
you have an existing injury or health condition, talk to a doctor or physio before starting a
new program.

## Licence

MIT — do whatever you like with it. Fork it, rewrite the program, strip out the parts you don't
need.
