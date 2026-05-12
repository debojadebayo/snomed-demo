# SNOMED CT Demo — Implementation Plan

_2026-05-12_

## Overview

A single `index.html` file that viscerally demonstrates the performance gap between a FHIR terminology server API call (how most production health tech systems look up SNOMED concepts today) and a compiled perfect-hash lookup embedded in the application binary. Target audience: engineers and clinical informatics leads at health-tech vendors, reached via cold LinkedIn DM.

---

## Desired End State

Opening `index.html` directly in a browser (no server, fully offline) shows:

1. A dark, clinical two-panel layout with a dropdown of 8 SNOMED concepts and a "Run lookup" button.
2. Clicking "Run lookup" triggers both panels simultaneously.
3. Left panel animates an HTTP request lifecycle in the style of a browser DevTools network waterfall — DNS, TCP connect, TLS, request sent, waiting (TTFB), content download — with the simulated timer counting to ~51ms. The TTFB bar is visibly the dominant phase.
4. Right panel flashes through a four-step pipeline (input → hash → offset → result) in under 200ms real time, displaying "0.4 μs" on completion.
5. After both panels finish, a centred "~125,000× faster" badge appears between the panels.
6. A footer explains what a perfect hash for SNOMED is and why the API call is eliminated entirely. Attribution: "built by @debojadebayo".

### Verification

- [ ] Open `index.html` directly in Chrome, Firefox, and Safari (no server)
- [ ] All 8 concept lookups animate correctly without console errors
- [ ] Button is disabled during animation, re-enabled after
- [ ] Ratio badge only appears after both panels complete, resets on next run
- [ ] Fonts load from CDN; page still renders legibly offline via `monospace` fallback

---

## What We Are NOT Doing

- No real SNOMED CT data embedded — concept IDs and names are curated constants
- No free-text search or fuzzy matching
- No server, no build step, no npm, no bundler
- No mobile-optimised layout (fixed two-column)
- No toggleable "in-memory HashMap" comparison mode

---

## Implementation: Single Phase

Everything ships as one `index.html`. Four logical sections: **Data**, **CSS**, **HTML structure**, **JS animation engine**.

---

### Section 1 — Data (inline JS constants)

**8 pre-loaded SNOMED concepts** (real concept IDs):

| Display name | Concept ID |
|---|---|
| Myocardial infarction | 22298006 |
| Type 2 diabetes mellitus | 44054006 |
| Essential hypertension | 59621000 |
| Asthma | 195967001 |
| Atrial fibrillation | 49436004 |
| Sepsis | 91302008 |
| Acute kidney injury | 40095003 |
| Community-acquired pneumonia | 385093006 |

Each concept entry also stores a simulated memory offset string (e.g. `"0x1f4a2"`) derived from the concept ID, used by the right panel.

**HTTP request phases** (fixed sequence, same for all concepts, ±2ms jitter applied at runtime):

| Phase | Label shown | Simulated latency | Real-time animation delay |
|---|---|---|---|
| DNS lookup | `dns lookup` | 2 ms | 200 ms |
| TCP connect | `tcp connect` | 1 ms | 150 ms |
| TLS handshake | `tls handshake` | 3 ms | 250 ms |
| Request sent | `request sent` | 1 ms | 150 ms |
| Waiting (TTFB) | `waiting (ttfb)` | 41 ms | 1200 ms |
| Content download | `content download` | 3 ms | 300 ms |
| **Total** | | **~51 ms** | **~2.25 s** |

The TTFB bar should be rendered visibly wider/longer than all other phases to make the bottleneck obvious at a glance.

---

### Section 2 — CSS

**Design tokens** (CSS custom properties on `:root`):

```css
--bg:         #0b0d12   /* page background */
--surface:    #111520   /* panel background */
--border:     #1c2333   /* panel borders */
--text:       #d4d8e0   /* primary text */
--muted:      #4a5568   /* secondary / dim text */
--amber:      #f5a524   /* left panel accent (slow = hot) */
--amber-dim:  #7a5012   /* inactive phase bar fill */
--amber-bar:  #c47d10   /* active phase bar colour */
--mint:       #16d098   /* right panel accent (fast = cool) */
--mint-dim:   #0a4535   /* pipeline step default fill */
--white:      #f0f2f5   /* result values */
```

**Font**: `'JetBrains Mono'` loaded from Google Fonts CDN. Fallback: `'Courier New', monospace`.

**Layout**:
- `body`: dark bg, flex column, min-height 100vh, padding 24px
- `header`: narrow strip — title left, controls (`<select>` + button) right, border-bottom
- `main.panels`: flex row, gap 0, flex: 1
- Each `.panel`: flex: 1, surface bg, border, padding 24px, display flex column, gap 12px
- `.divider`: 1px vertical border `--border`, position relative; `.ratio-badge` absolute-centred, hidden until both complete
- `footer`: border-top, padding-top 16px, muted text, two lines

**Network waterfall row** (`.phase-row`):
- Two columns: label (fixed 160px, muted, right-aligned) + bar container (flex: 1)
- Bar: `height: 20px`, border-radius 3px, `background: --amber-dim`; inner `.bar-fill` grows from 0 to its proportional width via CSS transition when phase completes
- Active phase: bar fill animates from 0% → full width over the real-time delay; label turns amber
- Completed phase: label stays amber, bar full; faint amber border on row

**TTFB bar**: proportional width is `(41/51) * 100%` ≈ 80% of the available bar container — visually dominant.

**Pipeline step styling** (right panel, `.pipe-step`):
- Default: `--mint-dim` bg, `--muted` border, `--muted` text
- Active: `--mint` border + mint drop-shadow, `--mint` text
- Complete: `--mint` border, `--white` text

**Button**: amber border, dark bg, `--amber` text on hover; disabled: muted, cursor not-allowed.

**Ratio badge**: large monospace text, `--white` colour, fade-in 300ms CSS transition.

---

### Section 3 — HTML Structure

```html
<header>
  <span class="title">SNOMED CT // Lookup Benchmark</span>
  <div class="controls">
    <select id="concept-select"><!-- options injected by JS --></select>
    <button id="run-btn">Run lookup</button>
  </div>
</header>

<main class="panels">

  <!-- LEFT PANEL -->
  <section class="panel panel-left">
    <div class="panel-label">
      Terminology Server API
      <span class="tag">FHIR $lookup · Snowstorm</span>
    </div>

    <div class="request-url" id="request-url">
      <!-- animated in by JS: GET /fhir/CodeSystem/$lookup?system=...&code=... -->
    </div>

    <div class="waterfall" id="waterfall">
      <!-- phase rows injected by JS, one per HTTP phase -->
    </div>

    <div class="result-row hidden" id="left-result">
      <span class="result-id"></span>
      <span class="result-name"></span>
    </div>

    <div class="timer" id="left-timer">—</div>
  </section>

  <!-- DIVIDER + RATIO -->
  <div class="divider">
    <div class="ratio-badge hidden" id="ratio-badge">
      ~125,000×<br><span class="ratio-sub">faster</span>
    </div>
  </div>

  <!-- RIGHT PANEL -->
  <section class="panel panel-right">
    <div class="panel-label">
      Compiled Perfect Hash
      <span class="tag">embedded · zero I/O</span>
    </div>

    <div class="pipeline" id="pipeline">
      <div class="pipe-step" id="step-input">
        <span class="step-label">input</span>
        <span class="step-value" id="input-val">—</span>
      </div>
      <div class="pipe-arrow">↓</div>
      <div class="pipe-step" id="step-hash">
        <span class="step-label">perfect_hash()</span>
        <span class="step-value">deterministic slot</span>
      </div>
      <div class="pipe-arrow">↓</div>
      <div class="pipe-step" id="step-offset">
        <span class="step-label">memory offset</span>
        <span class="step-value" id="offset-val">—</span>
      </div>
      <div class="pipe-arrow">↓</div>
      <div class="pipe-step" id="step-result">
        <span class="step-label">result</span>
        <span class="step-value" id="right-result-val">—</span>
      </div>
    </div>

    <div class="timer" id="right-timer">—</div>
  </section>

</main>

<footer>
  <p class="explainer">
    Most health systems resolve SNOMED concepts via a terminology server API (Snowstorm, Ontoserver, or similar) — a network round-trip on every lookup.
    SNOMED CT is a closed, versioned vocabulary (~360 000 concepts): a perfect minimal hash compiled per release maps every concept identifier directly to a memory slot.
    No network call. No I/O. No comparisons. The structure is rebuilt when a new SNOMED release ships.
  </p>
  <p class="attribution">built by <a href="https://linkedin.com/in/debojadebayo" target="_blank">@debojadebayo</a></p>
</footer>
```

---

### Section 4 — JavaScript Animation Engine

**Initialisation** (`DOMContentLoaded`):
1. Populate `<select>` options from the concepts array (value = index, text = display name).
2. Inject waterfall rows into `#waterfall` — one `.phase-row` per HTTP phase, bars at 0 width.
3. Attach `click` on `#run-btn` → `runLookup()`.

**`runLookup()`**:
1. Read selected concept.
2. Disable `#run-btn`.
3. Reset state: hide ratio badge, clear timers, reset all phase rows (bars to 0, labels dim), reset pipeline steps, clear `#request-url`, hide `#left-result`.
4. Kick off `leftAnimation(concept)` and `rightAnimation(concept)` concurrently.
5. `Promise.all([leftDone, rightDone])` → reveal `#ratio-badge` (fade in), re-enable button.

**`leftAnimation(concept)`** — returns a Promise:

```
1. Animate request URL appearing character-by-character (or fade-in, 200ms):
   "GET /fhir/CodeSystem/$lookup
    ?system=http://snomed.info/sct
    &code={concept.id}"

2. For each phase in sequence:
   a. Set phase row label to amber (.active)
   b. Animate bar fill from 0 → proportional width over the real-time delay
      (CSS transition duration = realTimeDelay ms)
   c. Increment displayed simulated timer by phase's simulated latency
   d. Wait for real-time delay to complete (Promise/setTimeout)
   e. Mark phase complete (label stays amber, bar stays full)

3. After all phases:
   - Show #left-result with concept ID + name
   - Set #left-timer: "51 ms  ·  FHIR API round-trip"
```

Simulated timer updates happen at each phase completion — displayed as integer ms climbing from 0 to ~51.

**`rightAnimation(concept)`** — returns a Promise:

```
60ms delay between each step (real time):

step-input  → active: display concept name
step-hash   → active: "computing slot..."
step-offset → active: display concept.offset (e.g. "0x1f4a2")
step-result → active: display "{concept.id}  {concept.name}"

After step-result:
  #right-timer: "0.4 μs  ·  perfect hash"
```

**Timing display**:
- Left (during run): `"4 ms"`, `"7 ms"`, `"13 ms"`, ... (integer ms, increments at each phase completion)
- Left (final): `"51 ms  ·  FHIR API round-trip"`
- Right (final): `"0.4 μs  ·  perfect hash"`

**Jitter**: at the start of each `runLookup()`, generate a random `±2ms` jitter value and apply it to the TTFB simulated latency only (so total varies between ~49–53ms). Keeps repeated runs feeling live.

---

## Success Criteria

### Automated:
- [ ] No JS errors on load (browser devtools console clean)
- [ ] No JS errors running any of the 8 concepts back-to-back

### Manual:
- [ ] Open from disk (`file://`) — page renders, fonts load or fallback gracefully
- [ ] TTFB bar is visibly the widest phase bar
- [ ] Left animation takes ~2–2.5s to complete; clearly step-by-step
- [ ] Right animation completes in under 300ms; feels instant vs left
- [ ] Ratio badge fades in after both complete; hidden at start of next run
- [ ] Button disabled during run, re-enabled after
- [ ] No horizontal overflow at 1280px viewport
- [ ] Footer text and @debojadebayo link render correctly

---

## References

- Source of truth for concept IDs: SNOMED International browser (snomed.org)
- Snowstorm terminology server: SNOMED International open-source reference implementation (Elasticsearch-backed)
- `idea.md`: `/Users/debo/github.com/debojadebayo/snomed-demo/idea.md`
