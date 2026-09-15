# Part Coding status model — working notes

Context for the changes made to `rk_part_coding.html` across passes, plus
any open questions still needing a product decision.

## What's been built

1. **Status pill contrast** — the selected state on the summary pills
   (`.cnt[aria-pressed="true"]`) was easy to miss (1px border only). It now
   gets a doubled border/ring effect plus bolder text so the active filter
   is obvious at a glance.

2. **Out of Stock status** — its own bucket (`Auto`, `Manual`, `Review`,
   `Out of Stock`, `RFQ`), rather than overloading `Manual` + a free-text
   reason. Any match with 0 on hand is flagged `Out of Stock`, **regardless
   of match confidence** — a fuzzy/fallback match that would otherwise land
   in `Review` still flips to `Out of Stock` if the top hit has 0 on hand.
   Applies both to the initial AI match (`buildRow`) and to a person
   explicitly picking a 0-stock candidate from the popover.

3. **Out of Stock is sticky, and the reason always records the audit
   trail.** Once a line has ever been flagged Out of Stock, the status
   *stays* Out of Stock even after it's resolved to an in-stock alternate —
   that's the point of giving it its own bucket: Jeff can filter "this
   shipped short at some point" without it disappearing into `Manual` or
   the reason being buried in a text field. The Reason field automatically
   records what happened (`Out of stock (oldCode) — swapped to newCode.`)
   regardless of whether the ISR typed anything, appending to whatever
   custom note they wrote rather than overwriting it.

4. **Fixed a data bug**: rows with `status: "Manual"` had a placeholder
   `conf: 5` left over from copy/pasting the RFQ shape, which didn't match
   the actual candidate match score shown in the Reason popover (e.g.
   `44870` showed 5% in the table but 62% in the popover). Corrected all
   existing Manual rows to use their real match score (44870 → 62%,
   35915 → 67%, 38644 → 44%).

5. **Status transition on confirm**: confirming a candidate in the picker
   sets the row to `Manual` (a person made/overrode the call) — never
   `Auto`, which only ever comes from the AI itself with nobody touching
   it. `Out of Stock` is the one exception (see #3). This also meant fixing
   two `CODEBOOK` seed entries (44870, 35915) that were hardcoded to
   `status: "Manual"` — they now start `Review` like any other AI match, so
   clicking **Code It** never produces a `Manual` row on its own. The two
   `Manual` rows still seeded in `adRows` are fine as-is — that's the Admin
   queue's pre-existing history, not live Code It output.

6. **Table filtering** — Part Coding and Admin tables now have the same
   free-text search pattern as the Rule Engine's `fSearch`: `pcSearch` /
   `adSearch` inputs filtering on description, part code, RK description,
   source, and reason. Pill counts are computed against the
   search-filtered set first, so counts stay meaningful while typing. CSV
   export and "Clear filters" respect/reset the search box too.

7. **Column sorting** — Part Coding and Admin tables have the same
   click-to-sort column headers (with carets) as the Rule Engine table.
   Click a header to sort ascending, click again for descending;
   `Mark`/`Quantity`/`RK Inv`/`Conf` sort numerically (nulls sort lowest),
   everything else sorts as text. `Reason`/`Approval` stay unsorted since
   they're icon-only columns.

8. **Confidence % is color-coded** — the Conf column text is now green /
   amber / red using the same 90%-and-up / 70–89% / under-70% thresholds
   already used for candidate match badges in the popover (`mpClass`), so
   match quality reads at a glance without adding another pill. Applies to
   every row's Conf value, including `Out of Stock` and `Manual` rows —
   the color reflects match quality, which is a separate concern from the
   inventory problem the status itself flags.

9. **Export is what moves a line to Jeff's queue — a move, not a copy.**
   Clicking Export on the Part Coding screen downloads the CSV (unchanged)
   *and* pushes the same rows into the Admin queue (`adRows`), tagged with
   whichever customer is selected in the scope dropdown. Every status goes
   — Review, RFQ, and Out of Stock included, not just Auto/Manual; nothing
   is held back waiting for the batch to be "fully reviewed" first, since
   everything ultimately shows up on Jeff's screen. The exported rows are
   then **removed from the Part Coding table** — they only ever exist in
   one place at a time, so there's no double-counting and no way to
   re-export the same line twice. Export respects whatever's currently
   filtered/searched, so exporting a subset leaves the rest in Part Coding
   for further work.

## Status model

- **Auto** (green) — AI is confident, no human involved yet. Never a
  starting point for anything a person has touched.
- **Review** (amber) — AI has a candidate but confidence is mid-range;
  needs a person to confirm or pick an alternate.
- **RFQ** (red) — no candidate cleared the bar at all; needs sourcing, not
  just a part-code pick.
- **Out of Stock** (orange) — the top match has 0 on hand, regardless of
  how confident that match is. Distinct from Review/RFQ because the
  *problem* isn't match quality, it's inventory. Sticky once flagged (see
  #3 above) — confidence thresholds don't apply to it since an Out of
  Stock line is never "final" as-is; it always gets resolved to something
  else before it would ship.
- **Manual** (blue) — a person has confirmed or overridden a pick. Never
  an AI-native starting status; it only exists after a human acts on a
  Review/RFQ/Auto row via the candidate picker. Confidence thresholds
  (90/70) are kept as-is — hand-authored per row in this demo data, not
  derived from a formal rule yet (see Roadmap below).

Nothing in the matcher (`buildRow`, `fallbackMatch`, or the `CODEBOOK` seed
data) assigns `Manual` as a starting status — clicking **Code It** only ever
produces `Auto`, `Review`, `RFQ`, or `Out of Stock`. The two `Manual` rows
still in `adRows` are the Admin queue's pre-existing history (lines already
resolved before this session), not something Code It generates.

**Reject is terminal, by design.** When Jeff rejects a line on the Admin
screen, it does not get kicked back to the ISR for another pass — it's a
record-keeping state. Rejected (and approved) match decisions build up a
history that the first screen's matcher searches over, so the AI gets
better with more examples over time rather than looping a line back and
forth between screens.

## Known limitations / minor gaps (not blocking, just noted)

- **`MBA Scope` isn't one of the Admin customer filter's hardcoded
  options** (`Customer: All / Diamondback Energy / XTO Energy / Ovintiv`).
  Lines exported while `pcCustomer` is set to "MBA Scope" still show up
  fine under "All" and are searchable by text, just not selectable via
  that dropdown. Making the Admin customer filter populate dynamically
  from whatever's actually in `adRows` (like the Rule Engine's
  `renderSelect` pattern) would close this gap if it matters.
