# Untitled Stay-at-Home Dad Roguelike
## Design Doc v0.1

**Author:** Morgan (Pickle Moon)
**Status:** Pitch / pre-production
**Engine:** Godot (latest stable)

---

## 1. Pitch

A first-person domestic roguelike about getting through a week as a stay-at-home dad. You shop, cook, clean, and try to read your partner — who you never see — through the cues she leaves and the calls she makes. You aren't really supposed to succeed. You're supposed to get through, mostly, by the skin of your teeth, and to feel the texture of a life that asks small acts of attention from you every day.

It's cosy in setting and pace. It's a roguelike in structure. The dissonance is the point.

---

## 2. Tone & Theme

**Mood:** Wistful. Poignant. Moody but never bleak. Late-afternoon light through a kitchen window. The hum of a fridge. A child laughing in the next room. The relief of a clean sink and the dread of tomorrow's school run.

**Core theme:** *Care is paying attention.* The game is fundamentally about noticing — your partner's mood, what's running low, what's on sale, what your kid hasn't eaten in three days. Success is a function of attentiveness, not optimisation.

**What it is not:**
- It is not a parenting simulator. The child is felt, not played with.
- It is not a relationship-management dating sim. Your partner is a person, not a meter.
- It is not Overcooked. There is no timer screaming at you. Pressure is ambient, not mechanical.
- It is not bleak. No one dies. No one leaves. The stakes are dignity, attentiveness, and the quiet pride of a well-run home.

---

## 3. Loop & Structure

### Run length
**One in-game week (7 days).** A run is ~45-75 minutes. Each day is a vignette of 5-10 minutes with planning beats in between.

### The arc of a run
- **Mon–Tue:** Things go well. You have a full pantry, a clean kitchen, a clear head. You're establishing what your partner needs this week.
- **Wed–Thu:** Cracks appear. The shop you planned around is closed for renovation. Your kid won't eat the thing you prepped. You start improvising.
- **Fri–Sat:** You're cooking from leftovers and half-bags of things. You're tired. The signals from your partner are getting harder to read because *you're* fogged.
- **Sun:** A reckoning. You either close the week with grace or you limp across the line. Either way, a new week begins, modified by what you carried over.

You are **meant to slowly lose ground.** A perfect week should feel rare and earned. A "good enough" week is the realistic target. A bad week is still completable and still has moments worth having.

### Daily structure
Each in-game day breaks into rhythm beats. The player doesn't experience these as discrete "phases" — they flow:

1. **Morning** — wake up (with or without hangover), see what state the kitchen is in, hear what the partner is doing before she leaves
2. **Mid-morning** — plan, look at the fridge, check the pantry, decide whether you need to shop
3. **The call** — partner calls from work, roughly the same time each day, never exactly. She tells you (or doesn't) what she wants for dinner. You read her voice.
4. **The errand** — shopping, or not, depending on the day
5. **Prep & cook** — meal prep and cooking are separate activities, often on different days
6. **Dinner** — handled mostly in dialogue/sound, not gameplay. You serve, she eats.
7. **Dishes & evening** — wind-down. Sometimes she joins you in the kitchen. Sometimes she's on the couch with the TV.

---

## 4. Perspective & Presentation

**First-person throughout.** No avatar. You are a pair of hands and a point of view.

Top-down only as a **stylised map screen** for choosing where to drive — a folded paper map aesthetic, not a playable overworld. Selecting a destination cuts to first-person inside that location.

**Why first-person:**
- Solves the solo-dev pixel art bottleneck. You render spaces and objects, not characters with eight directions of animation.
- The partner's absence becomes a design feature, not a workaround. You see her wine glass, her keys, her note on the bench — never her.
- The child works the same way: a toy on the floor, a small voice from down the hall, a hand entering frame.
- Forces the player to *look* at shelves, fridges, lists. Attention is the gameplay.

**Visual style:** Pixel art at a chunky resolution (think *Cloud Gardens* / *Norco* density rather than tiny 16×16 sprites). High-contrast, warm lighting. Heavy use of time-of-day lighting tints — morning blue, afternoon amber, evening warm-dark — to do a lot of mood work for free.

---

## 5. Subsystems

A deliberately small number of systems, designed to interact richly. Resist the temptation to add more.

### 5.1 The Partner

She is the central NPC and you never see her. She is expressed through:
- **The daily call.** Voice acting if budget allows, otherwise stylised text with audio cues (breathing, background office noise, traffic). Her tone, word choice, and what she *doesn't* say carry information.
- **Environmental traces.** Glass left on the side table. Book face-down on a different chair. Painkillers out on the bench. The robe she wears when she's cold-coming-down-with-something.
- **Texts during the day.** Short, ambient. "ugh meeting just ended" or "can you pick up tampons" or nothing for hours.
- **Evening presence.** Heard from the next room or felt next to you doing dishes. Dialogue, not visual.

She has hidden state the player must infer: energy, mood, cycle phase, what kind of week she's having at work, what she's been craving. None of this is shown on a UI. The game rewards correctly reading her with score (more below), but never tells you what you read correctly until end-of-week.

**Critical design rule:** the partner is never punishing. She doesn't get angry when you miss a cue. She gets quietly disappointed, or she doesn't notice, or she takes care of herself. The penalty for missing her is *the player feeling the miss*, not a stat decrease cutscene.

### 5.2 Shopping

You drive to shops (map screen → first-person inside). Several shops in the area, each with personality:
- The big supermarket (everything, but ugly fluorescent lighting and crowds)
- The local IGA-equivalent (limited, but you know the staff)
- The butcher
- The greengrocer
- The bottle shop
- The 24-hour servo (last resort, expensive, limited)

**Layouts shift between runs.** Not randomised chaos — each shop has a stable identity, but aisles rearrange, products move, the bread is where the cereal used to be. You will think you know a shop and then have to relearn it. This is the "just as you've figured the layout out" feeling, encoded.

**Closures and disruptions** are rolled per-run. Butcher closed Tuesday. Greengrocer has no leafy greens this week (supply issue). Big supermarket has half its lights out and the deli closed early.

**Sales and stockouts** rotate. A great deal on lamb mince on Thursday is meaningless if you don't have a plan for it. A meal you'd planned around chickpeas dies when the shelf is empty.

**Side encounters.** Other shoppers, staff, an old friend from before the kid. These are interruptions — sometimes welcome, sometimes derailing. They eat time and attention. Occasionally you drop your list (literally), or forget what you came for. Pacing rule: at most one significant side encounter per shopping trip.

### 5.3 Cooking (Prep + Active Cook)

Two distinct activities.

**Prep** is reading, choosing, chopping, marinating. Done in a calmer headspace, usually mid-day. First-person at the counter. Tactile — you pick up a knife, you turn an onion.

**Active cook** is heat, timing, attention. Multiple things happening on the stove. Not Overcooked-frantic — more like the satisfying-but-real attention of actually cooking. The phone might ring during this. The kid might need something.

**Recipes are not given.** You learn what your partner likes by watching what she finishes and what she leaves. Over many runs (meta-progression), you accrue a personal recipe book of *things that worked for this household.*

### 5.4 Leftovers, Pantry, Cross-meal economy

Critical to the difficulty curve. Early week: you cook fresh from full ingredients. Late week: you're making something credible from half a bag of rice, the back half of a roast, and the wilting end of a bunch of coriander.

**The pantry is a real space**, viewed in first-person. You open it and look. Things are at the back. The brown sugar is behind the chickpeas. This is texture, not friction — keep it readable.

Leftovers and unused ingredients should *enable* late-week play, not just be a resource grind. The cosy pleasure of Friday-night-fridge-archaeology is part of the design.

### 5.5 Dishes & wind-down

Dishes are gameplay but low-stakes. A rhythm activity, almost meditative. The interesting variable is whether the partner joins you.

**Solo dishes:** quiet, reflective, ambient music, maybe an internal monologue beat or a memory.

**Partner joins:** dialogue. She's debriefing her day. You're listening. The "game" is the listening — small choices about how you respond, which can reinforce or undermine the reading you've been doing all day.

This is where a lot of the wistful tone lives. Don't underbuild it because it sounds simple.

### 5.6 Drinking

Wine with dinner is a relationship beat. Noticing she wants one and pouring it without being asked is a positive read. Pouring one when she's been off it for a few days is a miss.

You drink too. Sometimes she opens a second bottle. Sometimes you match her, sometimes you don't. Drinking more than you should:
- Slightly fogs the next morning's perception (text and audio cues from the partner are subtler, harder to read)
- Slows prep and dishes
- Is not framed as failure. It's framed as a Tuesday.

### 5.7 The Child

Mechanically minimal. Present as:
- Background sound (laughter, complaining, the iPad)
- Interruptions to other activities (rare, weighted, never punishing — a 30-second beat, not a stop)
- A constraint on the day (school pickup time is a hard event you must respect)
- Story beats that shift the day (sick day, school excursion, lost something)

**You do not interact with the child mechanically.** No feeding minigame. No play minigame. The child exists to constrain and texture the day, not to add a system.

---

## 6. Scoring & Feedback

The end of each in-game day shows a **brief, ambient summary** — not a stat sheet. Three or four lines of dialogue or observation:

> *She had a second glass of wine and didn't reach for her phone.*
> *The kid ate everything on the plate.*
> *You went to bed with a clean sink.*

Or:

> *She picked at dinner.*
> *You forgot to defrost the chicken for tomorrow.*
> *The recycling didn't go out.*

End-of-week is a slightly longer reflection. A score exists under the hood for run-grading and meta-progression unlocks, but it is not shown as a number. The player should *feel* how the week went.

---

## 7. Meta-progression

Across runs:

- **Recipe book** — successful meals get noted. You can re-cook them faster and more reliably.
- **Knowledge of the partner** — over many runs, you build intuition. The game doesn't surface this as a stat, but the player gets better at reading her. (This is the deep replay hook.)
- **Shop familiarity** — even though layouts shift, you learn each shop's character.
- **House upgrades** — sparingly. A better knife. A second pot. A whiteboard on the fridge. Earned through good weeks, not bought.

**No skill tree, no XP bar.** Progress is felt, not displayed.

---

## 8. The Difficulty Curve

Per-run curve is described in §3. Across runs, difficulty does **not** ratchet up indefinitely. Instead, later runs introduce more **variance** — bigger swings, weirder weeks, more unusual partner states (a friend visiting, a stomach bug going through the house, an anniversary). The game gets less predictable, not harder.

Failure state: there is no game over. A bad week ends, a new week begins. Truly catastrophic weeks (you didn't cook a single proper meal, you forgot the school pickup, you drank every night) are noted but not punished beyond the natural consequence of the next week starting depleted.

---

## 9. Scope & Solo-Dev Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| Pixel art volume | **High** | First-person perspective. No character animation. Heavy asset reuse across shops/rooms. Limited NPC roster. |
| Subsystem creep | **High** | Hard cap at the systems in §5. New ideas go in a backlog, not the build. |
| Writing volume (partner dialogue) | **Medium-High** | Modular dialogue snippets recombined by mood/topic, not hand-authored per day. Voice acting only if a clear win. |
| Tone collapse (cosy ↔ stressful) | **Medium** | Playtest the *feel* early. Pull stress levers down before pulling cosy levers up. |
| Roguelike variety vs handcrafted feel | **Medium** | Authored shop identities + procedural layout drift. Authored partner archetypes + procedural week-state. |
| Player not understanding the "no UI" reading layer | **Medium** | Tutorial week with slightly heavier cues, then taper. End-of-week reflection makes hidden reads legible after the fact. |

---

## 10. Open Questions

1. **Voice acting for the partner?** Huge tonal lift if done well, catastrophic if done poorly. Probably skip for v1, design dialogue to read well silently.
2. **How much authored narrative vs procedural?** Lean procedural for replay, but a few authored week-arcs (anniversary week, sick-kid week) might be worth the cost.
3. **Platform-first decision.** PC-first via Steam (matches the audience for this kind of game), with mobile as a possible later port. Touch controls would change a lot of the cooking/shopping interactions.
4. **Length of the run.** 45-75 min is my estimate. Needs prototype validation. If a run is too long, the roguelike replay loop breaks.
5. **What does the title do?** Not yet named. A working title would help. Something domestic, slightly off-kilter.

---

## 11. Next Steps

1. **Vertical slice prototype:** one shop (first-person navigation + shelf-reading), one cooking sequence (prep + active), one phone call, one end-of-day reflection. No procedural systems yet. Hand-author everything to validate the *feel*.
2. **Tone playtest:** the vertical slice with three or four people, not asking "is it fun" but "how did it make you feel." If "wistful" doesn't come up at least once, the doc is wrong, not the build.
3. **Art style test:** mock up a single kitchen counter scene at target fidelity. Estimate hours. Multiply by the total scene count. If the number is insane, reduce scene count, not fidelity.
4. **Then** build the procedural layer.

---

## Appendix A: Morgan's stream-of-consciousness notes (raw)

*Unedited. Captured for later integration.*

- meal variety — is it important? do you get a different SORT of partner each run?
- recipe book each run is great. it's your 'power up over time' mechanic, speeding things up considerably
- week-open inventory semi-random(?) to give structure to bounce off of, and queue certain events throughout the shape of the run
- you can drop things you need for the meal — think spilled milk — and have to either race off to the servo to see if they have, or improvise. this shouldn't make perfect score impossible, just difficult
- the hangover feels core. your partner can be like "um, I'm talking to you?!" but in game there has been no sign — you literally didn't hear them when they were talking to you
- partner texting you throughout the day is great
- distractions while cooking. script the child to be like LOOK AT ME and they're juggling or something while your X is close to being ready or boiled or whatever, a crucial point you should be paying attention during. a phone call from their boss etc. DAD HELP I'VE HURT MY KNEE
- ordering in absolutely an option, needs to make sense financially
- meal PREP is an option when you have time — like full meals you can throw in the freezer and bust out later
- on art: first person yes, but more thinking pixel art mainly, hand-drawn nods. or actually — pixel art gameplay but you DO EVENTUALLY get rewarded with cutscenes of your partner, happily smiling at you over the counter, or disappointedly calling it a night because they have to be up early
- environmental traces of the partner — the house should feel alive in a nice way, yet you do have gameplay beats where you have to clean their stuff up. things should be persistent in a way that's fun and thoughtful. you guys have a couple drinks? they're there to be tidied up in the morning
- hidden state is correct, inferred is correct. subtle hints and that sort of thing
- driving to the shops feels good — more opportunities for things to go wrong. detours, traffic, crashes, car won't start
- side quests are always random but not always detrimental. they aren't daily necessarily, so they should be a surprise when you get them. they're distractions! that is the core of the game — giving you something to do and making you steer away or choose not to. either way it's very distracting and hard to focus on your current goal
- leftover casserole could simplify leftovers substantially. possibly too much, but it could give design respite
- interacting with the child, while difficult and straining your timetable, rewards you in other ways. gives the partner a boost or something. tangential but tangible. not direct, but obvious to the alert
- not every child-related beat is important enough to do. scattering them in when there's time is beneficial. do too many and you run out of time to do things in order and properly
- whiteboard is good. maybe a notebook too. backups for if your phone dies and you can't access your recipes or shopping lists or something
- upgrades that are diegetic and improve gameplay without being numerical
- difficulty curve needs some tuning
