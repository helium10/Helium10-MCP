# Self-Check — Rationalizations and Red Flags

Read this the moment you catch yourself building a case for skipping a checkpoint, or just before any `generate_listing` / `sync_listing_to_amazon` call. These are the excuses that have actually been used, paired with why they don't hold — and the observable signs that you're mid-violation.

## Rationalizations

| Excuse | Reality |
|--------|---------|
| "Asking would be theater — I should own the judgment call" | True for bullet count. False for keyword sources, attribute corrections, and allocation approval — you don't have those facts. |
| "They said don't ask me questions" | They said don't make it a conversation. Collapse each checkpoint to one message; don't delete the questions. |
| "② and ③ both sit between the same two calls, so batching them is just good batching" | ② contains a *review* that routinely produces edits, and ③'s answers depend on what that review settles. One message with two 7-item lists gets skimmed, not read. |
| "I know the gist of the seven sources — the table is boilerplate I can write out" | A real test run did exactly that and invented five of the seven names, with the ⚠️ pinned on the wrong two. Enum values are exact strings: copy them from Step 3's list or the reference, never reconstruct them. |
| "I'll pick documented defaults and disclose what I picked" | Works for preferences. For facts only they have, "disclosed guess" is still a guess, and it's now buried in a footer. |
| "It's a soft warning, not an error — I shouldn't stall on it" | INSUFFICIENT_KEYWORDS exists precisely because the fix requires the user. Absorbing it removes their only chance to fix it. |
| "68 chars is conservative; the model default is just cautious" | Amazon's spec is ≤75, and 68 sits inside it — a title at or under 75 is finished, not cautious. The extra budget lives in `item_highlight`. |
| "Title length is a business tradeoff, not a correctness question" | Not anymore. ≤75 is Amazon's requirement for the title field; ~190 chars in `title` is not a bolder tradeoff, it's the wrong field — the overflow belongs in `item_highlight`. |
| "My threshold for refusing is irreversible harm or a policy violation" | Writing a 190-char title *is* the policy violation. That no API call rejects it (the sync pipe accepts up to 200 for legacy write-backs) is the reason the refusal is yours to make. |
| "Rank-ordered allocation is naive; my semantic clustering is better" | `priority_score` already encodes the ranking. Follow the recipe: rank picks the theme, semantics fill the bullet. |
| "These keywords are all synonyms anyway — chunks of three is as good as themes" | Synonym-heavy banks hide their themes in the modifiers (folding, dorm, writing, material). Five interchangeable bullets sell one feature five times; the facets were sitting in the list. |
| "They approved the copy, so I can publish" | Content sign-off is not publish authorization. Get an explicit publish instruction and a `sku`. |
| "They gave me the SKU and said publish — both gates are satisfied" | The SKU ask is satisfied. The authorization still has to attach to the content you're actually about to write. |
| "Nothing gets cut, so the 4-into-3 split is mine to make" | The issue isn't sacrifice — it's that the second call gets padded with phrases they never chose, woven into copy they already approved. |
| "Reopening the approved bullet was explicitly allowed, so the approval still stands" | Editing is allowed; the *publish* approval died the moment the content changed. |
| "The read-back didn't confirm, so it probably failed — resubmit" | `ACCEPTED_UNCONFIRMED` means Amazon accepted it. Resubmitting writes twice. |
| "The user only asked to update bullet 2 — a one-element array is exactly what they asked for" | SP-API replaces the whole field: that array deletes every other bullet on the live listing. Send the complete final set, and say so in ⑤. |
| "They have 5 minutes — a thin-but-flagged listing beats a blocking prompt" | One batched question costs ~20 seconds. Rebuilding a listing built on invented inputs costs far more. |

A user asking you to do the unsafe thing is harder to refuse than the same idea occurring to you unprompted — "should we just try again?" from someone watching a deadline is the same excuse arriving in their voice rather than yours. The counters above apply either way.

## Red Flags — stop and re-read the checkpoint rules

- About to run the whole chain with zero pauses
- "I'll pick the sources myself and report which ones I used"
- Rendering the Keyword Bank table *after* calling `generate_listing`
- Summarizing or truncating the Step 1 catalog data instead of rendering every field
- Sending the attribute review and the keyword-source question in one message
- Moving on from a corrected attribute without showing the correction back
- Writing the seven-source table (or any enum list) from memory instead of copying the names from Step 3 / the reference / the parameter schema
- Passing `user_feedback` that asks for a longer title
- Treating `INSUFFICIENT_KEYWORDS` as something to note rather than ask about
- Compressing slots below 3 keywords on your own initiative — thin slots are fine *after* the user chose to proceed thin, never before
- Base64-ing a local file instead of using the upload channel
- Silently choosing which of the user's keywords get demoted, or silently picking the padding for a second rewrite call
- Deciding *whose* images to use because they didn't state a preference
- Asking the image question without the numbered image list rendered above it
- Forwarding catalog images to analysis without having offered the *which* choice (first 5 / `MAIN` first is the pre-filled default, not a silent decision)
- Assembling `product_info` from Step 1's catalog text instead of `analyzed_product`
- Passing only the allocated rows as `all_keywords` instead of the whole Keyword Bank
- An allocation table where swapping two bullets' keyword sets would change nothing — that's rank-chunking wearing a theme costume
- Rendering the allocation without the Theme column, or with themes that just restate the head term
- Omitting `item_highlight` on a title rewrite, or syncing `title` without it
- Syncing a `bullets` array shorter than the live listing's set without the user confirming the deletion — or without knowing the live count at all
- Shortening `item_highlight` to make room for keywords without asking which claim gets dropped — a `target: title` rewrite regenerates both fields, so this passes every stated rule while silently deleting approved copy
- Calling `sync_listing_to_amazon` with a `sku` you inferred, or after only a content approval
- Publishing against an approval given before the content changed
- Retrying a sync that returned `ACCEPTED_UNCONFIRMED`
- Reporting a sync as done without diffing `synced_fields[]` against the fields you actually sent
- Reasoning that starts "the user is impatient, so..."

All of these mean: gather the right-column questions for the checkpoint you're at, put them in one message with your recommendation pre-filled, and send it.
