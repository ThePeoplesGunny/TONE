# TONE Beta 2.5 — Deficiency Report
*Full source audit. All findings traced to line numbers and function names.*
*Severity assigned relative to impact on correctness, Beta 2.6 readiness, and data integrity.*

---

## Severity Classification

| Level    | Meaning                                                              |
|----------|----------------------------------------------------------------------|
| CRITICAL | Produces wrong output, corrupts data, or will break Layer 4 directly |
| HIGH     | Functional gap with misleading or incorrect behavior                 |
| MEDIUM   | Incomplete feature or confusing edge case                            |
| LOW      | Dead code, cosmetic inconsistency, or deferred concern               |

---

## CRITICAL

---

### DEF-01 · `baseFret` SVG logic always collapses to 1
**File location:** `chordDiagramSVG()` — lines 1570–1572
**Code:**
```js
let baseFret = usedFrets.length ? Math.min(...usedFrets) : 1;
if (baseFret > 1 && Math.max(...usedFrets) - baseFret < 4) baseFret = baseFret;
else baseFret = 1;
```
**Problem:** The `else` branch unconditionally sets `baseFret = 1` for any voicing where `Math.max - Math.min >= 4`. This fires for virtually all non-open chords. High-position chords (frets 5, 7, 9) always render starting from fret 1, placing dots far outside the diagram's visible fret window. The diagram shows an impossible voicing.
**Impact:** Every non-open chord voicing is rendered incorrectly.
**Fix direction:** `baseFret` should equal `Math.min(...usedFrets)` when `min > 0`, with a nut indicator suppressed and a position label added.

---

### DEF-02 · `Cmaj7` and similar tokens fail silently
**File location:** `parseChordToken()` — lines 2064–2070 (`QUALITY_SUFFIXES`)
**Code:**
```js
const QUALITY_SUFFIXES = [
  { suffix:'maj7',   key:'maj7'   },  // ← present
  { suffix:'aug',    key:'aug'    },
  { suffix:'maj',    key:'major'  },  // ← strips 'maj' before 'maj7' can match if order wrong
  ...
  { suffix:'',       key:'major'  },
];
```
**Verification:** For token `Cmaj7`: root = `C`, rest = `maj7`. Matching iterates `QUALITY_SUFFIXES` longest-first. `maj7` IS listed — but only if the array order places it before `maj`. Current order puts `maj7` at index 0, which is correct. **However:** `min7b5` is not in QUALITY_SUFFIXES at all. Token `Cm7b5` → rest = `m7b5` → no match → `null`. **`Cm7b5` and `Cmin7b5` fail silently.** Additionally `Cdim7` → rest = `dim7` → present. Confirmed: `m7b5` suffix is missing.
**Impact:** Chord tokens using `m7b5` suffix are silently dropped as parse errors. No indication to user.
**Fix direction:** Add `{ suffix:'m7b5', key:'min7b5' }` to `QUALITY_SUFFIXES` before the `m` entry.

---

### DEF-03 · `analyzeChord` overloads `func` field — breaks Layer 4
**File location:** `analyzeChord()` — lines 2257–2264
**Code:**
```js
// Secondary dominant branch:
return {
  numeral: 'V/' + dChord.numeral,
  func: 'dominant',         // ← same as plain dominant
  status: 'secondary',
  source: 'sec. dominant → ...',
  color: 'var(--c-VII)',
};
```
**Problem:** `func: 'dominant'` is assigned to both plain dominant chords and secondary dominants. The `prog-function` div renders `analysis.func` directly. Beta 2.6 will add emotional tendency labels by reading `analysis.func`. A secondary dominant (`V/IV` = temporary tension toward IV) has a meaningfully different emotional character than a plain dominant (`V` = tension seeking tonic). Both will receive the identical label.
**Impact:** Layer 4 cannot differentiate these two without a code change. The bug must be resolved before 2.6 labels are written, or the labels will be wrong.
**Fix direction:** Either add `func: 'secondary'` to the secondary dominant return, or rename the func values so the Layer 4 lookup can distinguish them cleanly.

---

### DEF-04 · Save regex vulnerable to data corruption
**File location:** `saveAndDownload()` — line 3117
**Code:**
```js
const replaced = src.replace(/const PRESETS = \[[\s\S]*?\];/, presetsBlock);
```
**Problem:** The non-greedy `[\s\S]*?` terminates at the *first* occurrence of `];` in the document after `const PRESETS = [`. If any preset's `notes`, `tags`, or any string field contains the substring `];`, the regex will prematurely close the match at that point, replacing only a partial PRESETS block. The resulting HTML will have a malformed `const PRESETS` followed by the remainder of the original array — silent data loss on save.
**Current exposure:** The 5 existing presets have empty `notes` fields. Safe today, not safe with user data.
**Impact:** Save system will corrupt the file when notes contain `];`. No error is thrown.
**Fix direction:** Use a more specific regex anchor (e.g., match `const PRESETS = ` followed by the JSON array structure), or serialize by replacing between unique comment markers rather than pattern-matching the JS literal.

---

## HIGH

---

### DEF-05 · Inner strings cannot be muted — physically invalid voicings
**File location:** `computeVoicings()` DFS — lines 1521–1524
**Code:**
```js
// Option A: mute this string (only outer strings)
if (strIdx === 0 || strIdx === tuningNotes.length - 1) {
  dfs(strIdx + 1, [...current, null], minFret, maxFret);
}
```
**Problem:** Only the low E (index 0) and high e (index 5) can be muted. Strings 2–5 are always forced to play. Voicings like `x02210` (Am open) require muting string 6 (low E), which is supported, but many common shapes mute inner strings. Example: `x3231x` is a valid C voicing shape. This filter was intended to prevent floating mutes but overcorrects.
**Impact:** A large range of valid voicings is never generated. Some generated voicings require playing strings that a real chord would leave unplayed.

---

### DEF-06 · `detectKey` ties always break in favor of Major — harmonically wrong
**File location:** `detectKey()` — lines 2166–2169
**Code:**
```js
if (score > bestScore || (score === bestScore && mode === 'major')) {
  bestScore = score;
  bestKey = { root, mode, score, maxScore: parsedChords.length * 2 };
}
```
**Problem:** When C Major and A Minor score equally (common for vi-IV-I-V progressions), the algorithm always returns Major. The relative minor relationship means these two keys will frequently tie. No musical information (first chord, last chord, chord emphasis) is used to break the tie — only a hardcoded mode preference.
**Impact:** Progressions with strong minor tonality are frequently misidentified as their relative major. The decoder then labels chords incorrectly (the `i` tonic becomes `vi`).

---

### DEF-07 · `V/X` secondary dominant chips colored wrong in Library
**File location:** `fpChipsHtml()` — lines 2996–3010
**Code:**
```js
const base = numeral.replace(/[^A-Za-z]/g,'').toLowerCase();
const majorMap = {i:0, ii:2, iii:4, iv:5, v:7, vi:9, vii:11};
...
let semi = map[base] ?? 0;
```
**Problem:** For a numeral like `V/IV`, `replace(/[^A-Za-z]/g,'')` produces `'viv'`. This is not a key in either map, so `semi = 0` (root color) is applied. The chip for a secondary dominant is always colored as the root interval (red), regardless of the actual harmonic target.
**Impact:** Secondary dominant chips in the Library fingerprint display are visually incorrect. A `V/IV` should carry the dominant color (cyan), not the root color (red).

---

### DEF-08 · Slash chord parsing silently drops bass note
**File location:** `parseProgression()` — line 2120
**Code:**
```js
const tokens = input.trim().split(/[\s,/|]+/).filter(t => t.length > 0);
```
**Problem:** The forward slash `/` is in the split pattern. Input `Am/C G F` tokenizes to `['Am', 'C', 'G', 'F']`. The `C` is treated as a standalone C major chord, not as a bass note modifier of Am. This changes both the harmonic analysis (C becomes a tonic function chord instead of a bass note) and the fingerprint. No warning is shown.
**Impact:** Slash chords entered by users produce wrong analysis and wrong fingerprints silently.

---

### DEF-09 · `checkModalBorrowing` sparse for minor keys
**File location:** `checkModalBorrowing()` — lines 2228–2233
**Code:**
```js
} else {  // minor key
  if (semi===2  && quality==='major') return 'Dorian (II)';
  if (semi===5  && quality==='major') return 'Melodic Minor / Lydian (IV) ';
  if (semi===0  && quality==='major') return 'Parallel major (I)';
}
```
**Problem:** Only 3 borrowing cases handled for minor keys. Missing at minimum:
- `semi===3, major` → Parallel major (♭III)
- `semi===8, major` → Parallel major (♭VI)
- `semi===10, major` → Parallel major (♭VII)  — this is diatonic natural minor, should not appear here but currently unhandled
- `semi===2, dom7` → Secondary applied function
**Impact:** Most borrowed chords in minor-key progressions fall through to the 'chromatic' bucket, which is factually wrong. The label `chromatic` will appear where `borrowed` should appear in the decoder cards. This also affects Layer 4 label assignment.

---

## MEDIUM

---

### DEF-10 · Span check is position-naive
**File location:** `computeVoicings()` DFS — line 1518
**Code:**
```js
const span = usedFrets.length ? Math.max(...usedFrets) - Math.min(...usedFrets) : 0;
if (span <= 4) results.push([...current]);
```
**Problem:** `span <= 4` is applied uniformly regardless of fret position. Physically, a 4-fret span at fret 1-4 is near-impossible for most players (frets are wider, fingers are spread). A 4-fret span at fret 9-12 is comfortable (frets narrow). The filter produces false positives at low positions and may exclude valid high-position voicings.
**Note:** This is the Pillar 3 gap documented in TONE_CONTEXT.md. Flagged here because it also affects 2.6 (the chord cards shown to users are not physically validated).

---

### DEF-11 · Heat map mode is a `const` with no UI control
**File location:** Line 2390
**Code:**
```js
const currentHeatMode = 'both';
```
**Problem:** `heatMapSVG` supports three modes: `'interval'`, `'frequency'`, `'both'`. Only `'both'` is ever used. There is no selector in the HTML to change this. The mode switch was built into the rendering function but never wired to a UI element.
**Impact:** Users cannot switch between interval-color view and frequency-density view. A documented feature exists invisibly in dead code.

---

### DEF-12 · SVG hardcodes theme colors — breaks on non-default themes
**File location:** `fretboardSVG()` lines 1677–1708, `heatMapSVG()` lines 2327–2355, `chordDiagramSVG()` line 1637
**Code (example):**
```js
svg += `<circle ... fill="#2a2a2f"/>`;          // fret markers
svg += `<rect ... fill="#c8c8cc" .../>`;        // nut
svg += `<text ... fill="#0e0e0f" .../>`;        // interval text in dots
svg += `<line ... stroke="#2a2a2f" .../>`;       // fret lines
svg += `<text ... fill="#6a6a78" .../>`;        // string labels
```
**Problem:** These are default-theme hex values. CSS variables (`var(--border)`, `var(--text-dim)`, `var(--bg)`) exist for exactly these roles but are not used inside SVG. When Gruvbox, Nord, Slate, or Low Light themes are active, fretboard backgrounds and nut/string colors do not adapt.
**Impact:** Theme switching is partially broken for all fretboard surfaces. The interval color dots (via `iv.color`) adapt correctly; everything else does not.

---

### DEF-13 · Tab content does not re-render after settings change
**File location:** `switchView()` — lines 1968–1973, `applySettings()` (settings change handlers)
**Problem:** When the user changes global tuning or fret count in Settings and then switches to a different tab, the tab content is not re-rendered with the new settings. It shows the last render state. The user must trigger a manual re-render (click a button) to see the updated output.
**Impact:** Settings changes appear to have no effect until the user discovers the manual trigger. Confusing behavior.

---

### DEF-14 · Theory tab Chromatic Spectrum hardcoded to C root
**File location:** `renderTheory()` — line 1939
**Code:**
```js
NOTES.forEach(n => {
  const iv = intervalBetween('C', n);  // always C
  chromCard += ...
});
```
**Problem:** The interval color swatches are always computed relative to C. The Theory tab has no awareness of what root is active in Chords or Scales. This is the Layer 8 gap, but it creates a specific misleading behavior: the user may be working in E minor and see the chromatic row with C as red/root.
**Impact:** Low severity now; becomes a lie once cross-tab continuity (Layer 7) is built.

---

## LOW

---

### DEF-15 · `inversionLabel` is declared but never used
**File location:** `renderChords()` — lines 1832, 1836–1838
**Code:**
```js
const inversionLabel = isInversion ? `<div class="chord-inversion">1st inv.</div>` : '';
// ...
let invText = '';
if (isInversion) {
  const bassInterval = intervalBetween(root, bassNote).symbol;
  invText = `<div class="chord-inversion">${bassInterval} in bass</div>`;
}
```
**Problem:** `inversionLabel` is computed and assigned but never interpolated into the HTML template. `invText` supersedes it. `inversionLabel` is dead code.

---

### DEF-16 · `VOICINGS` database defined but never referenced
**File location:** Lines 1483–1494
**Code:**
```js
const VOICINGS = {
  major: {
    C:  [{ name:'C (open)', pos:[null,'32010', 'x32010'] }, ...],
    ...
  }
};
```
**Problem:** This object is defined but no function ever reads from it. The algorithmic voicing engine (`computeVoicings`) replaced it completely. Adds ~15 lines to file size with no function.

---

### DEF-17 · Library mode field restricted to major/minor only
**File location:** Library form HTML — lines 1289–1292
**Code:**
```html
<select id="lib-mode">
  <option value="major">Major</option>
  <option value="minor">Minor</option>
</select>
```
**Problem:** `SCALE_FORMULAS` has 16 entries. Users recording modal songs (Dorian, Mixolydian, Phrygian) have no accurate mode field. The decoder analysis for a song labeled `major` but actually Mixolydian will misclassify the ♭VII as borrowed rather than diatonic.
**Note:** Expanding this requires updates to `getDiatonicChords` and `detectKey` as well — not a one-field change.

---

### DEF-18 · Delete has no confirmation — one misclick loses data
**File location:** `deletePreset()` — line 3061
**Code:**
```js
function deletePreset(id) {
  const idx = PRESETS.findIndex(p => p.id === id);
  if (idx < 0) return;
  PRESETS.splice(idx, 1);  // immediate, no confirm
  markPending();
  ...
}
```
**Problem:** A single click on `✕` permanently removes the song from the in-memory PRESETS array. Undo does not exist. The `beforeunload` warning fires only on tab/window close, not on delete. If the user forgets to save after accidental deletion, the data is gone.

---

## Summary Table

| ID     | Location                          | Severity | Blocks 2.6? |
|--------|-----------------------------------|----------|-------------|
| DEF-01 | `chordDiagramSVG` L1570–1572      | CRITICAL | No          |
| DEF-02 | `QUALITY_SUFFIXES` L2064          | CRITICAL | No          |
| DEF-03 | `analyzeChord` L2257              | CRITICAL | **YES**     |
| DEF-04 | `saveAndDownload` L3117           | CRITICAL | No          |
| DEF-05 | `computeVoicings` DFS L1521       | HIGH     | No          |
| DEF-06 | `detectKey` L2167                 | HIGH     | No          |
| DEF-07 | `fpChipsHtml` L3000               | HIGH     | No          |
| DEF-08 | `parseProgression` L2120          | HIGH     | No          |
| DEF-09 | `checkModalBorrowing` L2228       | HIGH     | Partial     |
| DEF-10 | `computeVoicings` span L1518      | MEDIUM   | No          |
| DEF-11 | `currentHeatMode` L2390           | MEDIUM   | No          |
| DEF-12 | SVG hardcoded colors              | MEDIUM   | No          |
| DEF-13 | `switchView` no re-render         | MEDIUM   | No          |
| DEF-14 | `renderTheory` C-root hardcode    | MEDIUM   | No          |
| DEF-15 | `inversionLabel` dead variable    | LOW      | No          |
| DEF-16 | `VOICINGS` dead object            | LOW      | No          |
| DEF-17 | Library mode field               | LOW      | No          |
| DEF-18 | `deletePreset` no confirm         | LOW      | No          |

---

## Recommendation Before 2.6

**Must fix before building Layer 4:**
- **DEF-03** — `func` field overloading. Layer 4 reads `analysis.func` to assign tendency labels. If secondary dominants have `func:'dominant'`, the labels will be wrong on delivery. Fix is surgical: change one return value.

**Recommend fixing in same pass as 2.6:**
- **DEF-04** — Save regex. Simple fix. Risk of data loss grows as the library grows.
- **DEF-09** — Modal borrowing gaps in minor keys. Layer 4 labels for borrowed chords depend on correct classification. Some borrowings will be misclassified as chromatic.

**Can defer to later beta:**
All remaining deficiencies are self-contained and do not interact with Layer 4 label logic.

