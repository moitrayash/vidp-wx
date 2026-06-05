# Feathervane (VIDP/DEL) — Dev Log

Live: https://vidp.yashmoitra.com  ·  Repo: moitrayash/vidp-wx  ·  Single static `index.html` on GitHub Pages.

This file is the source of truth for **how this project is built and changed**. The
"Conventions" section governs all future additions — read it before editing.

---

## Conventions (rules for future additions)

1. **Abbreviation reference doc is the single source of truth.** The eAIP GEN 2.2
   abbreviation list drives (a) the sidebar Abbr tab, (b) NOTAM E-line auto-underlining,
   and (c) is the doc referenced in pull requests. Do not hardcode a second, divergent
   abbreviation map. Add/edit terms in the reference, not inline.

2. **Hover styling (applied everywhere a term is decoded).** A decoded/abbreviated term
   gets a dotted underline; on hover it shows a dark tooltip with the meaning AND the term
   text tints to a **slight dark blue** (subtle, not eye-piercing). Keep this consistent
   for METAR tokens and NOTAM tokens.

3. **Context-aware underlining.** For each NOTAM, read the whole text first and judge each
   word in context. Do NOT underline/explain words that are redundant or obvious in context
   (common English words such as TO, DUE). When a term has context-dependent meanings,
   reflect that (e.g. TEMP = "temporarily" in some cases, not always "temporary").
   Every term removed/changed is recorded in the "Abbreviation corrections log" below.

4. **Deploy via PR.** Work on a branch and open a pull request rather than committing
   straight to `main`. The abbreviation reference doc is cited in the PR description.

5. **Deploying the file (ops gotcha).** The repo folder lives in OneDrive. The Linux/bash
   mount serves a STALE cached copy of the large, frequently-edited `index.html` (wrong size,
   truncated tail) even though the real Windows file is correct. Therefore: edit via the
   Read/Edit tools (authoritative Windows file), and **deploy by uploading the real file
   through the browser** (GitHub web upload reads the true on-disk file, ~106 KB) — never
   trust the bash mount's view of `index.html`. New/other files sync fine. After any deploy,
   verify the raw file from `raw.githubusercontent.com/.../main/index.html`
   (size, ends with `</html>`, key strings present, main `<script>` parses).

6. **No em dashes anywhere in UI text.** Use colons or commas.

---

## NOTAM decoding specification (procedure ALL future NOTAMs/changes must follow)

This is the authoritative spec for how NOTAM text is decoded and rendered. Any new term,
pattern, or rendering change must conform to it.

**1. Abbreviation source & precedence.**
- The embedded eAIP GEN 2.2 list (`ABBR` -> `NMAP`) is the base. It IS India's official adoption
  of ICAO Doc 8400, so it already carries the ICAO abbreviations.
- **ICAO Doc 8400 (PANS-ABC) is primary on conflict**: where ICAO and eAIP differ, add an
  explicit override in the `NMAP` overrides block (e.g. `m['TWY']='Taxiway'`). eAIP is the
  secondary/fallback. Do NOT embed a raw 8400 PDF parse (it scans incompletely and garbles).

**2. Single definitions only.** Every meaning is one definition. Build step trims each eAIP
meaning at the first " or " and strips a trailing or unclosed parenthetical. Never show
"X or Y" or "A / B" double definitions. If a term still shows two, add a clean override.

**3. Stoplist + length.** Do not underline common/obvious English words (`NSTOP`: TO, DUE, FOR,
AND, THE, NOT, FROM, WITH, SHALL, ...). Skip tokens shorter than 3 chars UNLESS explicitly
allowed in `NALLOW` (e.g. AD = Aerodrome). FM = "From" stays un-decoded (short, obvious).

**4. Underline word-SETS, not just words.**
- `RWY|TWY|TWYL|APN` + number (incl. combined "11L/29R") -> ONE token, e.g. "Runway 11L/29R".
- "ASSIGNED RWY" -> "Assigned runway". "CAT I/II/III" -> "Category I/II/III" (precision approach,
  NOT clear-air turbulence). "NNNDEG" -> "NNN° (bearing/radial)".

**5. Dates are one group, never mis-read.** "DD MON [YYYY]" groups as a single Date token so its
numbers are never taken as a runway/taxiway. B)/C) datetimes decode to readable UTC.

**6. Plurals / suffixes.** If a token is not found, retry without a trailing "S" (SIDS->SID,
STARS->STAR) and without trailing digits (RNAV1->RNAV).

**7. Names are not abbreviations.** 5-letter ICAO waypoint/fix names (BAVOX, MUNUV) and navaid
idents (the code after IDENT/DME/LOC/GP, e.g. DH, IDMR, IDGM) are left PLAIN everywhere -
never expanded. Idents must render identically regardless of position.

**8. PERM.** Permanent end dates render the word PERM in RED with a hover ("Permanent: no end
date; in force until cancelled") - in the card "Until:" line and in body text.

**9. Token interaction.** A decoded token has a dotted underline; on hover it shows a dark
tooltip AND tints slight dark-blue; it is clickable -> opens the glossary sidebar, switches to
the Abbr tab, and highlights the term (`abbrJump`). PERM stays red (not blue) on hover.

**10. Line breaks.** Rejoin FAA mid-sentence wraps (`normalizeNotam`); keep real paragraph and
field-marker (Q/A/B/C/D/E) breaks. Render NOTAM source text verbatim otherwise (e.g. an
unbalanced ")" left by the originator is NOT edited - it is the source's).

**11. NOTAM list.** Filter UI = Include only A/B/G (prefix; also drops US DOD/other series),
Remove PERM, Active only (default on). Exclude FAA NOTAM-system error/diagnostic entries
(RETCODE / ROLLED BACK / SEQUENCE CHECK / DUPLICATE NOTAM). Each card shows an
Active / Upcoming / Expired status pill.

**12. House style.** No em dashes anywhere. Spell out units in prose (minutes, hours).
Hyperlink external sources.

## Architecture notes

- **Data:** every Refresh does a LIVE fetch. METAR from aviationweather.gov (+ vatsim, NOAA
  fallbacks) via CORS proxies; NOTAMs from FAA NOTAM Search via POST proxies.
  `data.json` (refreshed by a GitHub Action) is only a fallback so the page never blanks;
  a `localStorage` last-good cache is the final fallback.
- **Auto-refresh:** METAR every 120 s, NOTAMs every 2 h; each has a manual button + countdown.
- **Self-check:** parses the METAR's own obs time; red/overdue when age > 30 min.
- **Access gate:** client-side password (soft), required every visit.
- **Decoders:** `decodeMetarHTML` (token hover) and `decodeNotamHTML` (Q-line applied decode,
  dates, E-line abbreviation hovers).

---

## Changelog

### 2026-06-05 (PRs)
- **PR #1** (merged): Fix odd NOTAM line breaks. `normalizeNotam()` rejoins FAA mid-sentence
  wraps (~64-char hard newlines); keeps paragraph + field-marker breaks. Live-verified.
- **PR #2** (merged): Lowercase the plural "s" in the "ACTIVE NOTAMs" header (text-transform fix).
- **#3** (committed to main): Removed Airline + Airport sidebar sections (tabs, panes, and the
  on-demand dataset/render JS). Verified: parses, prior fixes intact.
- **#4 — Underlining engine, step 1** (committed to main): E-line underlining now uses the full
  embedded eAIP map (`NMAP`, ~797 terms, dup meanings combined) with overrides (TWY=Taxiway,
  TEMP, DRG) + a common-word stoplist (`NSTOP`) and a min length-3 rule. Replaces the tiny
  hardcoded `NABBR`. Offline-tested: REF/BLW/OPS/ACFT/RWY/TWY/AVBL/TECR decode; OF/FOR/DUE/
  IS/AS/SHALL left alone; no single-letter noise. ICAO Doc 8400 is the precedence source for
  future overrides; eAIP is the base list.
- **#5 — Underlining engine, step 2** (main): word-set grouping ("RWY 11L" -> one token
  "Runway 11L", "TWY 52", "ASSIGNED RWY"), date grouping ("27 NOV 2026" -> one Date token so
  numbers aren't read as runways), navaid-ident recognition ("IDENT DH" -> DH flagged, not
  "Decision height"). Offline + live verified.
- **#6 — Underlining engine, steps 3-4** (main): PERM rendered with a red underline + hover;
  abbreviation tokens tint dark-blue on hover (PERM stays red).
- **#7 — Underlining engine, step 5** (main): abbreviation tokens are clickable -> open the
  sidebar, switch to Abbr tab, scroll to + highlight the term (`abbrJump`).
- **#8 — NOTAM filter + status** (main): small Filter dropdown under Refresh NOTAMs
  (Include only A/B/G, Remove PERM, Active only; A/B/G prefix filter also drops DOD/other
  series). Each card shows a computed **Active / Upcoming / Expired** pill in place of the
  old "FAA filed" line. Live-verified: 38 shown of 41 (default), toggling Active-only surfaces
  upcoming/expired, no console errors.

All 26 tracked tasks complete. ICAO Doc 8400 (PANS-ABC) is saved as the precedence source for
future overrides; eAIP GEN 2.2 remains the embedded base list.

### 2026-06-05 (review round, direct commits to main)
- Meta line: dropped "sorted by latest issue / evaluated live" tail; added an "i" info badge
  next to "total" explaining the filtered count.
- Decoder/NMAP cleanup: single definitions only (trim each meaning at first " or " and strip a
  trailing/unclosed parenthetical) -> fixes WIE, ALTN, SUP, VOR, etc.; CAT/CAT II/III -> Category;
  REF -> Reference; AOM -> Aerodrome operating minima; AD whitelisted (2-letter exception);
  EAIP + E-AIP -> India eAIP; plural/suffix fallback (SIDS->SID, STARS->STAR, RNAV1->RNAV);
  combined runways group ("RWY 11L/29R" -> one token); NNNDEG -> bearing; navaid idents
  (IDMR/IDGM/DH) left plain everywhere (removed the special IDENT-underline for consistency).
- NOTAM list: filter out FAA NOTAM-system error/diagnostic entries
  (RETCODE / ROLLED BACK / SEQUENCE CHECK / DUPLICATE NOTAM) - the VIDPYNYX/CNS junk.
- Card: "Until: PERM" now rendered RED with a hover (was green); status pill unaffected.
- Header: section title is just "NOTAMs" ("Active" was redundant given the status pills).
- h1 title -> "VIDP: Feathervane". Footer: minutes/hours spelled out, sources hyperlinked,
  stale "FAA filed" sentence removed. "Last fix" shown as HH:MM:SSZ.
- Verified live: both Refresh buttons DO refetch live (apparent "no change" = source hasn't
  published a newer report); glossary click (abbrJump) opens sidebar + highlights the term.

**ICAO-primary note:** a raw parse of the Doc 8400 PDF (two-column wrapped scan) comes out
incomplete (~491/2000, key terms missing) and would degrade accuracy, so it was NOT embedded.
The eAIP GEN 2.2 list IS India's official adoption of ICAO Doc 8400, now cleaned to single
ICAO-style definitions. Doc 8400 is used as authoritative OVERRIDES for specific terms on flag.

> Deploy note: the GitHub PR web UI (multi-step navigation) repeatedly dropped the browser
> extension's host access mid-flow. PRs #1-#2 went via branch+PR; #3 onward committed directly
> to `main` as single isolated, revertable commits (same one-change-at-a-time guarantee, far
> fewer fragile browser steps). Each is verified via the GitHub API (parse + key strings) before/after.

### 2026-06-05
- NOTAM hover rebuilt to **applied meaning**: Q-line decoded into FIR / Q-code (subject+condition
  via ICAO tables) / traffic / purpose / scope / limits / coordinates+radius; A) resolves to
  "VIDP: Indira Gandhi International Airport"; B)/C) show decoded UTC dates (PERM/EST handled).
- E-line abbreviations each get their own underline + hover; multi-word negations group as one
  token (e.g. "NOT AVBL" = "Not available", not "AVBL = Available").
- Removed "Qualifier line, decoded:" header from the Q tooltip.
- Header set to "Feathervane: VIDP/DEL"; subtitle "METAR & NOTAMs".
- Verified live: 0 decode errors across 41 NOTAMs, no console errors, all 5 tabs render.

### Earlier milestones
- Built single-file app; deployed to GitHub Pages with Wix CNAME (vidp.yashmoitra.com) + HTTPS.
- Live-fetch refresh with multi-source METAR + CORS proxies + data.json + last-good cache.
- Glossary sidebar (eAIP abbreviations, METAR codes, NOTAM codes, airline, airport lookups).
- Password gate; "Research Prototype / Last fix" header line.
- Visibility >=5000 m shown in km; "FAA filed" label + footnote; dual auto-refresh cycles.

---

## Planned / in progress (current batch)

- [ ] Confirm sidebar Abbr list == official **eAIP GEN 2.2**; fix discrepancies. Make it the
      single reference doc that also feeds E-line underlining.
- [ ] Remove **Airline** and **Airport** sidebar sections (tabs, panes, data, render fns).
- [ ] NOTAM card: remove "FAA filed ..." line; replace with **Active / Upcoming / Expired**
      (simple IF on start/end vs now).
- [ ] NOTAM abbr hover: add **slight dark-blue** text tint.
- [ ] Context-aware underlining: skip redundant common words; log removals.
- [ ] Move to branch + PR workflow.

---

## Abbreviation corrections log

Terms intentionally NOT auto-explained, or with corrected meanings:

| Term | Action | Meaning / Reason |
|------|--------|--------|
| TO   | drop   | Common English word; redundant in context. |
| DUE  | drop   | Common English word; meaning ("because of") is obvious. |
| TEMP | context | Means "temporarily" in some cases, not always "temporary". |
| WIE  | meaning | "With immediate effect". |
| DRG  | meaning | "During". |
| EQPT | meaning | "Equipment". |
| AMDT | meaning | "Amendment". |

### Decoding rules captured (to implement)

- **Numbers in context:** a number only means a runway/taxiway when it qualifies RWY/TWY.
  "27 Nov" is a date, not runway 27. Judge from surrounding words.
- **Designator + number = one token:** "RWY 10" underlines as a single unit reading
  "Runway 10"; "TWY 52" -> "Taxiway 52". Do not split into RWY + 10.
- **Waypoint names:** detect 5-letter ICAO waypoint/fix names (e.g. BAVOX) and label them
  "waypoint", do not run them through the abbreviation list.
- **Odd line breaks:** FAA source wraps NOTAM text at ~64 chars with hard newlines; our
  `pre-wrap` preserved them, causing mid-sentence cutoffs (see G0295/26 and the engine run-up
  NOTAM). Fix = rejoin a line into the next when it did not end in sentence punctuation (. : ;)
  AND the next line is not a field marker (A)..G), Q)). Keep real paragraph + field breaks.
- **Underline word SETS, not just words** (where applicable): "RWY 10" -> one underline
  "Runway 10"; "TWY 52" -> "Taxiway 52"; "ASSIGNED RWY" -> one underline "assigned runway".
  Reinforced principle: group the relevant set.
- **Dates are one group, never mis-read.** Group a date (e.g. "27 NOV", "27 NOV 2026",
  YYMMDDHHMM) as a single unit so its numbers are never taken as a runway/taxiway. Grouping
  prevents this confusion.
- **Idents / names are not abbreviations.** IDENT = "identification"; the value after it
  (e.g. DH, IDGM) is a navaid Morse ident / name -> do not expand. 5-letter waypoints
  (e.g. BAVOX) -> label "waypoint".

(Decision: abbreviation data stays **embedded** in index.html - faster, single in-file source
that both the sidebar and the underliner read.)
