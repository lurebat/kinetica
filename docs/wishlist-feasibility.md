# Wishlist feasibility review

A pass over a user wishlist: for each item, how Kinetica behaves today, why the user sees what
they see, a basic design, a complexity estimate, and what wmkeyboard and LeanType do.

Line numbers point at the commit this review was written against and drift as code moves; the
named functions are the stable reference.

**Complexity scale.** **S**: one file or one setting, a day or less. **M**: several files,
a new setting and tests, a few days. **L**: new subsystem, persistence/migration, or touch-model
change; a week or more.

**Reference keyboards.**
[wmkeyboard](https://github.com/wasi-master/wmkeyboard) (Kotlin/Compose, own gesture engine, JSON
layouts, typed `KeyAction` model) and [LeanType](https://github.com/LeanBitLab/LeanType)
(HeliBoard/AOSP LatinIME fork: `PointerTracker`, `BatchInputArbiter`, integer `KeyCode` actions).

## Summary

| # | Item | Today | Complexity | Verdict |
|---|------|-------|-----------|---------|
| 1 | Hebrew layout with English backspace position | Hebrew backspace is top-right by design; one layout file per language | **S** (change it) / **M** (per-language variants) | Feasible |
| 2 | Consistent key bindings everywhere | Seven binding surfaces, five storage formats, three spellings per action | **S** quick fixes / **M** registry façade + text format / **L** storage unification | Feasible; phased |
| 3 | Autospace timer visualisation | Timer exists; nothing shows it | **M** | Feasible |
| 4 | Language switch key or gesture | Action exists (toolbar, chords); no key, no swipe | **S** gesture / **M** key | Feasible |
| 5 | Space after punctuation | Not an option; punctuation hugs the previous word | **M** | Feasible with care |
| 6 | Rest a finger on a key while typing | Hold timer fires on the resting finger even mid-swipe; its tap can join the word | **S** (no holds mid-swipe) / **M** (resting mode) | Feasible; one product decision |
| 7 | Reverse suggestion wheel direction | Hard-coded mapping | **S** (opt-in setting) | Trivial |
| 8 | Spacebar chords don't work | **Resolved**: they work (user error) | — | Discoverability only |
| 9 | Number row cancels glides; numbers in words | Likely the top-row up-flick firing; digits split words | **S** fix / **M** digits join words / **L** glide digits | Feasible; trace to confirm |
| 10 | Show chords when holding `?123`/space | Not shown | **M** | Feasible |
| 11 | Emoji search (and GIFs) | Data already shipped; no search UI; app is network-free | **M** emoji / **L** GIFs (network, compliance) | Emoji yes; GIFs opt-in only |
| 12 | Downrank or banish suggestions | Block and slide-to-adjust exist but are hidden; no real downrank | **M** (hold menu + penalty) | Feasible |
| 13 | Long-press shortcuts on number keys | Number keys are tap-only | **M** | Feasible |
| 14 | Long-press toolbar keys | Toolbar is tap-only; arranged only in settings | **M** | Feasible; edit-mode policy |
| 15 | Clipboard history (pins, images, codes) | Doesn't exist | **M** text / **L** all phases | Feasible; privacy-first |

Suggested order by value per effort: 7, 6 (phase 1), 9 (fix), 2 (quick fixes), 1, 4, 12, 14, 3,
15 (phase 1), 2 (registry), 13, 10, 11 (emoji), 9 (digits in words), 6 (phase 2), 5, 15 (later
phases), 11 (GIFs). Item 15 phase 2 needs item 11's search capture buffer.

---

## 1. Hebrew layout with backspace where English has it

**Today.** Hebrew uses its own asset, `app/src/main/assets/layouts/native_he.json`, chosen ahead of
any arrangement by `AlphaLayouts.name()` (`native_<lang>` wins). Its backspace is the last key of the
**top** row (`x=0.80, y=0.0, w=0.20`, line 17). English `qwerty.json` puts it at the right of the
**bottom letter row** (`x=0.85, y=0.50, w=0.15`, line 35). The board is `"fixedArrangement": true`,
and no mutation in `LayoutMutations` relocates special keys.

**Is there already a multiple-layouts-per-language system?** Partly. `AlphaLayouts.name()` resolves
in this order:

1. `native_<lang>`: if it exists, it **always** wins.
2. `<arrangement>_<lang>`: `arrangement` is the **global** `pref_key_arrangement`
   (`qwerty | qwertz | qzerty | azerty`). Today only `azerty_fr.json` uses this.
3. `qwerty_<lang>`, then `qwerty`.

QWERTZ and QZERTY aren't files; `LayoutMutations.withLetterArrangement()` swaps a letter pair at build
time, and `fixedArrangement: true` opts a layout out. `KineticaIME.arrangementOnLanguageChange()`
auto-switches the global setting to AZERTY when entering French and back when leaving it.

So there is variant lookup, but it is keyed by one global Latin setting, and native-script languages
(he, ar, ru, uk) skip it entirely. A Hebrew variant needs step 1 to become variant-aware.

**Design options.**

- **A. Edit `native_he.json`** to the SI-1452 shape, with backspace at the right of the bottom
  letter row. This matches wmkeyboard's
  [`he_si1452.wmlayout.json`](https://github.com/wasi-master/wmkeyboard/blob/main/app/src/main/assets/layouts/he_si1452.wmlayout.json#L116-L164),
  which puts shift bottom-left and delete bottom-right as on QWERTY. Changes everyone's muscle memory.
- **B. Per-language layout variants (recommended).** Ship `native_he_si1452.json` beside
  `native_he.json`. Add a **per-language** variant preference (e.g. `pref_layout_variant_<lang>`)
  that `AlphaLayouts.name()` tries first as `<base>_<variant>`. Settings lists the variants found in
  bundled assets for the current language, so adding a variant is adding a file. This generalises
  cleanly: the global arrangement setting could later become the default variant for Latin languages,
  and the French AZERTY auto-switch becomes a per-language default instead of a global write that has
  to be undone. LeanType does the same (`hebrew.json` and `hebrew_1452_2.json`, selected per
  subtype).
- **C. A generic `withBackspacePlacement()` mutation.** Not recommended: moving one key means
  reflowing the row it leaves and the row it joins, which is layout-specific.

**Complexity.** A: **S**. B: **M** (resolution order, per-language preference, settings row,
backup, the bundled-layout test listed in `ADDING_A_LANGUAGE.md`). Folding the global arrangement
into the variants as well is a further **M**, mainly in migration.

**Gotchas.** Keep the key id `backspace`: edge-swipe bindings are keyed by id
(`EdgeSwipeBindings`). Coordinates are not mirrored for RTL, so the asset says exactly where the key
goes. Decoder geometry is derived from the layout, so the decoder needs no change, but letter
positions move and that should be sanity-checked on real swipes.

---

## 2. Key bindings are inconsistent

**Today the four main binding surfaces each have their own action set:**

| Surface | Where | Accepts |
|---|---|---|
| Chords (`?123`/space + key) | `ChordSettingsActivity`, `ShortcutKinds.chords` | text, Ctrl combos, **every** `EditorAction` (stored as `action:<name>`) |
| Edge swipes | `EdgeSwipeSettingsActivity`, `ShortcutKinds.edgeSwipes` | text, actions, Ctrl combos, `emoji` |
| Letter long-press lists | `AlphaLayouts.build()` → `ActionRow.glyphForToken()` | text, plus `:name` tokens or a toolbar glyph, **minus** `ActionRow.NOT_IN_LISTS` |
| Comma / period key | `LayoutMutations.withPunctuationKey()` | a fixed menu: `keep / remove / char / text / paste / select_all` |

**Why the user sees it.**

- **Switch language in letter lists.** `NEXT_LANGUAGE` is **not** in `NOT_IN_LISTS`, so
  `:next_language` (and the `⇄` glyph) should already parse. The help text
  (`pref_letter_alternates_note_summary`, `res/values/strings.xml:380`) just doesn't list it, nor
  `:one_handed`, `:forward_delete`, `:page_up` and other names that do parse. Not yet verified on a
  device.
- **`:select_all` in letter lists** is rejected (it's in `NOT_IN_LISTS`, because its glyph `ALL` is
  plausible text), even though the KDoc on `glyphForToken` uses it as the example. The `:name` form
  could accept it: the ambiguity is only with the glyph, not the name.
- **Comma/period** never go through the action parser. Their menu offers two actions; `text` mode
  inserts the string literally, so `:next_language` there types `:next_language`.

**The full inventory is wider than four.** Counting everything a user can bind, there are seven
places, stored in five different ways:

| Surface | Storage | Output encoding |
|---|---|---|
| Chords | Room `chord_shortcuts(chord, expansion)` | `ShortcutKinds`: text, `combo:ctrl+a`, `action:name` |
| Edge swipes | pref `pref_edge_swipes`, JSON `[{key,dir,out}]` | same, plus reserved `emoji`; implicit top/bottom-row swipes are generated at runtime and never stored |
| Letter long-press lists | pref per letter (`Prefs.letterAlternatesKey`), free text, max 14 | glyph (`⎘`) or `:name` |
| Comma / period | `pref_comma_mode` + `_custom`, `pref_*_alternates` | mode enum + literal text |
| Enter alternates | `pref_enter_alternates` (`"? ! ,"`) | literal text |
| Toolbar / `?123` menu | `pref_bar_actions` (string set) + `_order` (CSV) | `EditorAction` enum names, actions only |
| Expansions | Room `Expansion` | text targets; four actions excluded |

So one action has three spellings (`action:paste`, `:paste`, `⎘`), selection and order are stored
separately, and the user can't see the implicit bindings that win over theirs.

**Quick wins (S), independent of the registry:** list every accepted `:name` in the help text
(generated from `EditorAction.entries - NOT_IN_LISTS`); accept every action **by name** in
`glyphForToken` (keep `NOT_IN_LISTS` for glyph matching only); run comma/period `text` mode through
the same parser.

### Unified binding registry (L, phased)

**Model.** Every binding is `Trigger → Output`.

- `Output` is one sealed type: `Text(s)`, `Action(EditorAction)`, `Combo(KeyCombo)`,
  `Special(emoji|…)`. One codec reads all legacy spellings and writes one canonical form.
- `Trigger` names the surface and its key: `chord(mode, l)`, `chord(space, p)`,
  `swipe(backspace, up)`, `hold(a, 3)` (third entry of the long-press list), `hold(comma, 1)`,
  `hold(enter, 1)`, `bar(2)`, `menu(5)`, `expand(brb, 1)`.
- An **availability matrix** says which outputs each surface accepts. A rejection always carries a
  reason the UI shows ("long-press lists can't hold ⇧ combos because …") instead of silently
  dropping it. This is what fixes the user's actual complaint: today the rules differ and are
  invisible.

**One picker everywhere.** Every "choose what this does" field opens the same sheet with four tabs:
*Text*, *Action* (searchable, glyph + plain-language name + one-line description), *Key combo*
(modifier toggles + key), *Special*. Actions the surface can't take are shown disabled with the
reason, not hidden.

**Text format.** Each settings screen gets an "Edit as text" toggle, and settings gets a global
*Export bindings* / *Import bindings*. Same line format in both, one binding per line, versioned
header, comments allowed:

```text
# kinetica-bindings 1
chord   mode+l        action:next_language
chord   space+p       action:toggle_peck_mode
swipe   backspace up  text:!
swipe   enter up      text:?
# implicit: swipe q up -> text:% (number-row symbols)   ← shown, read-only
hold    a             text:á  text:à  action:select_all
hold    comma         text:;  action:paste
bar                   action:settings  action:next_language  action:one_handed
expand  brb           text:be right back
```

- **Import is a dry run first:** parse everything, show a diff (added / changed / removed /
  invalid-with-line-number-and-reason), then *Merge* or *Replace*. Nothing is written until the user
  confirms. Unknown actions from a newer version are kept as invalid lines, not silently dropped.
- Input is lenient (`:paste`, `⎘`, `action:paste` all accepted), output is always canonical.
- Implicit and default bindings appear as comments so the user can see what they'd shadow.
- It slots into the existing backup: `Backup.kt` already writes typed TSV records and skips unknown
  types, so a `binding` record type is additive.

**Phases.**

1. *Façade (M):* the sealed `Output` type, the codec, and the matrix, sitting **over existing
   storage**. Every screen switches to the shared picker. No migration.
2. *Text (M):* per-screen "edit as text", global export/import with the dry-run diff.
3. *Storage (L, optional):* move the pref-backed surfaces into one Room table. Only worth it if
   phases 1–2 show real friction; the façade already gives users the consistency.

wmkeyboard's sealed, serialisable
[`KeyAction`](https://github.com/wasi-master/wmkeyboard/blob/main/core/language/src/main/java/com/wasimaster/wmkeyboard/core/layout/KeyActions.kt#L17-L28)
is the model to copy; LeanType's
[`KeyCode`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/keyboard/internal/keyboard_parser/floris/KeyCode.kt)
ints are unified at dispatch but untyped and have no text format.

**Gotchas.** Glyph collisions with real text (`→`, `»`, `123`) are why `NOT_IN_LISTS` exists, so
the text format uses explicit `text:` / `action:` prefixes and never guesses. Toolbar order and
selection must become one ordered list in the format, even while storage keeps them apart.
Long-press list caps (14) and bar width limits are matrix rules, reported at import.

---

## 3. Autospace timer visualisation

**Today.** Autospace is a real delayed commit. `KineticaIME.scheduleAutospace()` posts
`autospaceRunnable` after `autospaceDelayMs` (swipe), `autospaceTapDelayMs` (tap) or
`autospaceSingleLetterDelayMs`; defaults 300 ms, range 10–800 ms (`Prefs.kt`).
`cancelAutospace()` removes it. When it fires it re-checks the editor before committing `" "`.
`SuggestionBarView.onDraw()` paints one solid `bgPaint` rectangle, then the content.

**Design.** What the user described fits well:

- `SuggestionBarView.showAutospaceCountdown(durationMs)` / `clearAutospaceCountdown()`.
- Call it from `scheduleAutospace()` with the chosen delay; clear it from `cancelAutospace()` and
  when the runnable fires.
- In `onDraw()`, after the background, draw a rectangle in a low-alpha theme accent whose width is
  `1 - elapsed/duration`, and `postInvalidateOnAnimation()` while active. A thin bar along the bottom
  edge is a quieter alternative worth offering.
- Setting: off / bar / full background.

**Complexity.** **M**: the drawing is small; keeping the bar in sync with every cancel/reschedule
path is the work.

**Gotchas.** The runnable can decline to insert when it fires, so the bar reaching zero doesn't
promise a space. At 300 ms the animation is brief; it helps most for users who raise the delay.
Neither reference keyboard shows a pending-autospace timer, so this would be original.

---

## 4. Dedicated language switch key or gesture

**Today.** `EditorAction.NEXT_LANGUAGE` (`action:next_language`) cycles enabled languages via
`KineticaIME.cycleLanguage()`. It is on the toolbar by default and bindable as a chord. There is no
globe key and no swipe for it. Edge swipes accept any output string, so the dispatcher can already
run it from a swipe.

**Design options.**

- **A. Edge swipe (S).** Make sure `NEXT_LANGUAGE` is offered in the edge-swipe picker and suggest a
  default such as *spacebar swipe up*. Horizontal spacebar swipes are cursor movement
  (`EdgeSwipeBindings` flags the collision), but vertical ones are free.
- **B. Spacebar vertical swipe as a built-in (S–M).** What LeanType does:
  [`onLanguageSlide`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/keyboard/KeyboardActionListenerImpl.kt#L628-L653).
- **C. Globe key (M).** New `KeyType`, added to every layout or inserted by a mutation (shrinking the
  spacebar, the reverse of what `withPunctuationKey` does on removal). wmkeyboard has a globe key with
  drag targets
  ([`GlobeKey.kt`](https://github.com/wasi-master/wmkeyboard/blob/main/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/GlobeKey.kt#L38-L57)).
- **D. Keyboard-wide swipe (L).** Needs a gesture route outside the per-key detector and competes with
  glide typing. Not recommended.

**Gotchas.** `ActionRow.available()` hides the action with fewer than two languages; gestures should
no-op the same way.

---

## 5. Insert space after punctuation

**Today.** There is no such option. Kinetica's model goes the other way: punctuation *hugs* the
previous word. `hugsPreviousWord()` and `HUGGING_PUNCTUATION = ".,!?;:)]}»…"` decide this, and
`eatAutospaceBefore()` removes a pending autospace before them. The nearest setting is
`pref_tidy_spaces` (off by default), which avoids double spaces and spaces before closers.

**Design.**

- New preference "Space after punctuation". After a hugging `.,!?;:` is committed, arm the existing
  autospace machinery (`scheduleAutospace()` with the tap delay) instead of a new path. That brings
  retraction and the countdown from item 3 for free.
- Suppress in the known traps: digit before and after (`3.14`, `1,000`), URL/email context, `...`,
  and URL/email/number input fields.

wmkeyboard's
[`HugPunctuation.kt`](https://github.com/wasi-master/wmkeyboard/blob/main/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/HugPunctuation.kt#L29-L60)
handles exactly these exceptions (numeric separators, email dots) and is a good checklist. LeanType
inherits LatinIME's `SpaceState` (phantom/weak/double) machine.

**Complexity.** **M**: little code, many edge cases; needs a unit test per exception.

---

## 6. Resting a finger on a key while typing

**How touch works today.**

- `KeyboardView.handleDown()` routes each new pointer, in order, to a chord, the gesture engine
  (letters), space, backspace or a special key.
- `GestureEngine.onPointerDown()` gives letter pointers one of **two slots**, picked by which half of
  the keyboard they land on, falling back to the other slot if that one is taken. A third letter
  pointer is refused while both are active.
- Each stream is finalised **when its own pointer lifts**, into a `TapToken` or `SwipeToken`
  (`KineticaIME.onTokenFinalized`). `ThumbBeam` decodes the left and right token streams together.
- `onAllPointersUp()` is an empty no-op. A word is committed by space or punctuation, **not** by all
  fingers lifting. (An earlier draft of this review said decoding waits for all pointers; it doesn't.)
- There is one global pending hold (`pendingHoldPid`, `longPressMs` default 500 ms). The hold timer
  never checks whether another pointer is mid-swipe. When it fires it opens the alternates popup and
  calls `engine.cancelPointer()`.

**What happens to a resting finger, by case.**

| Resting finger is on… | Other thumb… | Result today |
|---|---|---|
| A letter with alternates | swipes a word | After 500 ms the rest finger opens a popup and is cancelled. |
| A letter without alternates | swipes a word | When the rest finger lifts, it becomes a `TapToken` and is mixed into the word. |
| Any letter | taps space | Space commits normally. The rest finger does **not** hold the word open. |

**What LeanType actually does** (checked at `f25809d`). It has no explicit resting mode, and neither
Gboard nor SwiftKey document one. Its behaviour emerges from AOSP's `PointerTracker`:

- When a newer pointer **lifts**, older non-modifier pointers get a phantom up and emit their key,
  unless a gesture is in progress.
- `BatchInputArbiter.mayEndBatchInput` ends a gesture only when the lifting pointer is the last one
  (active count 1). So a resting finger keeps the gesture open until it lifts: that's the "word can't
  finish while I'm holding" behaviour the user is used to.
- The long-press timer isn't started while a gesture runs, and starting a gesture dismisses popups.
  Default long-press is 300 ms.

**Design, in two phases.**

1. **Holds don't fire mid-swipe (S).** In `onHoldTimerFired()` and `handleMove()`, if any other
   engine stream is swipe-committed (`engine.isSwipeCommitted(pid)` already exists per stream; add an
   engine-wide query), drop the pending hold **without cancelling** the resting stream. This alone
   matches the user's first ask and is safe for everyone, so it needs no setting.
2. **Explicit resting classification (M), behind a setting.** A letter pointer that stays inside hold
   slop and is older than a threshold (start around 150 ms) when another letter pointer starts a
   swipe becomes *resting*:
   - it emits no token, opens no popup, triggers no key repeat;
   - the swiping thumb's lift ends the swipe as usual;
   - optionally, "hold the word open while a finger rests": defer the autospace/commit until the rest
     finger lifts, which is LeanType's feel.

**Open product question.** Should a resting finger **ever** type its letter? LeanType emits it in some
cases (phantom up when not gesturing). Kinetica could:

- **never** emit it (simplest, most predictable);
- emit it only if it lifts quickly **and** no swipe happened meanwhile (then it was a tap, not a rest);
- emit it LeanType-style.

**Complexity.** Phase 1 **S**; phase 2 **M**. `ThumbBeam` doesn't need changes as long as resting
pointers never become tokens.

**Gotchas.** Two-thumb tapping (left thumb lingers on `t` while the right taps `h`) must not be read as
resting. The age threshold needs measuring against the capture corpus; `DecodeTrace` already logs
pointer timing. With only two slots, a resting finger occupies one, so the user can't swipe with the
same half's thumb: the slot fallback handles it, but a third finger is still refused. Tests extend
`GestureEngineTest`, `DualSimultaneousSwipeTest` and the hold tests in `KeyboardView`.

---

## 7. Reverse the suggestion wheel

**Today.** The vertical wheel is the **recent-words** column. In `SuggestionBarView.onTouchEvent()`
(the `ZONE_RECENT` release branch, ~line 716) finger-down (`dy > 0`) picks `col.above` and finger-up
picks `col.below`, like dragging a physical wheel.

**Design.** An opt-in preference, "Reverse recent-word wheel", **off by default**, so the current
wheel behaviour stays the default. It swaps `above`/`below` at that one site and leaves the upward
flick that picks an ordinary suggestion alone. If the column draws a direction hint, the hint must
follow the setting too.

**Complexity.** **S.** Neither reference keyboard has this.

---

## 8. Spacebar chords don't work

**Resolved: user error. The user confirmed space chords work.** What follows stays as a
discoverability note.

**They are implemented.** `ChordTrigger.SPACE` exists, chords are stored as `space:<key>`, and
`KeyboardView.handleDown()` checks `armedTriggers()` for every key press. The rules:

- **Something must be bound.** There are no default space chords (`ChordDefaults` only creates
  `space:` rows when migrating a `?123` chord that a default shadows). A chord must be added in chord
  settings with the *space* trigger.
- Space must already be held **stationary** for `spaceChordArmMs` (default 50 ms) when the second key
  goes down.
- Space must **not move** past the hold slop first: drift turns it into a cursor slide
  (`SpacebarCursorController`) and disarms it, and resting thumbs drift.
- Space must still be held when the second key **lifts** (`chordAtLift`).

Likely causes, in order: nothing bound to space; thumb drift starting a cursor slide; releasing space
before the letter.

**Design (S).** Ship a couple of default space chords, or surface the trigger choice in chord
settings. Give space a larger slop before a slide while a chord is possible. Tint the spacebar when
armed (overlaps item 10). `chordLift()` already reports the miss reason through
`onChordMissed(trigger, c, reason)`; showing it briefly would make failures self-explaining.

wmkeyboard's chords are Braille-style and hardware Ctrl+Space; LeanType has no configurable chord
system. Kinetica is ahead here.

---

## 9. Gliding near the number row cancels the glide

**What the code shows.**

- Digits are plain `CHAR` keys (`num_*`). `KeyboardView.rebuild()` adds only letters to the swipe
  geometry, so a glide crossing a digit sees no key there and keeps going. **No path cancels a
  stream on entering a digit.**
- On lift, `handleUp()` runs `EdgeSwipeDetector` **before** finalising the stream. The detector fires
  when travel is mostly vertical (≥30 dp, 1.5× dominance, measured at lift **or at the furthest
  point**) and the pointer touched ≤3 keys.
- With the number row on, every top-row letter has an implicit up-swipe binding, and
  `LayoutMutations.withNumberRowSymbols()` puts symbols (`% \ | = [ ] < > { }`) first on those keys.

**Leading cause.** A short top-row glide that overshoots upward into the digits reads as an up-flick
on its start key: travel into the digit row adds no contacts (digits aren't letters), so the ≤3-key
test passes. The detector fires the symbol and the word is dropped, which looks exactly like a
cancelled glide. The detector's KDoc assumes "on the top row there are no letters above it", which
stops being a safety argument once the number row is above.

**Confirming it.** With `DecodeTrace` on, a failed glide logs
`edgeswipe fired key=… dir=UP out=… contacts=…`.

**Fix (S).** When the number row is on, either stop using the peak reading for top-row keys, or count
time spent outside the letter block as a contact. Both keep deliberate flicks working.

### Bigger question: numbers inside words

Today the decoder alphabet is letters plus apostrophe (`KineticaConstants`, `Trie`), and dictionary
words containing digits are dropped at load. A tapped digit goes through `onPunctuation()`, which
calls `finalizePendingWord()` first, so `h2o`, `mp3`, `4k` and `covid19` are split into pieces and
never learned.

| Option | What it means | Complexity | Risk |
|---|---|---|---|
| **A. Fix the detector only** | Glides survive the number row; digits stay separate tokens | **S** | None |
| **B. Tapped digits join the word** | A digit typed straight after letters (no space) extends the composing word as a literal; the whole token is committed and learned as one word | **M** | Must still split `word.` / `3pm` sensibly; learning must keep the digit form |
| **C. Glide through digits** | Digits join the swipe alphabet and trie; `mp3` can be glided | **L** | Every top-row glide that brushes the number row now has digit candidates; needs a strong prior against them |

Recommendation: **A now, B next.** B is what makes `mp3`/`h2o` usable and is how most keyboards
behave. C is the reason gliding keyboards keep digits out of the alphabet; leave it unless B proves
insufficient.

---

## 10. Show chords and gestures while a trigger is held

**Today.** `KeyboardView.drawKey()` draws `key.label` and `key.hintChar` (≈40% alpha). Trigger state
is internal (`modeHoldPointer`, `spacePointer`, `armedTriggers()`); nothing changes on screen while
`?123` or space is held.

**Design.**

- When a trigger becomes armed (after its arm time, not on touch-down, so the overlay never shows a
  chord that wouldn't fire yet), set `activeChordTrigger` and `invalidate()`.
- In `drawKey()`, for keys with a bound chord, draw the chord's short label (`ActionRow.glyph()` for
  actions, the first characters for text) in place of `hintChar`, and dim unbound keys.
- Draw it in the dynamic layer (like trails and pressed keys), not by rebuilding the static key cache.
- Later: the same overlay for edge-swipe bindings while a letter is held.

wmkeyboard's
[`HintBadge.kt`](https://github.com/wasi-master/wmkeyboard/blob/main/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/ui/HintBadge.kt#L16-L78)
draws shortcut badges on keys and is a good visual reference.

**Complexity.** **M.**

---

## 11. Emoji search (and GIFs)

**Today.** `EmojiPickerView` loads `assets/emoji_data.json`, whose entries already carry `name` and
optional `kw` keywords, and filters out emoji the device can't draw (`Paint.hasGlyph`). There is no
search UI. The manifest has **no `INTERNET` permission**.

**The hard part is input, not search.** The picker replaces the keyboard, so the query must be typed
on the keyboard itself, and Kinetica's whole input path writes to the host app.

### How the references solve it

- **wmkeyboard: keyboard-owned query buffer.** A `CaptureField` target (`EMOJI_SEARCH`) receives
  taps, glides, space, Enter and backspace; text goes into `emojiQuery`, **never** to the
  `InputConnection`. Backspace on an empty query is consumed; deleting in the app is a separate,
  labelled control. Search state is cleared on panel change and on every `onStartInput`.
  ([`CaptureField.kt`](https://github.com/wasi-master/wmkeyboard/blob/9e700bd2e1b5fc34c851d250f1c378eb7e53f73b/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/CaptureField.kt#L4-L22),
  [routing](https://github.com/wasi-master/wmkeyboard/blob/9e700bd2e1b5fc34c851d250f1c378eb7e53f73b/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/WMKeyboardService.kt#L8124-L8173))
- **LeanType: non-focusable `EditText`.** It swaps the key listener while searching. It had to fix
  focus hijacking explicitly
  ([`f3500a3`](https://github.com/LeanBitLab/LeanType/commit/f3500a3978bba251014d7990807f283c95692bfb)):
  an IME that focuses its own field can lose the host's `InputConnection`.
- **HeliBoard: separate activity.** Avoids focus inside the IME but needed fixes for lifecycle races
  ([HeliBorg/HeliBoard#2326](https://github.com/HeliBorg/HeliBoard/pull/2326)).

**Recommendation:** wmkeyboard's pattern. Kinetica already draws its own keyboard and centralises host
writes in `InputConnectionHelper`, so a `SearchCapture(query, cursor)` state in `KineticaIME` plus a
drawn query row is the natural fit. No focusable `EditText`.

### Pitfalls checklist

| Pitfall | Rule |
|---|---|
| Query leaks into the app | While capture is active, **every** input (tap, glide result, space, Enter, backspace, hardware key) goes to `SearchCapture`. Only choosing an emoji writes to the host. |
| Glide commits the query word to the app | The decoder result is handed to the query buffer, not the normal commit path. |
| Learning and autocorrect | Query words must not be learned, autocorrected, autospaced, or counted for undo. Skip `commitTracked` and all `teachesNothing`-gated paths. |
| Empty-query backspace deletes app text | Consume it. Offer an explicit "delete in app" if wanted. |
| `onUpdateSelection` | Must not react to query edits (it abandons pending words on cursor moves). |
| Shift / symbols / language switch | Affect the search keyboard only; restore prior state on exit. |
| Restart and rotation | One rule: clear search on `onStartInput` and on view rebuild. |
| Emoji identity | Emoji are multi-code-point (ZWJ, skin tones, flags). Never backspace or compare by `Char`. Keep the selected variant. |
| Localised search | English `name`/`kw` first; CLDR annotations for other languages (check the licence notice). |
| Accessibility | The drawn query row needs content descriptions and result-count announcements. |

**Complexity.** **M** for offline search with the capture buffer and English keywords; **L** with
localised CLDR keywords. Cheaper interim (**S–M**): when the emoji key is tapped right after a word,
show matching emoji in the suggestion bar.

### GIF search

**How wmkeyboard does it.** Providers KLIPY and GIPHY (keys: user-supplied first, then build-time),
Wikimedia Commons in the F-Droid build, and offline packs. An empty query shows trending; typing is
debounced 450 ms; enabled providers are queried concurrently. Previews use a 64 MiB disk cache, and
picked files are cached (~30 newest). Insertion reads the field's `contentMimeTypes`, picks a
compatible type, exposes the file through `FileProvider`, and calls
`InputConnectionCompat.commitContent`; on failure it puts the URI on the clipboard. A `nointernet`
build flavour removes the permission
([insertion](https://github.com/wasi-master/wmkeyboard/blob/9e700bd2e1b5fc34c851d250f1c378eb7e53f73b/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/WMKeyboardService.kt#L33286-L33361)).

**What it would cost Kinetica.** GIF search is a different kind of feature, not a bigger emoji search:

- **It adds `INTERNET`.** Kinetica is currently network-free, which is a real selling point for a
  keyboard. Queries go to a third party. Play Data Safety needs updating; on F-Droid it is likely
  flagged *NonFreeNet*.
- **Provider terms.** GIPHY requires "Powered by GIPHY" attribution and forbids mixing its results
  with other providers or reordering them. Tenor requires v2 keys and attribution.
- **Insertion is not universal.** Each field advertises its own MIME types; GIF, animated WebP and
  MP4 differ. Some apps accept but show one frame. Never promise it works everywhere.

**Proposal.** Ship emoji search first. If GIFs are wanted, do them as a **separate, opt-in build
flavour or module** (`INTERNET` only there), with one provider at a time, visible attribution, never
in password or private fields, `commitContent` with clipboard fallback, and a bounded cache.
Complexity **L**, mostly compliance and testing rather than code. A no-network middle ground: let
users import a local GIF/sticker pack and reuse the same insertion path (**M**).

---

## 12. Downrank or banish suggestions

**Today, all hidden behind one gesture.** In `SuggestionBarView`, holding a suggestion for
`REINFORCE_HOLD_MS` (450 ms) arms **weight adjustment**: every 24 dp up or down changes the word's
personal count, and going far enough down blocks it (`BarAdjust`). Blocking persists a
`BlockedWord(word, lang)` (`KineticaIME.onSuggestionBlocked()`) and `DictionaryLoader` strips blocked
words from the trie. Blocked words are managed in dictionary settings. The user didn't find any of
this, which is the real problem.

Sliding down lowers only the *personal* count; corpus frequency remains, so a common word can't be
pushed below its corpus rank. That's why "downrank" feels missing.

### Hold-for-menu: feasible (M)

A context menu in plain language on a held suggestion:

- **Never suggest "teh"** (block, with the language named)
- **Suggest "teh" less**
- **Suggest "teh" more**
- **Forget what I've taught about "teh"** (reset personal count and penalty)
- **Why am I seeing this?** (source: dictionary / learned / next-word / correction; personal count)
- **Add to dictionary** (for learned-only words)

**Resolving the gesture conflict.** Hold already means "adjust". Options:

1. **Hold still → menu; hold and slide → adjust (recommended).** Arm at 450 ms as today; if the
   finger leaves the slop, it's the existing slide. If it's still stationary after a longer delay
   (~650 ms), open the menu. Power users keep the slide; everyone else gets
   discoverable text.
2. Replace the slide with the menu. Simpler, but removes a fast gesture people may rely on.
3. Menu on a dedicated gesture (double-tap). Doesn't help discoverability.

**Implementation.**

- Reuse `KeyboardView`'s `PopupState` / `PopupStripView` popup, or a vertical list anchored above
  the cell.
- **Provenance plumbing is the main work.** `KineticaIME.barSuggestion()` reduces a `WordCandidate`
  to `Suggestion(word, tier, count)`, dropping its source and frequency weight. "Why am I seeing
  this?" and a correct "less" need those carried through to the bar.
- **Real downrank:** store a per-word penalty next to the personal count, load it with
  `personalCounts`, and apply it in `WordPredictor.scoreReading()` **and** the next-word and
  correction paths, or the word comes back through bigrams.
- Gate all writes on `EditorState.teachesNothing`, like existing learning.

**Complexity.** Menu + block + reset: **M**. With the penalty and provenance: **M**, upper end.

---

## 13. Shortcuts when holding number keys

**Today.** `LayoutMutations.withNumberRow()` creates ten plain `CHAR` keys (`num_1`…`num_0`) with no
alternates; its KDoc says digits are tap-only. With the row on, the top letter row gets the symbol
alternates instead.

**Design.** Give `num_*` keys an alternates list editable like the letter lists (parsed by the same
`:name` path, so actions work), or one `ShortcutKinds` binding per digit. `onHoldTimerFired()`
already opens a popup for any `ROUTE_SPECIAL` key with alternates, so the touch side needs little
work; the cost is settings UI and storage. A natural default is each digit's shifted symbol
(`1`→`!`, `2`→`@`, …).

**Complexity.** **M.** Neither reference keyboard has bindable number-row shortcuts.

---

## 14. Long-press toolbar keys for alternate actions

**Today.** Toolbar cells live inside `SuggestionBarView` and are **tap only**: no long-press, drag,
pin or edit mode on the keyboard. `PASTE` maps to the platform `android.R.id.paste`. The toolbar is
arranged only in settings (`ShortcutSetActivity`: checkboxes plus up/down buttons).

**About the rearrange/pin conflict.** I couldn't find in-keyboard rearranging or pinning in the code
(`main` or any branch), so today there's nothing for a long-press to collide with. The conflict is
real for the future, though: LeanType uses long-press on toolbar keys for alternate actions *and*
lets you pin keys, and if Kinetica later adds drag-to-rearrange, the two will fight over the same
gesture. A policy now avoids that:

| Option | Long-press does | Rearrange / pin via | Trade-off |
|---|---|---|---|
| **A. Edit mode (recommended)** | Alternate action, always | An explicit *Edit toolbar* mode (from the `?123` menu or settings); in it, drag reorders and tap pins/unpins, and nothing executes | One gesture, one meaning; editing is rare so an extra step is fine |
| B. Duration split | Alternate action at 400 ms | Drag after a longer hold | Hard to discover, easy to trigger the wrong one |
| C. Settings only | Alternate action | Settings screen (today's convention) | Simplest; no in-keyboard arranging |

**Design (M).**

- A `longPressAction(action)` table next to `ActionRow.glyph()`, a hold timer limited to action
  cells (excluded from the page-swipe candidate in `SuggestionBarView`), and a small corner dot on
  cells that have one.
- Starting mapping, after LeanType's
  [`getCodeForToolbarKeyLongClick`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/latin/utils/ToolbarUtils.kt#L298-L316):
  select all → select word, paste → clipboard history (item 15), copy → copy line, arrows →
  word/page jumps, next language → a language picker (new).
- *Select word* is a new local action (read around the cursor, `setSelection`).
- Once item 2's registry exists, long-press actions are just another binding surface
  (`barhold(2)`), so users can rebind them.

---

## 15. Clipboard history

Split out of item 14: it's an independent feature and the biggest one on the list.

### What others do

| Keyboard | Retention and pins | Content | Sensitive / OTP |
|---|---|---|---|
| SwiftKey | Unpinned clips expire after 1 h; pin to keep; drag to reorder | Text and images | Cloud sync of last clip (account) |
| Gboard | Temporary + pinned; exact limits undocumented | Text and images | No public OTP mechanism |
| FlorisBoard | Room; dedupe keeps pin; count and age eviction | Text, images, video copied to `noBackupFilesDir` | Honours and masks the sensitive flag |
| LeanType | SQLite; age cleanup spares pins | Text, images | Best UI: search, edit, multi-select, batch pin/delete, undo |
| wmkeyboard | 100 entries, 1 day; sensitive clips 5 min | Text, HTML, links, images, files | Secret heuristics; OTP/phone/URL extraction |
| FUTO | **Off by default**; 25 entries, 3 days, 40 MiB media | Text, PNG/JPEG/WebP (10 MiB cap) | Rejects sensitive clips unless allowed; nothing saved before unlock |

FlorisBoard's storage model, LeanType's UI and FUTO's privacy defaults are the three to borrow from.

### Design

**Phase 1: text history (M).**

- `ClipboardManager.OnPrimaryClipChangedListener` while the feature is on. Since Android 10 only the
  focused app or the **default IME** can read the clipboard, so clips copied while another keyboard
  was active are missed.
- A **separate** `clipboard.db` (not `user_dict.db`), excluded from backup. Kinetica sets
  `android:allowBackup="true"`, so this needs explicit backup and data-extraction rules.
- Entry: text, preview, MIME, content hash, copied-at, expires-at, pinned, sensitive.
- Dedupe by hash (refresh timestamp, keep pin). Evict unpinned by age **and** count.
- Panel in the emoji picker's slot: tap pastes, pin/delete per item, *Clear unpinned*, *Clear all*
  (confirm if pins exist). *Paste as plain text* via `commitText`.
- **Never store** when `EditorState.teachesNothing` (private mode or no-learning field), when the clip
  has `EXTRA_IS_SENSITIVE`, or in password fields. This gate already exists for every learning path.

**Phase 2: search, edit, multi-select (M).** Searching reuses the item 11 capture buffer, so do item
11 first or together. Undo for deletes.

**Phase 3: images (M–L).**

- Clipboard images are `content:` URIs whose read grant is temporary. Copy them into
  `noBackupFilesDir` immediately, or they're gone.
- Allow-list PNG/JPEG/WebP, cap input bytes **and** decoded pixels, build thumbnails off the main
  thread, and delete files with their entries.
- Paste with `FileProvider` + `commitContent`, only when the field's `contentMimeTypes` match;
  otherwise say the app doesn't accept images (no fake success). Shares the insertion path with GIFs
  in item 11.

**Phase 4: one-time codes (S–M).**

- **No SMS or notification access.** SMS Retriever and User Consent are for an app verifying its own
  number, not a keyboard feed, and Play restricts `READ_SMS`. A notification listener would be a
  privacy overreach.
- Instead: when a new clip is a standalone 4–8 digit code, keep it **in memory only** for a few
  minutes and show a *Paste code* chip in the suggestion bar while a numeric or OTP-like field is
  focused. Never persist, search, preview or log it; clear it after use.
- Separately, Android 11+ **inline autofill** (`supportsInlineSuggestions`) lets password managers
  show their own codes and passwords in Kinetica's bar without Kinetica seeing them. That's the
  proper route to "codes appear automatically".

**Defaults.** Off until the user opts in, unpinned text kept ≤24 h, about 25 entries, small media
cap, no sync. Android 13 shows its own "copied" preview and auto-clears the system clipboard after
about an hour; neither cleans Kinetica's copy, so Kinetica's own expiry must.

**Complexity.** Phase 1 **M**; all four phases together **L**.
