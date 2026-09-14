# Part Coding status model — working notes

Context for the changes made to `rk_part_coding.html` in this pass, plus open
questions that need a product decision before the next iteration.

## What changed this pass

1. **Status pill contrast** — the selected state on the summary pills
   (`.cnt[aria-pressed="true"]`) was easy to miss (1px border only). It now
   gets a doubled border/ring effect plus bolder text so the active filter
   is obvious at a glance.

2. **Out of Stock status** — added as its own bucket (`Auto`, `Manual`,
   `Review`, `Out of Stock`, `RFQ`), rather than overloading `Manual` +
   a free-text reason. Seeded one example end to end:
   - Part Coding screen: `6" 150# SS Blind Flange` matches part `36194` at
     96% confidence, but on-hand is 0 → status `Out of Stock`, inventory
     shown as **0 in red** so it's impossible to miss.
   - Opening the candidate picker on that row pre-fills the reason box with
     "Out of stock" and highlights any zero-inventory candidate in red.
   - Confirming a swap to an in-stock alternate (e.g. `36194D`) **keeps the
     status as `Out of Stock`** (does not collapse into `Manual`) — that's
     the whole point of giving it its own bucket: Jeff can filter/see
     "these were out of stock and got swapped" without reading every reason
     field. Source flips to `Manual` since a person made the call; the
     reason stays editable.
   - Added a matching resolved example to the Admin queue (XTO Energy,
     mark 7) so this shows up in Jeff's approval screen already actioned.

3. **Fixed a data bug you caught**: rows with `status: "Manual"` had a
   placeholder `conf: 5` left over from copy/pasting the RFQ shape, which
   didn't match the actual candidate match score shown in the Reason
   popover (e.g. `44870` showed 5% in the table but 62% in the popover).
   Corrected all existing Manual rows to use their real match score
   (44870 → 62%, 35915 → 67%, 38644 → 44%).

4. **Status transition on confirm**: previously, confirming a candidate in
   the picker always set the row to `Auto`. That's backwards — `Auto` should
   mean "the AI was confident enough, no human touched it." Now, confirming
   a pick sets the status to `Manual` (a person made/overrode the call),
   except for `Out of Stock` rows, which stay `Out of Stock` per #2 above.

5. **Table filtering** — Part Coding and Admin tables only had the status
   pills to filter by; the Rule Engine screen also has a free-text search
   box (`fSearch`) that matches across several fields. Added the same
   pattern: `pcSearch` / `adSearch` inputs, filtering on description, part
   code, RK description, source, and reason. Pill counts are computed
   against the search-filtered set first (same order of operations as the
   Rule Engine's `baseFiltered()`), so counts stay meaningful while typing.
   CSV export and "Clear filters" now respect/reset the search box too.

6. **Column sorting** — Part Coding and Admin tables now have the same
   click-to-sort column headers (with carets) as the Rule Engine table.
   Click a header to sort ascending, click again to flip to descending;
   `Mark`/`Quantity`/`RK Inv`/`Conf` sort numerically (nulls sort lowest),
   everything else sorts as text. `Reason`/`Approval` stay unsorted since
   they're icon-only columns, same as the Rule Engine's action column.

## Status model, as clarified in this pass

- **Auto** (green) — AI is confident, no human involved yet.
- **Review** (amber) — AI has a candidate but confidence is mid-range;
  needs a person to confirm or pick an alternate.
- **RFQ** (red) — no candidate cleared the bar at all; needs sourcing, not
  just a part-code pick.
- **Out of Stock** (orange) — AI found a confident match, but the top hit
  has 0 on hand. Distinct from Review/RFQ because the *problem* isn't match
  quality, it's inventory — flagged so the ISR catches it before a quote
  goes out short.
- **Manual** (blue) — a person has confirmed or overridden a pick. This is
  never an AI-native starting status; it only exists after a human acts on
  a Review/RFQ/Auto row via the candidate picker.

Nothing in the matcher (`buildRow`, `fallbackMatch`) assigns `Manual` as a
starting status anymore in spirit — the two remaining `Manual`-seeded rows
in `CODEBOOK`/`adRows` are intentionally there to demonstrate "what an
already-reviewed line looks like," not to imply the AI can emit `Manual`
on its own.

**Reject is terminal, by design.** When Jeff rejects a line on the Admin
screen, it does not get kicked back to the ISR for another pass — it's a
record-keeping state. The point is that rejected (and approved) match
decisions build up a history that the first screen's matcher searches over,
so the AI gets better with more examples over time rather than looping a
line back and forth between screens.

## Open questions / roadmap (not implemented — need a decision first)

1. **Confidence thresholds.** What % cutoffs actually separate Auto /
   Review / RFQ? Right now every row's status + conf% is hand-authored in
   the demo data, not derived from a rule (e.g. "≥90 = Auto, 60–89 =
   Review, <60 = RFQ"). Once real matching is wired up, we need agreed
   thresholds — and to decide whether they're global or vary by customer
   scope (the `pcScope` selector — Generic / Customer Specific / Strict —
   No Substitutes — hints that they might).

2. **Where does "Out of Stock" trigger from?** Today it's only wired for a
   confident (Auto-level) match with 0 on hand. Should a *Review*-level
   match that also happens to have 0 inventory get flagged Out of Stock
   too, or does it just stay Review (since it already needs a human look)?

3. **Does Out of Stock ever fall back to Auto?** If the ISR swaps to an
   alternate that itself is a 100% match with plenty of stock, should the
   line ever "graduate" back to Auto, or does Out of Stock permanently mark
   the line for audit purposes (so Jeff always knows this shipped short on
   the first pass)? Currently implemented as: always stays Out of Stock
   once flagged, forever.

4. **"Send to Admin" / mark-as-reviewed action.** There's currently no way
   on the Part Coding screen to say "this batch is done, send it to Jeff's
   queue." The Part Coding table (`pcRows`) and the Admin queue (`adRows`)
   are separate, unconnected datasets in this wireframe. Real questions:
   - Is submission per-line or per-batch?
   - Does a batch need every line to be Auto/Manual (i.e., nothing left in
     Review/RFQ/Out of Stock) before it can be submitted, or can it go
     partially, with the unresolved lines held back?
   - Should Admin only ever show Auto/Manual lines (fully resolved), and
     hide Review/RFQ/Out of Stock until an ISR clears them? That's the
     behavior implied by "Jeff should only see Auto and Manual once
     everything's reviewed" — but the current Admin seed data intentionally
     includes Review/RFQ rows too, so this needs a decision either way.
