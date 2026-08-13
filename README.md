# Lift Log

A small offline-first workout tracker that does one thing properly: **it tells you when to add weight.**

No account, no server, no analytics, no build step, no dependencies. Six static files, about 48 KB.
Add it to your phone's home screen and it runs like a native app.

---

## Why it exists

Most beginner training programs fail for the same boring reason — nobody writes their numbers down,
so nobody knows whether to add weight this week. Almost every other feature in a fitness app is
downstream of that one habit.

So this is built around a single rule, **double progression**:

> When you hit the top of the rep range on every set, add weight next session.

Log your sets. Hit the top of the range across the board and that exercise turns yellow next
session with the new weight already filled in. That's the whole idea. Everything else is support.

---

## What's in it

**Today** — daily checklist, step / water / protein meters, and a seven-day completion strip. The
strip is the point: individual days don't matter much, the pattern does.

**Train** — alternating full-body A/B sessions. Each exercise expands to show a form diagram (grey
figure is the start position, dark is the finish), three form cues, and swap options for when the
machine you wanted is occupied. Automatic 90-second rest timer with a haptic buzz at the end.

**Progress** — working weight over time for each lift, plus a five-rung ladder for getting your
first pull-up.

**Routines** — a morning and a wind-down list, and the Apple Shortcuts recipes below.

Full JSON export and import, so your data is yours and portable.

---

## Install

It must be served over HTTPS — iOS won't install a home-screen app from `file://`.

**Host it** (any of these, free):

- **GitHub Pages** — Settings → Pages → Deploy from a branch, `main`, `/ (root)`
- **Cloudflare Pages** or **Netlify** — drag the folder onto the dashboard

**iPhone:** open the URL **in Safari** — Chrome on iOS can't install PWAs — then Share → **Add to
Home Screen**.

**Android:** Chrome offers "Install app" in the menu.

**Locally:**

```bash
python3 -m http.server 8000
# http://localhost:8000
```

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app — markup, CSS and JS in one file |
| `manifest.webmanifest` | Name, icons, standalone display |
| `sw.js` | Service worker; caches the shell so it works offline |
| `icon-180/192/512.png` | Home screen icons |

---

## Making it yours

The program is data, not code. Everything lives in objects at the top of the `<script>` block.

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

Worth editing:

- `PROGRAM.A` / `PROGRAM.B` — the two workouts
- `CORE` — the finisher
- `DEFAULT_TASKS`, `MORNING`, `BEDTIME` — the daily and routine lists
- `WARMUP` — the pre-session checklist
- `WATER_TARGET` (cups), `STEP_TARGET`, `TARGET` (total sessions)
- The gym-specific advice block in `vTrain()` — replace with your own or delete it

**After any edit, bump `const CACHE` in `sw.js`.** Otherwise the service worker keeps serving the
old cached version to anyone who already installed it.

### The diagrams

Stick figures generated from joint coordinates — no image assets. A pose is fourteen numbers
(head, neck, hip, knee, ankle, elbow, hand) and each exercise draws a ghost figure at the start
position and a solid one at the finish:

```js
goblet: () => ground()
  + fig(P([96,28, 96,42, 96,80, 96,102, 96,124, 106,58, 106,50]), 1)   // standing (ghost)
  + fig(P([86,46, 88,58, 78,88, 104,100, 98,124, 98,72, 98,64]))       // bottom position
  + bell(98,64) + arrow(130,52,130,86)
```

viewBox is 200×140. Helpers: `ground()`, `bar()`, `pad()`, `bell()`, `arrow()`. Adding an exercise
diagram is one entry in the `DIA` object.

---

## Apple Health, alarms and notifications

| Want | Reality |
|---|---|
| Read steps from Apple Health | No web API exists — HealthKit is native-only. **Shortcuts bridges it.** Works well. |
| Trigger on an alarm | Not directly. A Shortcuts Automation can. |
| Reliable daily reminders | Use the iOS Reminders app. Web Push on iOS isn't dependable enough to build a habit on. |
| Sync with MyFitnessPal / Cronometer | Neither has an open consumer API. Protein is logged manually. |

None of that is a limitation of this app — it's the iOS sandbox. A native app with a paid developer
account could do Health and local notifications properly. Everything else Shortcuts covers free.

### The Shortcuts bridge

The app reads URL parameters, so any Shortcut that opens a URL can write into it.

| Parameter | Effect |
|---|---|
| `?steps=8234` | Sets today's step count |
| `?water=1` | Adds one cup |
| `?protein=120` | Sets today's protein total in grams |
| `?check=creatine` | Ticks a checklist item by its id |
| `?tab=today` | Opens on a given tab |

Parameters are stripped from the address bar once read, so refreshing won't double-count.

#### Importing steps — the version that actually works

Most write-ups of this are wrong. Documented properly because it took a few goes:

1. Shortcuts → **+** → Add Action → search "health" → **Find All Health Samples**
   *(if the action reads "Find Health Samples", tap that wording — it's a toggle, and the two behave differently)*
2. **Type:** `Steps`
3. **Add Filter** → `Start Date` → **is today**. Not a range — "last 7 days" sums the whole week.
   Filtering also stops it churning through your entire Health history.
4. **Group By: Day.** This is what does the summing. There is **no Sum, Statistic or Calculate
   field** on this action, despite what most guides claim. Ignore Unit, Fill Missing, Sort By and
   Limit. You should now get a single number rather than a list of several hundred samples.
5. Add Action → **Text** → type `YOUR-URL?steps=` and tap the Health variable to append it
6. Add Action → **Open URLs** → set its input to the **Text** variable from step 5

**Why the Text action in step 5?** Typing the URL straight into Open URLs throws *"couldn't convert
from rich text to URL"*. Building the string in a Text action first fixes it. If it still fails,
chain a **URL** action between them: Text → URL → Open URLs. Also check there's no space or newline
between `?steps=` and the variable.

**Debugging:** drop a **Quick Look** action anywhere in the shortcut and run it — it shows exactly
what you're holding at that point. A long list means it hasn't summed yet; a single number means
you're set.

**To automate:** Automation tab → **+** → **Time of Day** → 10:00 pm → Daily → your shortcut → turn
off *Ask Before Running*. Note that URL-opening automations may need the phone unlocked to complete,
so check the next morning that the number landed.

#### Important: home-screen apps and Safari don't share storage

On iOS, a web app added to the home screen gets its **own storage container**, isolated from
Safari. A Shortcut that opens a URL opens it in Safari — so the data lands in Safari's copy and the
installed app never sees it. Same URL, separate sets of data. This is an iOS sandbox rule; no
shortcut can work around it.

Your default browser makes no difference — Open URLs goes to Safari or Chrome or whatever else,
and each of those is its own container too. Only **Open App** reaches the installed copy.

So the steps shortcut ends with **Copy to Clipboard**, not Open URLs:

1. **Find All Health Samples** → Type `Steps` → Filter `Start Date is today` → **Group By: Day**
2. Add Action → **Text** → type `steps=` and tap the Health variable to append it
3. Add Action → **Copy to Clipboard** → input the Text variable
4. Add Action → **Open App** → pick **Lift Log** (installed home-screen web apps appear in this list)
5. In the app, tap **↓ Paste from Shortcut** on the Today tab

One extra tap, and it writes to the right place. iOS may show a "Paste" confirmation the first few
times — allow it.

The clipboard payload accepts `steps=8234`, several values joined with `&`
(`steps=8234&water=1`), or just a bare number, which is read as steps.

The `?steps=` URL parameters still work, and are the right approach if you use the app as a Safari
tab rather than installing it.

#### The others

**Water, one tap** — new Shortcut → **Open URLs** → `YOUR-URL?water=1`. No variable, so no rich text
problem. Then map it to **Settings → Accessibility → Touch → Back Tap → Double Tap**. Double-tap the
back of your phone after a glass. Lowest-friction logging there is.

**Open on your alarm** — Automation → **Alarm** → *When alarm is stopped* → Open URL
`YOUR-URL?tab=today`. Your list is the first thing on screen in the morning.

**Bedtime nudge** — Automation → **Time of Day** → **Show Notification**. Deliberately doesn't touch
the app; a plain iOS notification is far more reliable than anything a web app can send.

---

## Your data

Stored in `localStorage` on your device. Nothing leaves your phone — there's no server to send it to
and no third-party scripts on the page.

The trade-off: **iOS can evict web app storage after long periods of non-use**, and it's gone if you
delete the app or clear Safari data. Use **Back up data** on the Routines tab every few weeks; it
downloads a JSON file that **Restore** reads back.

---

## Deliberate omissions

- **No calorie tracking.** Calories need a real food database to be more than guesswork. Use a
  dedicated app. Mixing food logging into a training log tends to turn it into a weight-watching
  app, and session count is the number you actually control.
- **No daily weigh-ins.** Bodyweight is asked once, only to compute a protein target.
- **No punishing streaks.** The seven-day strip shows what happened; it doesn't reset to zero and
  shame you for a missed Tuesday.

---

## Not medical or coaching advice

A personal project built around a beginner's general fitness program. It isn't professional coaching
and it isn't medical advice. If you're new to lifting, get someone qualified to watch your first few
sessions — squats and Romanian deadlifts especially. If you have an existing injury or health
condition, speak to a doctor or physio before starting a new program.

## Licence

MIT. Fork it, rewrite the program, strip out what you don't need.
