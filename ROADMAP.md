# TONE — Roadmap
*Layer order is enforced. A layer cannot be built until all layers beneath it are complete.*
*This document defines what "done" means at each layer.*

---

## The Three-Layer Model — What Every Layer Is Building Toward

```
PHYSICAL LAYER       The guitar as constrained interface
                     Six strings, fixed intervals, finite frets, four fingers
                           ↕ bridged by the human ↕
LOGICAL LAYER        Relationships between notes
                     Intervals, chords, scales, keys, modes, harmonic function
                           ↕ bridged by the human ↕
EMOTIONAL LAYER      What the relationships tend to produce
                     Not housed in chords — emergent from context
```

Layers 0–3 built the logical layer. Layers 4–6 connect it to the emotional and physical layers. Layers 7–9 make the human the seamless bridge between all three.

---

## Completed Layers

### Layer 0 — Foundation ✓
- 12-note chromatic math engine
- Interval color system (12 colors, one per semitone, consistent across entire app)
- noteIndex, noteAt, intervalInfo, intervalBetween functions

### Layer 1 — Reference Surfaces ✓
- Chord voicing engine with SVG diagrams
- Scale fretboard visualization
- Theory reference (interval spectrum, chord formulas, scale step patterns)
- Multiple tunings, global settings, 5 color schemes

### Layer 2 — Harmonic Analysis ✓
- Key detection algorithm (AUTO mode)
- Chord function analysis: diatonic / borrowed / secondary dominant / chromatic
- Fretboard heat map (interval color + frequency opacity)
- Manual key selection, source labels (Diatonic / Auto-detected / Preset)

### Layer 3 — Pattern Recognition ✓
- Harmonic fingerprint engine (Roman numeral sequence computation)
- Four-tier match engine: Exact / Transposed / Close / Distant
- Song library with full CRUD
- Compare modal (side-by-side deviation analysis)
- Save system: PRESETS serialized into HTML source, timestamped download
- Library: flat song list, dual real-time filter, colored fingerprint chips per row

---

## Active Layer

### Layer 4 — Emotional Tendency (Pillar 1)

#### Beta 2.6 — Tendency labels on Decoder chord cards ✓ COMPLETE
All chord function cards in the Decoder show emotional tendency translations beneath the academic label. Eight `func` values covered. Quality overrides for maj7/min7/dom7 sourced from Kolchinsky et al. 2017. All labels use "tends toward" framing.

#### Beta 2.7 — Forward mapping panel ← CURRENT TARGET
**Definition of done:**
- Panel appears in Decoder showing full emotional tendency palette for active key + mode
- For each tendency quality: which chords and intervals in the current key tend toward it
- Vocabulary derived entirely from Beta 2.6 label system — no new emotional terms introduced
- Interval color system used throughout
- Panel does not appear before Beta 2.6 labels are confirmed working

---

## Queued Layers

### Layer 5 — Scale Context (Pillar 2)
**Definition of done:**
- Neighboring scales panel on Scales tab
- For active scale: shows every scale that differs by exactly one note
- Each neighbor labeled with: scale name, which note changes, tendency shift that results
- Pentatonic minor → Natural minor relationship explicitly shown
- Natural minor → Dorian relationship explicitly shown (the Santana/SRV split)
- Major → Mixolydian relationship explicitly shown (triumphant but restless)
- Interval color system used for the changed note
- All descriptions use tendency framing consistent with Layer 4 vocabulary

### Layer 6 — Playability Validation (Pillar 3)
**Definition of done:**
- Playability score computed for every generated voicing before display
- Hard suppression: voicings requiring span > 5 frets are not displayed
- Hard suppression: voicings requiring more than 4 simultaneous fretted fingers are not displayed
- Flagged (displayed with warning): barre chord requirement
- Flagged (displayed with warning): string skip greater than one string
- Fret-position-aware span calculation (spans widen in lower positions)
- Open strings preserved as valid
- No valid playable voicing suppressed as a side effect of the filter

### Layer 7 — Cross-Tab Continuity
**Definition of done:**
- A shared _activeContext object holds: key root, mode, active chord (optional)
- Decoder key selection propagates to Scales tab (same root/mode pre-selected)
- Decoder key selection propagates to Chords tab (same root pre-selected)
- Navigation between tabs does not reset active context
- User can override context within any tab without affecting others

### Layer 8 — Theory Tab Activation
**Definition of done:**
- Theory tab reads _activeContext and highlights relevant intervals
- Interval spectrum shows which intervals are active in current key
- Chord formula section highlights active chord quality if one is selected
- Color system legend displayed inline on each tab (not only in Theory)
- Tendency vocabulary from Layer 4 surfaced inline alongside academic labels

### Layer 9 — Reverse Emotion Engine (Capstone)
**Definition of done:**
- Input: emotional tendency target selected from the Layer 4 vocabulary
- Output: specific chords, scales, intervals, and fretboard positions that tend toward that quality
- Vocabulary is the Layer 4 system — not invented fresh, not genre-specific
- Results are playable (Layer 6 filter applied)
- Results shown on fretboard using interval color system
- Context modifiers surfaced: tempo, mode, chord quality all shown as factors that shift tendency
- This feature does not exist elsewhere at this level of specificity or theoretical honesty

---

## Open Technical Debt

*Deferred deficiencies from the Beta 2.5 audit. None are currently blocking. Reviewed before each new layer build. Resolved DEFs are moved to the version log.*

| DEF | Location | Severity | Issue | Resolves in |
|-----|----------|----------|-------|-------------|
| DEF-05 | `computeVoicings` DFS | HIGH | Inner strings cannot be muted — valid voicings never generated | Layer 6 (Pillar 3) |
| DEF-10 | `computeVoicings` span check | MEDIUM | Span filter position-naive — 4-fret limit same at fret 1 and fret 12 | Layer 6 (Pillar 3) |
| DEF-11 | `currentHeatMode` | MEDIUM | Heat map mode locked to `'both'` — `no UI toggle for interval/frequency view | Own pass, not layer-gated |
| DEF-14 | `renderTheory` | MEDIUM | Chromatic Spectrum hardcoded to C root — does not track active key | Layer 8 (cross-tab continuity prerequisite) |
| DEF-17 | Library mode field | LOW | Mode dropdown major/minor only — modal songs cannot be accurately recorded | Architectural — needs `getDiatonicChords` + `detectKey` updates |

**Cascade check for Beta 2.7 (forward mapping panel):**
DEF-05, DEF-10 → touch `computeVoicings`, not read by forward panel → deferred, not blocking
DEF-11, DEF-14, DEF-17 → touch heat map / theory / library, not read by forward panel → deferred, not blocking
**Beta 2.7 has no blocking debt. Clear to build.**

---

## Future Considerations (Not Scheduled)

**Harmonic decision moment detector:** Decoder mode that identifies which chord in a progression is the pentatonic danger point and surfaces the correct scalar response. The SRV → Alter Bridge bridge. Requires Layers 4 and 5 complete.

**Library data verification:** Cross-reference user-entered progressions against established sources. AI-based audio analysis deferred — contentious, pay-for-service, error-prone.

**Android packaging:** Capacitor wrapper. Only begins after Layer 9 complete or explicitly scheduled.

---

## Version Log

| Beta | Layers Complete | Notes |
|------|----------------|-------|
| 1.x  | 0, 1           | TiddlyWiki migration, native HTML, chord/scale/theory tabs |
| 2.0  | 2              | Decoder, key detection, chord analysis, heat map |
| 2.1  | 2 (extended)   | Manual key override, PRESETS architecture |
| 2.2  | 3              | Fingerprint engine, match panel, compare modal, library, save system |
| 2.3  | 3 (extended)   | Library edit mode |
| 2.4  | 3 (extended)   | Library song-first view, colored fingerprint chips |
| 2.5  | 3 (extended)   | Library dual filter |
| 2.5.1 | 3 (extended) | Pre-2.6 patch: DEF-03 secondary dominant func field, DEF-09 minor borrowing expanded (3→10 cases), DEF-04 save regex anchored |
| 2.5.2 | 3 (extended) | Deficiency patch: DEF-01 baseFret SVG fix, DEF-06 detectKey minor tiebreaker, DEF-07 V/X chip color, DEF-08 slash chord parse, DEF-12 SVG theme colors via getSvgColors(), DEF-13 switchView re-render, DEF-15/16 dead code removed, DEF-18 delete confirmation |
| 2.6  | 4 (partial)    | Emotional tendency labels on Decoder chord cards — Pillar 1 stage 1 |
| 2.7  | 4 (complete)   | Forward mapping panel — Pillar 1 stage 2 |
