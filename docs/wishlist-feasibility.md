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
| 1 | Hebrew layout with English backspace position | Hebrew backspace is top-right by design | **S** (just change it) / **M** (selectable variant) | Feasible |
| 2 | Consistent key bindings everywhere | Four binding surfaces with different action sets | **S** quick fixes / **L** unified registry | Feasible; quick wins first |
| 3 | Autospace timer visualisation | Timer exists; nothing shows it | **M** | Feasible |
| 4 | Language switch key or gesture | Action exists (toolbar, chords); no key, no swipe | **S** gesture / **M** key | Feasible |
| 5 | Space after punctuation | Not an option; punctuation hugs the previous word | **M** | Feasible with care |
| 6 | Rest a finger on a key while typing | Long-press timer hijacks the resting finger; its tap joins the word | **M–L** | Feasible; policy decision needed |
| 7 | Reverse suggestion wheel direction | Hard-coded mapping | **S** | Trivial |
| 8 | Spacebar chords don't work | Supported, but nothing bound by default and timing is strict | **S** | Mostly discoverability |
| 9 | Number row cancels glides | Cause not proven; leading suspect is the up-flick shortcut | **S** after reproduction | Needs a trace first |
| 10 | Show chords when holding `?123`/space | Not shown | **M** | Feasible |
| 11 | Emoji search | Data already shipped; no search UI | **M–L** | Feasible |
| 12 | Downrank or banish suggestions | Ban **already exists**; persistent downrank doesn't | **S** (discoverability) / **M** (downrank) | Mostly done |
| 13 | Long-press shortcuts on number keys | Number keys are tap-only | **M** | Feasible |
| 14 | Long-press toolbar keys (select word, clipboard) | No toolbar long-press, no clipboard history | **M** (long-press) / **L** (clipboard history) | Feasible; clipboard is the big part |

Suggested order by value per effort: 7, 8, 2 (quick fixes), 1, 4, 12, 3, 6, 9, 13, 10, 5, 14, 11.

---

## 1. Hebrew layout with backspace where English has it

**Today.** Hebrew uses its own asset, `app/src/main/assets/layouts/native_he.json`, chosen ahead of
any arrangement by `AlphaLayouts.name()` (`native_<lang>` wins). Its backspace is the last key of the
**top** row (`x=0.80, y=0.0, w=0.20`, line 17). English `qwerty.json` puts it at the right of the
**bottom letter row** (`x=0.85, y=0.50, w=0.15`, line 35). The board is `"fixedArrangement": true`,
and no mutation in `LayoutMutations` relocates special keys.

**Design options.**

- **A. Edit `native_he.json`** to the SI-1452 shape, with backspace at the right of the bottom
  letter row. This matches wmkeyboard's
  [`he_si1452.wmlayout.json`](https://github.com/wasi-master/wmkeyboard/blob/main/app/src/main/assets/layouts/he_si1452.wmlayout.json#L116-L164),
  which puts shift bottom-left and delete bottom-right as on QWERTY. Changes everyone's muscle memory.
- **B. Ship a variant asset** (e.g. `native_he_1452.json`) plus a per-language "layout variant"
  preference read by `AlphaLayouts.name()`. LeanType does the same (`hebrew.json` and
  `hebrew_1452_2.json`).
- **C. A generic `withBackspacePlacement()` mutation.** Not recommended: moving one key means
  reflowing the row it leaves and the row it joins, which is layout-specific.

**Complexity.** A: **S**. B: **M** (preference, settings row, backup, the bundled-layout test
listed in `ADDING_A_LANGUAGE.md`).

**Gotchas.** Keep the key id `backspace`: edge-swipe bindings are keyed by id
(`EdgeSwipeBindings`). Coordinates are not mirrored for RTL, so the asset says exactly where the key
goes. Decoder geometry is derived from the layout, so the decoder needs no change, but letter
positions move and that should be sanity-checked on real swipes.

---

## 2. Key bindings are inconsistent

**Today there are four binding surfaces, each with its own action set:**

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

**Design.**

1. *Quick wins (S):* list every accepted `:name` in the help text (generate it from
   `EditorAction.entries - NOT_IN_LISTS`); let `glyphForToken` accept every action **by name**
   (keep `NOT_IN_LISTS` for glyph matching only); run comma/period `text` mode through
   `glyphForToken` so `:name` works there too.
2. *Unified registry (L):* make `ShortcutKinds` the one model (text / action / Ctrl combo /
   reserved) with a per-surface availability flag, and give every surface the same picker instead of
   free text. wmkeyboard's sealed, serialisable
   [`KeyAction`](https://github.com/wasi-master/wmkeyboard/blob/main/core/language/src/main/java/com/wasimaster/wmkeyboard/core/layout/KeyActions.kt#L17-L28)
   is the cleaner model; LeanType's
   [`KeyCode`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/keyboard/internal/keyboard_parser/floris/KeyCode.kt)
   ints are unified at dispatch but untyped.

**Gotchas.** Stored preferences and backups hold heterogeneous strings (`:paste`, `⎘`,
`action:paste`), so the registry needs a decoder for all three forms. Glyph collisions with real
text (`→`, `»`, `123`) are why `NOT_IN_LISTS` exists; keep that guard for glyphs.

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

**Today.** Touch is genuinely multi-pointer: `KeyboardView.onTouchEvent()` handles
`ACTION_POINTER_DOWN`, and `GestureEngine` keeps **two streams, one per half of the keyboard**
(`maxPointers = 2`). A word is decoded only in `onAllPointersUp()`. That gives three problems for a
resting finger:

1. **The long-press timer hijacks it.** A stationary letter finger schedules a hold
   (`scheduleHold()`); after `longPressMs` (default 500) `onHoldTimerFired()` opens the alternates
   popup and calls `engine?.cancelPointer(pid)`. The resting finger becomes a popup.
2. **Its tap joins the word.** If no hold fires, the resting finger's stream finishes as a
   `TapToken` when it lifts and is decoded together with what the other thumb typed.
3. **The word never ends while it rests.** Decoding waits for *all* pointers up, so the other thumb's
   consecutive words merge into one decode.

Point 3 is what the user asked for ("the word can't finish while you're holding"), but combined with 1
and 2 it produces the disruption they describe. Also, only one pending hold exists at a time
(`pendingHoldPid`), and a third finger is refused while both slots are taken.

**Design: a "resting finger" mode.**

- When a second pointer goes down while a letter pointer is **stationary** (inside hold slop and older
  than, say, 150 ms), reclassify the stationary one as *resting*: cancel its hold timer and its engine
  stream so it neither contributes a letter nor keeps the word open.
- A resting finger that lifts without having moved emits nothing.
- While any engine stream is a committed swipe, don't schedule holds or open popups for other pointers
  (the user's "hold shortcuts shouldn't turn on mid-swipe").
- A setting, off by default, because it changes semantics for genuine two-thumb tapping.

LeanType gets this behaviour from LatinIME's `PointerTracker` +
[`BatchInputArbiter`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/keyboard/internal/BatchInputArbiter.kt#L160-L171),
where batch input isn't finished until the pointer count drops to one. Kinetica's two-slot,
decode-on-all-up design is different enough that this must be a deliberate policy, not a port.

**Complexity.** **M** for the mode above; **L** if the decoder must reason about resting pointers.
`ThumbBeam` is a decoder over tokens with one cursor per thumb, not a touch stitcher, so it shouldn't
need changes if resting pointers never become tokens.

**Gotchas.** Two-thumb tapping (left thumb on `t` slightly long while the right taps `h`) must not be
misread as resting; the stationary-and-old threshold needs measuring on the existing capture corpus
(`DecodeTrace` already logs inter-pointer distances). Needs multi-touch tests covering pointer-up
ordering.

---

## 7. Reverse the suggestion wheel

**Today.** The vertical wheel is the **recent-words** column. In `SuggestionBarView.onTouchEvent()`
(the `ZONE_RECENT` release branch, ~line 716) finger-down (`dy > 0`) picks `col.above` and finger-up
picks `col.below`, like dragging a physical wheel.

**Design.** Preference "Reverse recent-word wheel" that swaps `above`/`below` at that one site.
Leave the upward flick that picks an ordinary suggestion alone.

**Complexity.** **S.** Neither reference keyboard has this.

---

## 8. Spacebar chords don't work

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

**The engine shouldn't cancel on digits.** `KeyboardGeometry.keyAt()` knows letters only, so a glide
passing over a digit sees no new key; `handleMove()` keeps feeding the stream, and no path cancels a
stream on entering a digit.

**Leading hypothesis (unverified).** The top-row up-flick shortcut. On lift, `EdgeSwipeDetector`
fires an edge swipe instead of decoding when travel is mostly vertical (≥30 dp, 1.5× dominance, at
lift or at the furthest point) and the pointer touched ≤3 keys. Its KDoc reasons that "on the top row
it cannot steal a typed word: there are no letters above it". With the number row on there is room
above, and time over digits adds no contacts because they aren't letters. A short top-row glide that
overshoots upward into the digits can therefore read as an up-flick on its start key and fire that
key's corner shortcut, which looks like a cancelled glide.

**Next step.** Reproduce with `DecodeTrace` enabled; the detector logs `edgeswipe fired key=…
contacts=…`. If confirmed (**S**): with the number row on, stop consulting the peak reading for
top-row keys, or count leaving the letter block as a contact.

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

## 11. Emoji search

**Today.** `EmojiPickerView` loads `assets/emoji_data.json`, whose entries already carry `name` and
optional `kw` keywords; the code says they're kept for a future search panel. There is no search UI.

**The hard part is input, not search.** The picker replaces the keyboard, so there are no keys to type
the query with, and anything typed through the normal path goes to the host app.

**Design.**

- Search: an in-memory prefix index over `name` + `kw` (a few thousand entries; a sorted term list is
  plenty). wmkeyboard's
  [`EmojiSearch.kt`](https://github.com/wasi-master/wmkeyboard/blob/main/core/emoji/src/main/java/com/wasimaster/wmkeyboard/core/emoji/EmojiSearch.kt#L21-L129)
  ranks exact, prefix, keyword, synonym, then fuzzy matches.
- Input: show a compact letter board under a search pill and route its taps into a **local query
  buffer** instead of the `InputConnection`. That is wmkeyboard's approach
  ([`EmojiFields.kt`](https://github.com/wasi-master/wmkeyboard/blob/main/feature/ime/src/main/java/com/wasimaster/wmkeyboard/ime/ui/EmojiFields.kt#L579-L658)).
  LeanType instead embeds an `EditText` with a `TextWatcher`
  ([`EmojiPalettesView.kt`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/keyboard/emoji/EmojiPalettesView.kt#L274-L300)).
  An IME focusing its own `EditText` is fragile; the query buffer fits Kinetica better, since
  `KineticaIME` already owns all key dispatch.
- Cheaper interim win (**S–M**): when the emoji key is tapped right after a word, show emoji matching
  that word in the suggestion bar.

**Complexity.** **M** with the query buffer and English keywords; **L** with localized CLDR
annotations (LeanType downloads per-language `emoji_<lang>.dict`).

---

## 12. Downrank or banish suggestions

**Banishing already exists.** Long-press/slide a suggestion in the bar to arm a block →
`KineticaIME.onSuggestionBlocked()` persists a `BlockedWord(word, lang)` and removes it from the live
counts, and `DictionaryLoader` strips blocked words before building the trie. Blocked words are
managed in dictionary settings. That the user didn't find it suggests a hint (e.g. a `LearnTips`
entry) more than code.

**Downranking partly exists.** Sliding a suggestion down lowers its *personal* count, but corpus
frequency remains, so a common word can't be pushed below its corpus rank.

**Design for a real downrank (M).** Store a per-word penalty (a column next to the personal count, or
a negative count), load it with `personalCounts`, and multiply it into `WordPredictor.scoreReading()`
and the next-word path. Apply it to every suggestion source, or the word reappears through bigrams or
corrections.

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

**Today.** Toolbar cells are `ActionRow` actions dispatched through `performIfAction()`; they have no
long-press. `PASTE` maps to the platform `android.R.id.paste`. There is no clipboard history.

**Design, in two independent parts.**

1. **Toolbar long-press (M).** A `longPressAction(action)` table beside `ActionRow.glyph()`, a hold
   timer in the toolbar touch handler, and a small dot on cells that have one. LeanType's mapping is a
   ready list
   ([`ToolbarUtils.getCodeForToolbarKeyLongClick`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/latin/utils/ToolbarUtils.kt#L298-L316)):
   select all → select word, paste → clipboard panel, arrows → word/page jumps. Select word needs a
   new local action (read around the cursor, then `setSelection`). Could become user-bindable once
   item 2's registry exists.
2. **Clipboard history (L).** A `ClipboardManager.OnPrimaryClipChangedListener` while the IME runs, a
   Room table (`KineticaDb` exists), pin/delete/clear, expiry, and a panel in the emoji picker's place.
   Both references have one: LeanType's
   [`ClipboardHistoryManager`](https://github.com/LeanBitLab/LeanType/blob/main/app/src/main/java/helium314/keyboard/latin/ClipboardHistoryManager.kt#L327-L421)
   and wmkeyboard's
   [`ClipboardStore`](https://github.com/wasi-master/wmkeyboard/blob/main/core/content/src/main/java/com/wasimaster/wmkeyboard/core/clipboard/ClipboardStore.kt#L500-L626)
   (pins, cap, sensitive-content expiry).

**Gotchas.** Since Android 10, only the focused app or the active IME can read the clipboard, so clips
copied while another keyboard was active are missed. Clipboard content is sensitive: respect
`ClipDescription.EXTRA_IS_SENSITIVE` and private mode, and exclude history from backups by default.
