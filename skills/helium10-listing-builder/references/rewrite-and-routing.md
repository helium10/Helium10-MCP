# Rewriting a Section, and Where Leftover Keywords Go

Read this when the user wants a specific field changed, or when keywords are left over after a generation and need a home.

## Where leftover keywords go

Their instinct is right — unplaced high-volume keywords do deserve a home. Find the field with room; don't force them into one just because it sits next to the title. Check in this order and take the first that actually **fits**: `item_highlight`, then bullets, then description. Bullets outrank the description for higher-priority keywords, and reopening an approved bullet is a normal rewrite, not a regression.

**Expect `item_highlight` to be full.** A well-generated listing spends the whole 200-character title-pair budget, so the highlight routinely comes back at 120-125. Check the number rather than assuming it's the roomy option — in practice **bullets are the realistic first stop.**

**Judge room by whether the phrases actually fit, not by distance from the cap.** A bullet's usable space is the `max_bullet_len` you pass (reuse the value the listing was generated with — 250 unless it was overridden; the hard cap is 700), not its current text length. Raising it on the rewrite call to create room is your call to make and mention — raise it only as far as the phrases need, since it also changes the rhythm of copy the user already approved.

**If a cut is genuinely unavoidable** — nothing fits anywhere, or the user pinned you to one full field — then which phrases get sacrificed is a must-ask. Show the text, say it's at the cap, let them choose: you can measure the budget, you cannot know which claim sells their product. But a near-cap field is not by itself a reason to interrupt — if a later field has room, keep walking the list. Asking someone to sacrifice copy when nothing needs sacrificing is a checkpoint used as a preference poll.

## More keywords than a slot accepts

`target_keywords` accepts **exactly 3** phrases — not a maximum, an exact count. So four phrases is already an overflow situation, and the channel depends on the target:

- **`description`** — overflow goes in `optional_keywords` (that parameter applies here only). Which phrases go where is a must-ask.
- **`bullet` / `title`** — **no optional channel exists.** Overflow needs a second rewrite call, which must be padded with Keyword Bank phrases the user never named.

That padding is why the split is a must-ask — not because anything gets cut, but because you'd be weaving unchosen phrases into approved copy. Ask which of their phrases go first; let them pick the padding or explicitly delegate it. Never select it silently.

## Calling `rewrite_listing_section`

Requires a complete baseline first. Two places it can come from: the `generate_listing` / rewrite responses earlier in this conversation, or — on the single-field path — the live copy `retrieve_listing_by_asin` just fetched (its `product_name` field is the `title` baseline — same content, different field name). **Never re-run `generate_listing` just to recover a lost baseline**: it is non-idempotent, so you would replace copy the user approved with copy nobody has seen. Pass the **full** baseline as context even when rewriting one slot: `title`, all `bullets`, `description` — whatever the target is. `item_highlight` is the exception: it is only accepted for a `title` rewrite (see below), and on a `bullet` or `description` rewrite the value is silently dropped rather than rejected, so sending it there buys nothing.

- Draw `target_keywords` from the Keyword Bank, excluding everything already allocated in the baseline.
- **When `target` is `title`, pass the current `item_highlight` too.** Both are rewritten in the same AI call. Omitting it is not an error — the value is silently dropped and you lose the paired rewrite. This also means `target: title` is the *only* way to re-tighten the highlight; the call itself is legitimate, and what's forbidden is the value (a title over 75) and the silent deletion of approved claims — not the call shape.
- `optional_keywords` and `rufus_asins` apply to `description` only; put the **entire remaining Keyword Bank** in `optional_keywords` there, where unlike `target_keywords` they are weave-if-it-fits. `max_bullet_len` / `min_bullet_len` apply to `bullet` only.
- Before rewriting a bullet, make sure you hold its **exact** text — `rewrite_content` must match a `bullets` entry verbatim. Keep the source responses verbatim — a paraphrase or length estimate cannot be recovered into the original string; on the single-field path the verbatim source is the Step 1 render itself.

**On the single-field path, the 3 `target_keywords` pass the user once before the call** — render the bank (top-30 rule) with your proposed three marked, and a one-word go clears it. They are about to be woven into copy the user already likes; that is the ④-equivalent this path keeps.

**One call rewrites one bullet** — `rewrite_content` is a single string, so "tighten bullets 2 and 4" is two calls, and the `bullets` baseline you pass the second call must already contain the first call's result.

Each call returns a new `request_id` — keep it internal (pass it to a later sync or as `parent_request_id`; never surface it). Re-render the rewritten section with the same bolding + `Missing keywords` rules as a fresh generation; for a title rewrite show the new `item_highlight` alongside.

## Keyword relevance is the user's fact, not yours

A phrase coming out of the Keyword Bank means it has search volume — not that it describes this product. `bed rail with storage pocket` is a feature claim; if the product has no pocket, weaving it in is a false listing claim with real Amazon consequences, and high volume makes that worse rather than better.

When a leftover phrase asserts a feature, compatibility, or certification you haven't seen evidence for in the analyzed attributes or the existing copy, ask before placing it. Only the user knows what they actually ship.
