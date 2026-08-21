---
name: helium10-listing-builder
description: Use when a user wants to write, optimize, create, build, improve, rewrite, or publish an Amazon listing through the Helium 10 Listing Builder MCP tools - a whole listing or just one field of it (the title, a single bullet, the description). Triggers on this intent in any language or phrasing, e.g. "optimize my listing", "improve my title / bullets", "rewrite my description", "write a listing for this ASIN", "write a listing for my new product", "push it to Amazon". Also use when the user pushes back on the flow — telling you to skip confirmations, run it all in one go, stop asking questions, pick the keyword sources yourself, lengthen a generated title toward 200 characters, or publish right now.
---

# Listing Builder Workflow

## Overview

The Listing Builder tools expose atomic capabilities. Chaining them is your job, and the chain has mandatory user checkpoints that are not optional politeness — the tools cannot supply what those checkpoints collect.

**Core principle: a checkpoint exists because the user holds information you cannot derive. It is not a preference poll, and impatience does not supply the missing information.**

**The one fact that decides the most arguments:** Amazon split the legacy ≤200-character title into two fields — `title` ≤75 and `item_highlight` ≤125. The 200-character budget still exists; it now lives across the pair. **Any title at or under 75 characters is complete** — 68, 70, 75, all finished. Never treat one as short.

**Scope.** Orchestration only — what to call in what order, what to ask, what to render. Per-parameter limits, formats and required flags live in the tool schemas and are enforced there; a wrong length, count or enum is rejected before it reaches the backend with an error naming the fix. Everything below is a rule **no single call can validate**, because each call looks valid while the chain is wrong.

**Reference files** — read the relevant one at the point of use rather than up front:

| File | Read it when |
|---|---|
| `references/keyword-sources.md` | Composing the source question in ask ③, or a source came back with zero keywords |
| `references/rendering.md` | About to render one of the five mandatory outputs — Step 1 catalog data, the analyzed attributes, the Keyword Bank, the allocation table, or a generated/rewritten listing. Sync results aren't covered there; see `sync-outcomes.md` |
| `references/rewrite-and-routing.md` | Placing leftover keywords, or rewriting one field of an existing listing |
| `references/sync-outcomes.md` | Right after `sync_listing_to_amazon` returns |
| `references/self-check.md` | **You feel the pull to skip a checkpoint** — and before any `generate_listing` or `sync_listing_to_amazon` call |

## When to Use

Trigger on any of these, in any language or phrasing:

| User intent | Entry point |
|---|---|
| "optimize my listing" / "improve my title / bullets" / "rewrite my description" | Path B (Optimize) — start at `retrieve_listing_by_asin` |
| "write a listing for this product" (no ASIN yet) | Path A (New) — start at image collection |
| "the title isn't compelling, redo it" — a full listing already exists in-session | Jump to `rewrite_listing_section` |
| "just fix one bullet on this ASIN" — improve a single field of a live listing, no full rewrite wanted | Single-field path: Step 1 for the baseline → ③ + `find_keywords_with_multi_source` → render the bank (top-30 rule) with your proposed exactly-3 `target_keywords` and get one OK → `rewrite_listing_section` (one bullet per call) |
| "I've done my own keyword research — just generate it" — user brings keywords / product text, wants to skip straight to generation | Still Path A/B: `product_info` must come from image analysis and the bank from `find_keywords_with_multi_source` (their phrases join as user-supplied additions at ④) — there is no generate-direct entry |
| "looks good, publish it" — content already reviewed **and** publishing explicitly requested | `sync_listing_to_amazon` |

**Checkpoint scoping:** the blanket checkpoint rules below describe the full generation chain. On the single-field path only ③ and ⑤ fire — ①, ② and ④ each gate a tool (`analyze_product_images`, `generate_listing`) that this path never calls, so there is nothing for them to gate. The ④-equivalent on this path is the target-keyword OK described in the row above.

**Do not use** for reading/scoring an existing listing without writing copy — that's Listing Analyzer (`get_listing_details`, `get_listing_score`, `compare_listings`, `get_competitor_overview`, `get_top_keywords`). Those tools never author copy; all authoring goes through this flow.

## Cross-Tool State

The values that must stay consistent *across* calls. Nothing validates these, because each call on its own is well-formed.

| State | Rule |
|---|---|
| `marketplace` | A must-ask, and it has to be settled **before the first tool call** — which on Path B is `retrieve_listing_by_asin`, i.e. earlier than ①. That costs no round trip: you need the ASIN from them anyway, so confirm the marketplace in the same opening message, pre-filled so a bare "yes" clears both. Then pass the **same value to every tool** and **never ask again**. Output language is derived server-side from it — a wrong value silently produces a listing in the wrong language, and only `sync_listing_to_amazon` will ever complain. Take the code verbatim from the parameter's enum (`US`, `UK`, `DE`, ...); never invent a format. If the user volunteers it ("amazon.com", "US 站"), state the code you mapped it to instead of re-asking. |
| `asin` | Once established, reuse it. `sync_listing_to_amazon` defaults to it — don't re-ask unless the user says they want a different ASIN. |
| `product_info` | Must be the `analyzed_product` object from `analyze_product_images`, with the user's corrections merged in. A hand-built object passes schema validation and is still wrong. |
| Keyword Bank | Must come from `find_keywords_with_multi_source`. Keywords from other tools are structurally valid but lack `priority_score`, which the whole allocation depends on. |
| `request_id` | Returned by `analyze_product_images`, `generate_listing`, `rewrite_listing_section`. **Internal plumbing only — never show it to the user** (it is not memorable and buys them nothing). Pass the upstream one as `parent_request_id`, and pass Step 4/5's to `sync_listing_to_amazon` for validation. |

**Two languages are in play, and they are different settings.** You speak to the user in whatever language they wrote in; the listing copy comes back in the *marketplace's* language, derived server-side. A Chinese-speaking seller on `amazon.com` gets a Chinese conversation and an English listing — never "fix" the copy's language to match the chat, and never pick the marketplace to match the user's language.

## The Flow

```
Path A (new product, no ASIN)        Path B (optimize existing ASIN)
        │                                       │
        │                           retrieve_listing_by_asin
        │                                       │
        │                           ▼ RENDER every field (see Step 1)
        │                                       │
        └───────────────┬───────────────────────┘
                        │
   ① ASK: images — these catalog ones (if so, WHICH; default first 5, MAIN first)
        or their own (public URLs or local upload)
                        │
             ┌──────────┴──────────┐
        URLs │                     │ local files
             │      request_image_upload_url → upload → image_refs
             └──────────┬──────────┘  (channels mix — ≤5 combined)
                        │
               analyze_product_images
                        │
               ▼ RENDER the 7 analyzed attributes
                        │
   ② ASK: corrections to the 7 attributes · brand_asin · rufus_asins
        ↺ if they correct anything, re-show the corrected set before moving on
                        │
   ③ ASK: keyword sources (all 7, flag the 2 gated ones) · competitor ASINs
                        │
             find_keywords_with_multi_source
                        │
               ▼ RENDER Keyword Bank + slot allocation
                        │
   ④ ASK: approve the allocation — one message, every slot in it
        ⚠ if warning INSUFFICIENT_KEYWORDS → fold it into this same message
                        │
                 generate_listing
                        │
   ▼ RENDER listing: bolding + Missing keywords
                        │
             ┌──────────┴──────────┐
             │  rewrite_listing_section   (loop as needed)
             └──────────┬──────────┘
                        │
   ⑤ ASK: explicit publish authorization against the FINAL text · sku
                        │
             sync_listing_to_amazon ⚠️ DESTRUCTIVE
```

`①②③④⑤` = the checkpoints on the generation chain. `▼` = mandatory render. There are five render *artifacts* across four `▼` moments — the Keyword Bank and the allocation table share one, deliberately, because you can't judge the allocation without the bank in front of you. A rewritten section gets re-rendered the same way the generated listing does; it has no marker only because the rewrite loop can run any number of times.

They are not the complete list of times you wait. Two branches add their own, and neither is optional when it applies:

- **The local-file upload path inside ①.** How many files and which format are facts you must be *sure* of — but derive them when you can rather than asking; see Step 2 for when each applies. Collecting the `image_ref` values back from the user is a wait only when you can't run the upload yourself.
- **The rewrite phase between the listing and ⑤** — see Step 5.

Don't read the five checkpoint markers as the full set, or as permission to fold these into ⑤ or skip them.

Publishing is always the last thing that happens. A rewrite after ⑤ isn't a later stage — it means the content changed, so ⑤ has to happen again against the new text.

**Four asks is the minimum on the generation chain** (the single-field path is a shorter chain — see Checkpoint scoping under When to Use) — don't collapse them to zero, and don't split any one of them into several sequential questions. Branch asks (below) are additional, not a violation of this. ⑤ is separate because publishing is optional and must attach to final content.

"Don't split" means don't send in three messages what one message could have carried. It does not mean one round trip per checkpoint: when the user's reply *changes* something, confirming the change is a new message about new information, not the same question re-asked. ② and ④ both work that way — a corrected attribute and an edited slot each get shown back.

**② and ③ are separate checkpoints, and merging them is the most tempting mistake here.** Both sit between `analyze_product_images` and `find_keywords_with_multi_source`, so folding them into one message looks like exactly the batching the next section asks for. It isn't:

- **② contains a review, not just questions.** The user has to read seven generated attributes and decide whether each is right. That is the most cognitively expensive thing you will ever ask them, and it routinely produces edits — which means ② can legitimately take two round trips (correct → re-show → confirm). Bolting four more questions onto a message that may need re-sending wastes them.
- **③ is unanswerable until ② settles.** Which competitors to reverse-look-up depends on what the product actually turned out to be. If the user corrects `category` or `key_features` in ②, the competitor set they'd have named beforehand may be the wrong one.
- **A single message carrying two 7-item lists is not one checkpoint.** Seven attributes to audit *plus* seven keyword sources to choose from reads as a wall, and the usual response is to skim both.

Within each of them, batch fully: ② is one message, ③ is one message. `brand_asin` / `rufus_asins` belong in ② rather than ③ because they're the same subject the user is already looking at — you just showed them `brand_voice` and `target_audience`, and whether they want a competitor's tone mirrored may depend on what the analysis found. Asking either checkpoint *before* `analyze_product_images` runs is wrong: nothing is on screen yet for the user to react to.

## The Two Kinds of Questions

This distinction governs every checkpoint above. Get it right and everything else follows.

| | You may default and disclose | You must ask |
|---|---|---|
| **What it is** | Judgment calls you're better equipped to make | Facts only the user has |
| **Examples** | bullet count, bullet length, whether to run a rewrite pass | `marketplace` (confirm once, before the first call); which keyword **sources**; *whose* images and, for catalog images, *which* ones (pre-fill first 5, `MAIN` first, as the default); corrections to analyzed attributes; `brand_asin` / `rufus_asins`; **which competitor ASINs to reverse-look-up**; **slot allocation approval**; `sku`; publish authorization |
| **If you guess** | Slightly suboptimal output | Wrong output built on invented facts, and the user can't tell |

"Asking is theater" is true for the left column. It is false for the right column — those aren't preferences, they're inputs you don't have.

**When the user says "don't ask me questions":** collapse each checkpoint into **one message per checkpoint** — not one message for the whole flow — pre-fill your recommendation so a bare "go" is a valid reply, and proceed. That satisfies the actual complaint (round trips) without inventing answers. Never convert a right-column question into a default.

Batch **within** a checkpoint, never **across** them: ② cannot be sent before the analysis renders, ③ cannot be sent until ② settles, and ④ cannot be sent before the Keyword Bank exists. Likewise "approval covering every slot" means one message carrying the full table, not a question per slot.

## Step 1 — `retrieve_listing_by_asin` (Path B only)

**Render every field — no summarizing, no omitting**, and say so explicitly when a field comes back empty rather than dropping the line: a missing line is indistinguishable from a field that had content. → **`references/rendering.md`** for the exact layout.

Then send ask ① — one message, two halves, and it only makes sense **under the numbered image list you just rendered** (never ask it with the images off screen; the user can't pick from a list they can't see):

1. **Whose** images to analyze — these catalog ones, or their own. This half survives any amount of impatience: they may have better photography you can't see.
2. **If catalog: which ones.** `analyze_product_images` takes at most 5 and the catalog often returns more — and which five best show the product's features is their judgment, not yours. Pre-fill the default (the first 5, `MAIN` variant first) so a bare "use the catalog ones" or "go" accepts it; a picky user answers by number ("1, 3, 4, 6, 7") off the rendered list.

Both halves ride the same message as the Step 1 render — this adds no round trip. What impatience never buys is skipping the message: silence accepts the *default five*, it never re-decides *whose*.

## Step 2 — `analyze_product_images`

**At least one image is mandatory on both paths** — the tool rejects zero images, and `product_info` must be its output. A user with no photos at all cannot be served yet: say so and ask for a single usable image (packaging, a sample shot); never hand-build `product_info` to get around it.

**Never re-run the analysis to "refresh" or "double-check".** The call is non-idempotent — the same images can return different attributes — so a re-run silently discards whatever corrections the user made at ②. The same holds for `generate_listing`: a second call is a different listing, not an improvement pass; refinement is Step 5. (A call that *failed* is different — retrying it duplicates nothing; see When a Non-Destructive Call Fails.)

Two input channels, and picking the wrong one is not a schema error: `image_urls` for Step-1 catalog URLs or public URLs the user pastes; `image_refs` for **local files** — and **the two channels mix**: up to 5 images combined in one call, so two catalog URLs plus three local files is one call carrying both parameters, not a choice between them. Never base64 a file on disk into a data URL — a well-formed data URL passes validation, so nothing stops you, but you'll have burned a large share of the conversation on encoded bytes.

**For local files:** call `request_image_upload_url` first. Never *guess* the count or format — but derive them rather than asking when you can: if you can see the filesystem, list the directory and read the exact count and extensions off it. Asking costs a round trip for a fact the user doesn't uniquely hold, which is the kind of gate that burns credibility. Ask only when the files aren't visible to you. Then by client capability: run each returned `curl_command` if you can execute HTTP, otherwise hand the user the returned upload page and collect the `image_ref` values they paste back.

**One batch carries one `content_type`** — mixed jpg/png means one `request_image_upload_url` call per format, since the type is signed into the presigned URLs and a mismatched PUT fails. **Slots are single-use and expire** (`expires_at`): a failed upload, or a user who wanders off and returns, needs a fresh batch — re-request for the failed files only (refs that already uploaded stay valid); don't retry the old URL. Check `max_bytes` before pushing large photos.

**Then render the seven returned attributes and send ask ② — one message, these three items only:**

1. The attributes themselves, for correction. Merge whatever the user changes into `product_info`.
2. `brand_asin` — mirror one competitor's writing style.
3. `rufus_asins` — weave in Alexa for Shopping Q&A content (legacy parameter name for what Amazon renamed from Rufus in 2026).

Keyword sources and competitor ASINs are **not** in this message — they are ask ③, after this one settles. See the note under The Flow for why.

**Make correcting cheap.** Render the attributes so a partial reply works — label each one and invite edits by name ("fix any of these, or say go"), rather than asking them to restate the whole set. Put items 2 and 3 last, phrased so silence means no, so a user with nothing to correct can answer the whole checkpoint with one word.

**If they correct anything, show the corrected set back before ③.** They need to see that the edit landed the way they meant, and a merge into `product_info` is exactly where a misread instruction becomes invisible. Keep it short — the changed fields are enough, not another full table. If they then correct again, loop; two or three passes here is normal and much cheaper than generating a listing from a wrong attribute. Do not re-ask items 2 and 3 during that loop, and do not re-ask them later in the flow.

## Step 3 — `find_keywords_with_multi_source`

**Competitor ASINs are the main lever on Keyword Bank size.** The bank is built by reverse-looking-up keywords from the ASINs you hand it, so more source ASINs means a bigger bank — and CPS / competitor-rank metrics only populate when competitor data exists. On Path A (no ASIN yet) they're the *only* way the call can be made at all.

This flow has no competitor-discovery step, and the user is the authority on who they actually compete with. **Never invent them:** guessed ASINs silently return keywords for someone else's product and the whole listing gets built on them with no visible symptom. Ask them alongside the sources in ask ③ — the same message that already has to go out, so they cost no extra round trip and keep a thin bank from costing one later. If the user genuinely doesn't know their competitors, `search_products` (outside this flow) can *propose* candidates — propose, never adopt: fold the proposal into the ③ message, and what reaches `competitor_asins` is only what the user confirmed.

**`sources` is a must-ask, not a preference poll.** Two of the seven return zero keywords **silently, with no error**, gated on whether the user's store is connected in Helium 10 — a fact you cannot detect. Pick silently and you may build a listing on four sources while believing you used six.

The seven, by exact enum value (⚠️ = the two store-connection-gated ones):

`aba_converting_keywords` ⚠️ · `sqp_sales_keywords` ⚠️ · `top_keywords` · `opportunity_keywords` · `historical_top_keywords` · `top_performing_organic_keywords` · `top_performing_sponsored_keywords`

**Never reconstruct this list from memory when presenting it** — these are exact enum strings, and a test run that skipped the reference invented five of the seven names and pinned the ⚠️ on the wrong two. Anchor the names here, and take each source's one-line explanation verbatim from **`references/keyword-sources.md`** (the `sources` parameter schema carries the same guide). Present all seven with the two gated ones flagged; default to `aba_converting_keywords` + `top_keywords` if they have no preference. → **`references/keyword-sources.md`** also covers what to say when a gated source returns nothing.

**`filters` is accepted and IGNORED in this release** — the server applies its own per-source defaults, so "only keywords above 1,000 searches" cannot be pushed into the call. Omit the parameter. Never claim the results were filtered; apply their cut where it actually takes effect — when you build the allocation — and say so. Composition: cut the bank first, then run the allocation recipe on the survivors — the recipe's "highest-scoring" means highest among rows that passed the user's cut.

### Render Two Tables

Render the **Keyword Bank** and then the **slot allocation** — both in the ask ④ message, before `generate_listing`. That pair of tables plus ④ **is** the binding approval; don't show a "preview" now and a real table later.

Because they share one message, the bank is the one thing in this flow you **do** truncate: **top 30 rows, plus every allocated phrase and every user-supplied phrase wherever they rank, plus a line saying how many there are in total and offering the rest.** The bank can run to a few hundred rows, and a full-length table sitting above the approval request turns the checkpoint into a wall, and a wall gets skimmed instead of audited — which costs the user the very review the checkpoint exists for. The allocation table itself never gets abbreviated. → **`references/rendering.md`** for exact columns, sort order, the `cps` `X/10` format, the truncation line, and what happens if they ask for the full list.

## Keyword Allocation Recipe

Produce exactly this, in this order, from the Keyword Bank sorted by `priority_score` descending. What *you* propose comes only from the Keyword Bank — the user may add their own phrases on top (see below).

1. **TITLE** — the 3 highest-scoring keywords. Not required to be semantically related.
2. **Each BULLET** (5 by default) — take the highest-scoring *unallocated* keyword as that bullet's single theme, then add ~2 of its nearest-meaning keywords. One bullet = one product feature. Bullets outrank the description when competing for higher-priority keywords.

   **"Nearest-meaning" is not "next by rank."** Working straight down the ranked list in chunks of three produces five bullets that all sell the same feature in different words — the shape this recipe exists to prevent. When the bank is synonym-heavy (every phrase a variation of the head term), the themes hide in the *modifiers*: form factor ("folding"), use location ("dorm", "couch"), activity ("writing"), material, audience. Rank still picks each theme keyword; its companions are chosen by shared facet, wherever they sit in the list. The test: if swapping two bullets' keyword sets changes nothing, they aren't themes yet.

   Render each bullet's theme in its own column so the user can audit the one-feature-per-bullet shape — see `references/rendering.md`.
3. **DESCRIPTION** — the next 3 highest-scoring unallocated keywords. Not required to be semantically related.

Three per slot by default. The arithmetic behind the 21-keyword threshold: 3 (title) + 5×3 (bullets) + 3 (description) = 21.

**3 is the default, not the limit: each slot accepts up to 6, and the bullet count can be 5–10.** So a user who moves a fourth phrase into the title, or asks for 7 bullets, is asking for something the tools accept — do it, don't demote one of their existing picks to stay at 3 and don't treat it as needing a schema check first. Beyond 6 in a slot or 10 bullets, say which limit you hit and let them choose what drops.

`bullet_keywords` is **one array per bullet** — a list of lists, and its outer length is what sets the bullet count. Flattening it into a single list of 15 destroys the one-theme-per-bullet grouping the recipe exists to produce, and nothing in the response will tell you that happened.

**Phrases the user supplies themselves are allocatable.** They arrive with no `priority_score`, so they can't be ranked against scored rows — place them where the user says, or at the top of the slot they name, and pass them in `all_keywords` as phrase-only rows alongside the scored ones. Leave their score cell blank when you render. Don't discard a phrase because it lacks a score; the thin-bank ask below explicitly invites them. They do count toward filling the slots, but re-calling `find_keywords_with_multi_source` won't reflect them — the warning is computed from what that tool fetched, so it will read the same however many phrases the user hands you. Fill the slots and move on rather than chasing a number that can't change.

Show the **full** table (all slots, not a summary) and get approval covering every slot at ask ④ — one message carrying the whole table, never a question per slot. The user may add / remove / swap / reset any slot — re-show the entire table after every edit. Only call `generate_listing` after they explicitly approve.

Do not substitute your own clustering scheme for this recipe. `priority_score` already encodes the ranking; semantics belong inside a bullet whose theme rank already chose.

## The Thin-Bank Gate

`warning.code == INSUFFICIENT_KEYWORDS` (fewer than 21 for a 5-bullet listing) is not an error and not a soft hint you can absorb. The call succeeded; nothing will stop you from continuing. It means the user can fix it and you cannot.

**The server's 21 assumes the default 5-bullet config — it doesn't know how many bullets the user wants.** If they asked for more (7 bullets need 3 + 7×3 + 3 = 27), recompute the threshold yourself: a 24-phrase bank arrives with **no warning** and is still thin for their listing. The gate below applies whenever the bank is short of *your computed* number, warning or not.

Show the keywords **and** the warning, then ask for the three things that can grow it: more `sources`, **more competitor ASINs** (usually the highest-yield fix — each one adds another ASIN to reverse-look-up), or manual keywords they already know convert. The user may choose to proceed anyway; that's their call to make, not yours to make for them.

**Whether you may widen `sources` yourself depends on how they got set.** If you defaulted them, widening once and re-calling is your call. If the user explicitly picked the set, that's a right-column answer they already gave — don't silently override it; put the additions in the same message as a suggestion and let them accept.

Do NOT: silently retry, then compress slots to 2-and-1 keywords, and mention it in a footer. That converts their decision into your disclosure.

**Once they explicitly choose to proceed thin**, the compression is their disclosed decision rather than your workaround. Build it by holding these invariants, in this priority order — do **not** just fill slots front-to-back until the bank empties, which starves the last slots to zero and drops bullets by another name:

1. Title keeps its full 3.
2. Every bullet gets at least 1 theme keyword — all 5 survive.
3. Description gets at least 1.
4. Spread whatever remains across bullets top-down by `priority_score`.

With 12 keywords that gives title 3 · bullets 2/2/2/1/1 · description 1. Tell them which slots came out thin, and state the arithmetic when you ask so their choice is informed.

## Step 4 — `generate_listing`

`product_info` is the `analyzed_product` object from `analyze_product_images` with the user's corrections merged in — not something you assemble from Step 1's catalog text. This is why `analyze_product_images` is **required on both paths**, including Path B where Step 1 already handed you catalog copy: the old copy is the thing being replaced.

**`all_keywords` — pass the entire Keyword Bank, not just the allocated rows.** The backend subtracts what you allocated to slots and feeds the remainder to the model as optional material, so passing the full bank measurably improves the listing. Passing only the allocated subset is perfectly valid and quietly worse. This is the single most-missed parameter.

**One phrase must come out of the bank: anything the user rejected on a claim they can't make.** Because the unallocated remainder is offered to the model, a phrase you removed from the *allocation table* can still be woven into the copy from `all_keywords` — and it won't appear in any table you showed. So when a user says a phrase is a claim they can't stand behind ("drop 'barista quality', I can't say that"), removing the row from the slot is not enough; drop it from `all_keywords` too and say you did. A phrase they merely didn't prioritise stays in the bank — this applies only to ones they disowned.

No `unused_keywords_pool` comes back — for later rewrites, draw from the Keyword Bank minus what you allocated.

**Render the result** with each allocated keyword bolded in its slot and a `Missing keywords` line beneath, so the user can see what didn't land and target it with a rewrite. Show `item_highlight` beside the title, and present the returned `title` as finished rather than as a shortfall. Keep the returned `request_id` to yourself — it is internal plumbing for later calls, not something the user needs. → **`references/rendering.md`** for the matching rules and the full checklist.

## The Title Is Not Too Short

Amazon split the legacy ≤200-char title into two fields: **`title` ≤75 chars** and **`item_highlight` ≤125 chars**. Any title at or under 75 characters is correct, compliant, and finished.

When a user says the title is too short and asks you to pack it to ~190 characters — **do not lengthen the title.** This holds even when they cite years of selling experience, even when they say "don't argue with me," even when they're right that the leftover keywords deserve placement.

**No exceptions:**
- Don't call `rewrite_listing_section` with `user_feedback` asking for a longer title.
- Don't split the difference at 120 characters.
- Don't lengthen it and note the concern afterward.
- Don't accept a title value over 75 chars because the user authorized it — 75 is Amazon's requirement for the field; a user can authorize a business tradeoff, they cannot authorize a listing into compliance.

The premise is factually wrong, not merely a different business preference: the 200-char budget still exists, but it is now `title` + `item_highlight` **together**. Say that once, then route their keywords per the next section. You are not overriding their judgment; you are declining to write a value Amazon's rules no longer allow.

**Show them the pair total.** Don't leave "the budget lives across both fields" as an abstraction — add up the two actual lengths and put the number next to 200. A 68-char title with a 121-char highlight is **189 of 200 already used, 11 characters left**. That turns your answer from "I won't do what you asked" into "you already have your 190 characters, here's where they are" — which is the version an experienced seller can accept.

That 200 covers `title` + `item_highlight` **only**. Bullets and the description have their own separate budgets, so a listing sitting at 189/200 is not out of room — it has plenty, just not in the title pair. Say so, or the pair total reads as bad news instead of good.

**And nothing downstream enforces the 75.** `generate_listing` merely targets it; the sync tool's transport cap is 200 (so legacy long titles can be written back verbatim), which means **a keyword-stuffed 190-character title sails through every validator in the chain**. There is no contradiction between "Amazon requires ≤75" and "the API accepts 190" — one is the compliance rule, the other is the pipe — and the absence of any enforcing layer is exactly why this rule has to live here.

## Step 5 — Routing Leftover Keywords and `rewrite_listing_section`

Leftover high-volume keywords do deserve a home — find the field that has room rather than the field next to the title. Take the first that **actually fits**: `item_highlight`, then bullets, then description. **Expect `item_highlight` to be full** (a good listing spends the whole 200-char pair budget), so bullets are usually the realistic first stop.

Two things here are must-asks, because both would otherwise put words the user never chose into copy they already approved: **which claim gets cut** if nothing fits anywhere, and **how to split** when they hand you more phrases than `target_keywords` takes — it accepts **exactly 3**, so a fourth phrase is already an overflow. And when a leftover phrase asserts a product feature you haven't seen evidence for, ask before placing it — volume doesn't make a false claim safe.

→ **`references/rewrite-and-routing.md`** for the capacity test, the overflow channels per target, the full `rewrite_listing_section` parameter rules, and the baseline you must pass.

## Authorization Attaches to an Artifact, Not a Session

"Publish it" authorizes publishing **the thing they were looking at**. If you then decline part of their request — or the content changes for any reason — the authorization does not carry over to the new version. Re-confirm against what will actually be written.

A user who says "add these keywords to the title and then publish" has authorized a listing that will never exist. Do not publish the current version under that instruction.

Reviewing and approving copy is **not** publish authorization. "the title and bullets both look good" / "looks good" / "OK, let's go with this" / "that's it then" are all content sign-off — they close a round of review, they don't command an action. Require an explicit publish instruction: "publish it" / "push it live" / "sync it to Amazon". Closing phrases like these are ambiguous in every language; when the sentence would still make sense as "no more changes", it is not an instruction to write to Amazon.

**A volunteered `sku` is not publish authorization either.** A user who hands you a SKU up front has satisfied the SKU ask and nothing else. Both boxes look ticked and they aren't: the authorization must still attach to the final content, and if that content changed after they spoke, ask again against what will actually be written.

## When a Non-Destructive Call Fails

Every tool except `sync_listing_to_amazon` is safe to retry once — a failed call wrote nothing you could duplicate. Retry quietly, then stop and say what failed and what you need. Two rules survive any failure: it never licenses skipping the checkpoint that follows it, and the error's `remediation` field — when present — names the fix; read it before improvising.

- `retrieve_listing_by_asin` empty or invalid-ASIN: the ASIN or the `marketplace` is wrong, and both are the user's facts. Ask; don't fall through to Path A as though they'd asked for a new product.
- `analyze_product_images` fails: if the error's `details` identify an offending image, drop it and re-run on the rest (at least one must remain); never substitute your own reading of the photos.
- `find_keywords_with_multi_source` errors outright (as opposed to returning few): check that `asin` or `competitor_asins` is actually populated; with neither, no retry can succeed.
- `generate_listing` / `rewrite_listing_section` fails: re-check that `product_info` is the `analyzed_product` object and that the bullet baseline text is verbatim, before assuming the service is at fault.

## Step 6 — `sync_listing_to_amazon` ⚠️ DESTRUCTIVE

The only tool that mutates the user's live Amazon data. Three things no validator can supply for you:

- `marketplace` — the same value used in the earlier steps. Don't re-ask; reuse.
- `sku` — **ask once. You cannot derive it from the ASIN**, and a wrong-but-valid SKU overwrites a different product's listing.
- `title` and `item_highlight` are a **matched pair**: syncing the title means passing both. Sending `title` alone is accepted and leaves the highlight stale.
- **Every content field is individually optional; omitting one means "leave it unchanged"** — but at least one of the four must be present or the call is rejected. After a single-bullet rewrite, publish just the `bullets` field and omit the rest.
- **`bullets` is a whole-field value, not a patch.** The parameter is a plain list with no way to address one entry, so the array you send *becomes* the listing's bullets. A rewrite changes one bullet per call — so what you publish is all five in order: the four untouched ones plus the rewritten one. Sending only the new bullet would silently erase the other four.
- **The replacement is measured against the live listing, not your session.** Amazon treats bullet points as one field: syncing fewer bullets than the listing currently has deletes the extras — a 5-bullet listing synced with 2 keeps only those 2. So whenever `bullets` is in the payload, say so in ⑤ with the counts: "replaces all bullets — publishing M, the live listing shows N" (N comes from the Step 1 render; Path A has no live listing and no shrink risk). If M < N, that line is a must-ask naming the bullets that will be deleted — only the user knows whether the shrink is intended. And if the live listing was never fetched this session (the user brought content straight to sync), N is unknown: run `retrieve_listing_by_asin` (read-only) to get it before syncing bullets, rather than guessing.

Optionally pass a `request_id` from Step 4/5 to validate against that generation.

**Afterwards, the one rule that matters: nothing here is retryable as-is.** `ACCEPTED_UNCONFIRMED` means Amazon *accepted* the submission and the ~3s read-back simply couldn't confirm it yet — resubmitting writes twice. A rejection needs the content fixed via Step 5 first; an ownership or auth error needs something changed outside this tool. → **`references/sync-outcomes.md`** for each status and error code with its correct response.


## Self-Check — read it when you feel the pull

**`references/self-check.md`** holds every rationalization that has actually been used to skip a checkpoint, each paired with why it doesn't hold, plus the observable signs you're mid-violation. **Open it the moment you notice yourself constructing a case for proceeding** — and before any `generate_listing` or `sync_listing_to_amazon` call.

The signs that you're building such a case, and should go read it:

- You're about to run the chain with zero pauses, or to pick the keyword sources yourself
- You're weighing whether a warning is "soft enough" to note rather than ask about
- You're about to write a title over 75 characters, or to split the difference
- You're treating a content approval, or a volunteered `sku`, as publish authorization
- You're about to choose something on the user's behalf and disclose it in a footer
- Your reasoning starts "the user is impatient, so..."

That last one is the reliable tell. Impatience changes how you ask — one message instead of four, recommendation pre-filled — never whether you ask.

## Where NOT to Gate

Being blanket-cautious is its own failure and destroys your credibility on the gates that matter. Do not ask permission to: pick 5 bullets, use the default bullet length, widen `sources` once on a thin bank **if you were the one who picked them**, or skip a rewrite pass they didn't request. Default, proceed, mention in one line. (Image selection is not in this list: *which* catalog images go to analysis is asked in ①, with the first-5 default pre-filled — see Step 1.)

**`retrieve_listing_by_asin` is skippable only on Path A** — a new product with no ASIN, where there is no catalog listing to fetch. On Path B it stays, even when the user has already handed you their own images: the existing copy is the baseline they need on screen to judge what's being replaced, and ① ("catalog images or their own?") has nothing to choose against without it. Supplying images answers ① early; it does not remove Step 1.

This section licenses defaults on the left column of The Two Kinds of Questions and nothing else. It never authorizes dropping an item out of the asks that apply to your current path — all five on the generation chain, ③ and ⑤ on the single-field path — and it never authorizes folding ② and ③ into one message. For images specifically: a bare "use the catalog ones" accepts the pre-filled default five — it never licenses skipping the ① question or the numbered image render it depends on.
