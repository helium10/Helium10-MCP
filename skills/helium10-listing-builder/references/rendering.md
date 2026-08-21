# Rendering Formats

Exact output formats for the five mandatory renders. Read this when you're about to show the user tool output — these formats are how they audit your work, so abbreviating one defeats its point. The single exception is the Keyword Bank, which is capped for the opposite reason: at full length it is too big to audit. Every other render here goes out complete.

## Step 1 — the retrieved listing

Every field, no summarizing, no omitting:

```
Title(Item Name):          <product_name>
Item Highlight:            <item_highlight>
Bullet Points:             every bullet_points entry, each on its own numbered line
Description:               the complete description, not truncated
Images (N found):          every image url, numbered
```

If `item_highlight` / `bullet_points` / `description` is empty or null, **say so explicitly** — don't silently skip the line. A dropped line is indistinguishable from a field that had content.

**The image list is part of this render, not an optional extra** — number every URL (with its `variant` where present), because ask ① immediately below it invites the user to pick images *by number*. A run that renders the text fields but drops the images has broken ①: the user is being asked to choose from a list they were never shown. (On the single-field path no ① follows — there the image list may be omitted.)

## Step 2 — the analyzed attributes (ask ②)

All seven, one per labelled line, so the user can name the one they want changed instead of restating the set:

```
Product Name:     <product_name>
Brand Name:       <brand_name>
Category:         <category>
Description:      <description>
Key Features:     <key_features, each on its own line>
Brand Voice:      <brand_voice>
Target Audience:  <target_audience>
```

Any of the seven can come back empty. Show the label with the gap called out rather than dropping the line — an attribute the analysis failed to extract is the one most likely to need the user's correction, and a missing line reads as "not applicable".

Then invite edits by name — "correct any of these, or say go" — and put the `brand_asin` / `rufus_asins` questions after the list, phrased so silence means no. A user with nothing to change should be able to clear the whole checkpoint with one word.

If they do correct something, show just the changed fields back before ask ③:

```
Updated — Category: <new value> · Key Features: <new value>
```

Short on purpose. The point is to prove the edit landed as they meant it, not to re-run the audit; a second full table invites a second full review.

## Keyword Bank

One row per keyword, sorted by `priority_score` DESC:

```
# | Keyword | Search Volume | CPS | Priority Score | Sources
```

- Leave null cells blank — never render a literal `-`.
- Format `cps` as `X/10`, one decimal, trailing `.0` dropped: `4.8` → `4.8/10`, `4.0` → `4/10`.

**Cap the render at the top 30 rows.** Always show, on top of that: every allocated phrase even if one ranks below 30, and every phrase the user supplied themselves (those have no score, so they have no position in the sort — put them at the top of the table with the score cell blank).

The bank can run to a few hundred rows; a full-length table in the same message as the allocation table and an approval request is a wall, and a wall gets skimmed rather than audited. Thirty covers everything allocated plus the near-misses worth swapping in.

Say what you left out, on one line, using `total_count`:

```
Showing top 30 of 118 keywords by priority score — ask for the full list any time.
```

Mean that offer. The cap is a readability default, not a gate — if they ask for the long tail, print all of it. What they do next decides the approval: browsing it changes nothing, but pulling a phrase into a slot is an edit, so re-show the allocation table and get the approval against the new version. A request to see more is never itself an approval.

This is the one truncation the flow permits. It is not licence to abbreviate anything else: the Step 1 catalog data, the attributes, the allocation table and the generated listing all render in full.

## Slot allocation

`Slot | Theme | Keywords`, all slots, never a summary. **Theme is what makes the table auditable**: each bullet's Theme cell names the one product feature that bullet sells (derived from its theme keyword — the highest-priority phrase in the slot); TITLE and DESCRIPTION have no theme, leave their cell blank. Without this column, a correct theme-grouped allocation and a lazy rank-order chunking look identical, and the user approves a shape they can't see.

```
| Slot        | Theme                    | Keywords                                        |
|-------------|--------------------------|-------------------------------------------------|
| TITLE       |                          | laptop stand for bed · bed desk for laptop · …  |
| BULLET 1    | folds flat for storage   | folding lap desk · foldable bed table · …       |
| BULLET 2    | dorm & couch portability | lap desk for dorm · lap desk for couch · …      |
```

If two bullets end up with interchangeable themes, that is the allocation telling you it degenerated into chunking — regroup before showing the table, don't relabel.

Re-show the entire table after every edit the user makes — a diff of just the changed slot loses the shape they're approving.

This table plus ask ④ **is** the binding approval. Don't show a "preview" now and a second table later; that's a wasted round trip.

## Generated or rewritten listing

- Bold each allocated keyword where it appears in its slot — case-insensitive, multi-word matched as a contiguous span.
- Under each slot, a `Missing keywords` line listing allocated keywords that did not appear, so the user can target them with a rewrite.
- Bolding is display-only. Pass unbolded text to any rewrite call.
- Show `item_highlight` beside the title, never as a bullet.
- Present the returned `title` as finished, not as a shortfall.

For a title rewrite, show the new `item_highlight` alongside the new title — they were regenerated together.
