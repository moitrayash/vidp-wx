# Feathervane (VIDP/DEL) — Dev Log

Live: https://vidp.yashmoitra.com  ·  Repo: moitrayash/vidp-wx  ·  Single static `index.html` on GitHub Pages.

This file is the source of truth for **how this project is built and changed**. The
"Conventions" section governs all future additions — read it before editing.

---

## Conventions (rules for future additions)

1. **One dictionary, built by one cascade — no per-term patchwork.** The embedded `ABBR`
   list (~1137 entries) plus a small user-override layer (`OVR`) is the ONLY meaning source,
   read through a single helper `Dlook(key)`. Never hardcode a second divergent map and never
   sprinkle inline term overrides in the decode code. Add/correct terms in `OVR` (user-stated)
   or regenerate `ABBR` from source. See the decoding spec below for the authority cascade.

2. **Hover styling (applied everywhere a term is decoded).** A decoded/abbreviated term
   gets a dotted underline; on hover it shows a dark tooltip with the meaning AND the term
   text tints to a **slight dark blue** (subtle, not eye-piercing). Keep this consistent
   for METAR tokens and NOTAM tokens.

3. **Universal logic, not patchwork.** Any decode behaviour must be expressible as a rule that
   applies identically to all NOTAMs — current, expired, and future. If you find yourself
   "allow this one short token" or "fix this one word", that is patchwork: replace it with the
   general rule. Every change is backtested on the real expired-NOTAM corpus (see "Backtest
   procedure") before ship.

4. **Deploy by uploading the real file through the browser.** Commit directly to `main` as
   single isolated, revertable commits (the GitHub PR web UI repeatedly dropped the browser
   extension's host access mid-flow). The OneDrive/bash mount serves a STALE cached copy of the
   large `index.html`; the authoritative pipeline is: node-transform on a freshly fetched copy
   -> write to the session `outputs/` folder -> browser file-upload -> commit -> verify the
   deployed RUNTIME (not just the file).

5. **No em dashes anywhere in UI text.** Use colons or commas.

6. **Bump the "Last fix" build stamp on EVERY deploy.** The header line "Research Prototype ·
   Last fix <span id="lastFix">DD MON YYYY HHMMZ</span>" is a STATIC build/deploy stamp, not the
   live data-fetch time (the fetch->lastFix binding was deliberately removed). It means "when the
   app code was last changed". So on each code deploy, edit the literal date string in the
   `#lastFix` span to the current UTC deploy time (`date -u '+%d %b %Y %H%MZ'`). Do NOT re-wire it
   to any runtime clock or fetch time.

---

## Deploying for another aerodrome (replication)

The app is parameterized: **everything aerodrome-specific lives in one `CFG` block** at the top of
the main `<script>`. To stand up the app for a different AD, change ONLY `CFG`:

```
var CFG = {
  icao: 'VIDP',                                       // ICAO indicator -> METAR/NOTAM fetch, title, A-line
  iata: 'DEL',                                        // IATA code (display)
  appName: 'Feathervane',                             // product name (header + gate)
  airportName: 'Indira Gandhi International Airport',  // A-line friendly name
  fir: 'VIDF',                                         // home FIR code (Q-line)
  firName: 'Delhi FIR'                                // home FIR friendly name (Q-line)
};
```

`applyConfig()` (run at init) derives the page title, both `<h1>`s, the gate access note, and the
footer source links (`search <ICAO>`, `ids=<ICAO>`, `(<ICAO>)`) from `CFG`. `STATION`, `NLOC`
(A-line airport name) and `NFIR` (Q-line FIR name) also derive from `CFG`. Nothing else is
AD-specific, so the VIDP build is byte-for-byte unchanged after this refactor.

What does NOT need editing per-AD (already generic / national):
- **Decode engine** — `ABBR` (~1137 ICAO Doc 8400 + eAIP), `OVR`, `NPHRASE`, `NSTOP`, the regex
  groups, `Dlook`. All generic ICAO; identical for every AD worldwide.
- **Waypoint registry `WPT`** — the 775 five-letter fixes are India's national eAIP **ENR 4.4**
  list, so it is correct for *every Indian AD* (VIDP, VOMM, VABB, VECC, ...). Only for an AD in
  another country do you replace the `WPT` data set with that country's ENR 4.4 equivalent.
- Filters, status pills, collapse, fetch cadence, proxies, password gate mechanics.

Per-AD checklist: (1) edit `CFG`; (2) if non-India, swap `WPT`; (3) point the repo's `data.json`
GitHub Action at the new ICAO; (4) set the CNAME/subdomain. That's the whole port.

**Gotchas (verified by dry-running the decoder on real Bengaluru + Hyderabad NOTAMs):**
- **`CFG.fir` is NOT derivable from the ICAO. Look it up.** It only looked derivable for Delhi
  because Delhi *is* its own FIR (VIDP -> VIDF). India has only **four** FIRs; every AD maps to one:
  **VIDF** Delhi (north), **VABF** Mumbai (west), **VECF** Kolkata (east), **VOMF** Chennai (south,
  incl. Bengaluru VOBL, Hyderabad VOHS, Chennai VOMM). So for BLR and HYD, `fir:'VOMF',
  firName:'Chennai FIR'` — there is no "VOBF"/"VOHF".
- **Get the ICAO right** — it is not mechanical from the IATA. HYD = **VOHS** (Rajiv Gandhi,
  Shamshabad), not VOHY (old Begumpet). BLR = VOBL.
- **`data.json` fallback** ships with the previous AD's NOTAMs; until the Action is repointed
  (checklist #3), the offline fallback shows the wrong airport.
- Example configs: BLR = `{icao:'VOBL', iata:'BLR', appName:'Feathervane', airportName:'Kempegowda
  International Airport', fir:'VOMF', firName:'Chennai FIR'}`; HYD = same but
  `icao:'VOHS', iata:'HYD', airportName:'Rajiv Gandhi International Airport'`.
- Decode itself is clean on southern data (0 errors on 174 Chennai-NOF NOTAMs; the national ENR 4.4
  waypoints carry over). The only code change the southern data forced was the runway regex now
  tolerating **no space** (`RWY27`, `RWY09R/27L`) as well as `RWY 27` — different NOFs format
  runways differently; the pattern is `(?:RWY|TWY|TWYL|APN)\s*\d...`.

---

## NOTAM decoding specification (procedure ALL future NOTAMs/changes must follow)

This is the authoritative spec for how NOTAM text is decoded and rendered. Any new term,
pattern, or rendering change must conform to it.

**1. Authority cascade (one algorithm, top wins). `Dlook(key)` resolves every token meaning:**
   1. **User-stated corrections (`OVR`) = TOP authority.** Explicit values the user gave,
      which override the dictionary where the NOTAM-context sense differs from the literal
      ICAO entry. Current `OVR`: `FM=From`, `STAR=Standard terminal arrival route`,
      `DRG=During`, `WIE=With immediate effect`, `EQPT=Equipment`, `EST=Estimated`,
      `CAT=Category`. (Example why: ICAO Doc 8400's literal `FM` is the procedure-design term
      "Course from a fix to manual termination"; in NOTAM prose `FM` means "From".)
   2. **ICAO Doc 8400 (PANS-ABC)** parsed from the official PDF = primary dictionary.
   3. **India eAIP GEN 2.2** fills any term ICAO lacks, and is used when the ICAO parse is
      truncated (catches NOTAM -> "Notice to Airmen", RNAV -> "Area navigation").
   4. **No invented guesses.** A term in NEITHER list and not in `OVR` (e.g. **AOM**, **TXL**)
      is left PLAIN. Do not fabricate a meaning.
   - `ABBR` -> `NMAP` builds with NO term overrides in code; `OVR` is the only override layer.

**2. Single definitions only.** Every meaning is one definition (build trims at first " or "
   and " / " and strips trailing/unclosed parentheticals). Never show "X or Y" double meanings.

**3. The token rule is UNIVERSAL — no length whitelist (the old `NALLOW` patchwork is gone).**
   For each token: if it is in the English-collision **stoplist** (`NSTOP`) or shorter than
   **2 chars**, leave it plain; otherwise decode via `Dlook`. The ONLY filter is `NSTOP` =
   common English words (TO, DUE, FOR, AND, THE, NOT, FROM, WITH, SHALL, AS, IF, NO, UP, OF,
   ON, IN, ...). The dictionary was scanned against a broad English word list; the only
   collisions present — **LOSS, LINE, LONG, WIND** (plus BASE) — are stoplisted so e.g.
   "SIGNAL LOSS" does not become "Signal Airspeed". This one rule decodes all 2-char terms
   (FM, AD, GP, DH, ...) without per-token whitelisting.

**4. Underline word-SETS and structured patterns, not just words** (dedicated regex groups, in
   precedence order): multi-word phrases (NOT AVBL, WORK IN PROGRESS); dates ("DD MON [YYYY]");
   `RWY|TWY|TWYL|APN` + number incl. combined "11L/29R" -> one token; `CAT I/II/III` ->
   "Category ..." (precision approach, NOT clear-air turbulence); `NNNDEG` -> "NNN° (bearing/
   radial)"; **`FL` + 2-3 digits -> "Flight level NNN"**; **navaid ident** (`IDENT`/`DME` +
   code) -> keyword decoded + the code flagged as a Morse navaid identification (see 7).

**5. Dates are one group, never mis-read.** "DD MON [YYYY]" is a single Date token so its
   numbers are never taken as a runway/taxiway. B)/C) datetimes decode to readable UTC.

**6. Suffix handling — plural-S only; NO digit-strip.** If a token is not found, retry without
   a trailing "S" (SIDS->SID, STARS->STAR). The old digit-suffix base-strip was REMOVED: it
   produced false positives (TRA508->"Temporary reserved airspace", DP941->"Dew point", GEN2,
   ENR4, ILS25). Legitimate digit forms (FL/RWY/TWY/CAT/DEG) have their own patterns in (4);
   anything else with digits stays plain.

**7. Names and idents are not abbreviations.** 5-letter ICAO waypoint/fix names (BAVOX, MUNUV,
   OROTI) are left PLAIN. A navaid ident — the code right after `IDENT`/`DME` (DH, IDMR, IDGM)
   — renders with a tooltip explaining it is the Morse callsign identifying that navaid; it has
   no plain-language expansion. The ident rule overrides a dictionary collision (e.g. after
   "LOCATOR IDENT DH", DH is the ident, not "Decision height").

**8. PERM.** Permanent end dates render the word PERM in RED with a hover - in the card
   "Until:" line and in body text.

**9. Token interaction.** Decoded token = dotted underline; hover shows tooltip + slight
   dark-blue tint; clickable -> opens glossary sidebar, switches to Abbr tab, highlights the
   term (`abbrJump`). PERM stays red (not blue) on hover.

**10. Line breaks.** Rejoin FAA mid-sentence wraps (`normalizeNotam`); keep real paragraph and
   field-marker (Q/A/B/C/D/E) breaks. Render source text verbatim otherwise (an unbalanced ")"
   left by the originator is NOT edited).

**11. NOTAM list.** Filter UI = Include only A/B/G, Remove PERM, Active only (default on).
   Exclude FAA NOTAM-system error/diagnostic entries (RETCODE / ROLLED BACK / SEQUENCE CHECK /
   DUPLICATE NOTAM). Each card shows an Active / Upcoming / Expired status pill.

**12. House style.** No em dashes. Spell out units in prose. Hyperlink external sources.

## Backtest procedure (run before every decoder change)

The decoder is validated against a REAL corpus of expired NOTAMs, not synthetic samples.

- **Source:** AAI AIM India monthly NOTAM summaries —
  `https://aim-india.aai.aero/notam-summaries` -> Delhi, A and G series. Direct PDF pattern:
  `.../sites/default/files/notam_files/Delhi_A_2025_MM.pdf` (and `Delhi_G_2025_MM.pdf`).
  Each PDF lists every NOTAM still valid that month; together they are hundreds of real,
  now-expired VIDP/region NOTAMs.
- **Harness:** load the ACTUAL functions from the built `index.html` (extract the main
  `<script>`, cut the DOM `init` block, run under a minimal `document` shim via `vm`), then call
  the real `decodeNotamHTML('...\nE) ' + body)` on each parsed record. Do NOT test a hand-written
  replica of the logic — test the shipped code.
- **Checks:** 0 runtime errors; 0 records that fail to decode; NO digit-bearing false positives;
  every dictionary-key English word stoplisted; phrases (NOT AVBL) and dates stay intact; spot
  the feature cases (FM->From, GP, FL100, IDMR/IDGM/DH idents, AOM/TRA508/LOSS plain).
- **Last run (2026-06-06):** 306 expired A+G NOTAMs, 0 errors, 0 false positives.

## Architecture notes

- **Data:** every Refresh does a LIVE fetch. METAR from aviationweather.gov (+ fallbacks) via
  CORS proxies; NOTAMs from FAA NOTAM Search via POST proxies. `data.json` (GitHub Action) and a
  `localStorage` last-good cache are fallbacks so the page never blanks.
- **Auto-refresh:** METAR every 120 s, NOTAMs every 2 h; each has a manual button + countdown.
- **Access gate:** client-side password (soft), required every visit.
- **Decoders:** `decodeMetarHTML` and `decodeNotamHTML`; meaning lookups go through `Dlook`.

---

## Changelog

### 2026-06-08 (review round)
- **AOM** added to `OVR` = "Aerodrome operating minima" (real ICAO term, not in Doc 8400/eAIP
  abbreviation tables; user-directed, context-confirmed by "AOM BOX" on the IAP chart).
- **Waypoint recognition.** Embedded the 775 five-letter significant-point name-codes from
  **eAIP India ENR 4.4** (lo-alt + hi-alt ATS route fixes) as `WPT`. A 5-letter token is tagged
  "Waypoint" ONLY if it is in `WPT` (membership check, no guessing). Terminal SID/STAR-only fixes
  not on any airway (e.g. MUNUV) and procedure designators (e.g. OROTI6C) stay plain by design.
  Re-pull ENR 4.4 when airways change; navaid idents (IDMR/IDGM/DH after IDENT/DME) remain idents.
- **Collapsible NOTAM cards** + Collapse all / Expand all controls. Header click toggles; number
  stays left-aligned with a small triangle caret to its right.
- **"Last fix" is now a static build stamp** (see Convention 6) instead of the live fetch time.

### 2026-06-06 (universal decode rule + expired-NOTAM backtest)
- **Eliminated the length-whitelist patchwork.** Removed the `w.length < 3 && !NALLOW[k]`
  cutoff and the `NALLOW` whitelist entirely. New universal rule: decode any token >= 2 chars
  that is in `Dlook` and not in `NSTOP`. Fixes FM, AD, GP, DH and all ~164 short entries at
  once (FM = "From" via `OVR`).
- **Added the `OVR` user-override layer + `Dlook` cascade** (top authority over ICAO/eAIP),
  documented above. FM/STAR/DRG/WIE/EQPT/EST/CAT.
- **Added FL and navaid-ident patterns.** `FL100` -> "Flight level 100"; `IDENT/DME` + code ->
  the code flagged as a navaid Morse ident (IDMR, IDGM, and DH-as-locator-ident).
- **Removed the digit-suffix base-strip** — it caused false positives (TRA508, DP941, GEN2,
  ENR4, ILS25). Real digit forms have dedicated patterns; plural-S strip retained.
- **Stoplisted the 4 English-word dictionary collisions** found by a broad scan: LOSS (had a
  junk "Airspeed" parse), LINE, LONG, WIND.
- **Backtested on 306 real expired NOTAMs** (AAI Delhi A+G 2025 summaries) with the actual
  deployed `decodeNotamHTML`: 0 runtime errors, 0 decode failures, 0 digit false positives.
  Live runtime re-verified after deploy.
- **Correction:** AOM is in neither ICAO Doc 8400 nor eAIP GEN 2.2, so per the no-guess rule it
  is now left PLAIN (the earlier "Aerodrome operating minima" value was a guess and was removed).

### 2026-06-05 (earlier rounds)
- Underlining engine (word-sets, dates, navaid idents, PERM red, hover tint, click->sidebar),
  NOTAM filter dropdown + Active/Upcoming/Expired pills, ICAO Doc 8400 parsed and merged to a
  1137-entry ICAO-primary `ABBR` map, phrase-boundary fix, meta "i" badge, title "VIDP:
  Feathervane", single-definition cleanup, line-break rejoining. (See git history for detail.)

### Earlier milestones
- Built single-file app; deployed to GitHub Pages with Wix CNAME (vidp.yashmoitra.com) + HTTPS.
- Live-fetch refresh with multi-source METAR + CORS proxies + data.json + last-good cache.
- Glossary sidebar; password gate; dual auto-refresh cycles.

---

## Abbreviation corrections log

User-stated meanings (in `OVR`, top authority) and terms intentionally left plain:

| Term | Action | Meaning / Reason |
|------|--------|--------|
| FM   | OVR meaning | "From" (ICAO literal is a procedure-design term; prose sense applies). |
| STAR | OVR meaning | "Standard terminal arrival route" (user-set). |
| DRG  | OVR meaning | "During". |
| WIE  | OVR meaning | "With immediate effect". |
| EQPT | OVR meaning | "Equipment". |
| EST  | OVR meaning | "Estimated". |
| CAT  | OVR meaning | "Category" (precision-approach context, not clear-air turbulence). |
| AOM  | left PLAIN  | Not in ICAO Doc 8400 nor eAIP GEN 2.2 -> no guess. |
| TXL  | left PLAIN  | Not in either authoritative list -> no guess. |
| LOSS / LINE / LONG / WIND | stoplisted | Common English words that collide with dictionary keys. |
| TO / DUE | stoplisted | Common English words; redundant in context. |
| IDMR / IDGM / DH (after IDENT/DME) | ident | Navaid Morse callsign; not an abbreviation, no expansion. |
| BAVOX / MUNUV / OROTI | left PLAIN | 5-letter ICAO waypoint/fix names. |
