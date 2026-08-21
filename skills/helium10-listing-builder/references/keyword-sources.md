# Keyword Sources — the seven, and which two fail silently

Read this when composing the source question in ask ③, or when a source came back with zero keywords.

## Why this is the user's choice and not yours

The seductive rationalization is "the user can't evaluate which keyword source is better, that's why they hired a tool." Two of the seven return **zero keywords, silently, with no error**, based on a fact only the user knows:

- `aba_converting_keywords` — empty unless their Amazon store is connected/bound in Helium 10.
- `sqp_sales_keywords` — empty unless their store is connected **and** the main or a competitor ASIN belongs to that store.

You cannot detect this in advance. Pick silently and you may hand back a listing built on four sources' worth of keywords while believing you used six — and the user will never know why it's weak.

## The seven

Present all of them, put the ⚠️ preconditions in front of the user whenever those two are offered or selected, and default to **`aba_converting_keywords` + `top_keywords`** if they have no preference.

| Source | What it gives |
|---|---|
| `aba_converting_keywords` *(default)* | Brand Analytics terms that convert to sales in this niche. ⚠️ needs store connected |
| `sqp_sales_keywords` | Search Query Performance terms driving the most purchases. ⚠️ needs store connected **and** ASIN ownership |
| `top_keywords` *(default)* | Highest-traffic terms the ASIN and competitors already rank for |
| `opportunity_keywords` | High-traffic terms where the ASIN ranks weakly — ranking gaps to capture |
| `historical_top_keywords` | Top keywords aggregated over a longer historical window |
| `top_performing_organic_keywords` | Best organic-rank keywords for the ASIN / competitors |
| `top_performing_sponsored_keywords` | Best-performing sponsored (PPC) keywords |

## After the call

If either gated source contributed zero keywords, tell the user the store-connection precondition is the likely cause — don't let it read as "this niche has no data."

`sqp_sales_keywords` is effectively dead on Path A: a product that isn't listed yet can't satisfy the ASIN-ownership half of its precondition.

**When widening a thin bank, add non-gated sources only.** A zero from `aba_converting_keywords` is evidence the store isn't connected, which is also half of `sqp_sales_keywords`' precondition — adding it would return zero too. Reach for `opportunity_keywords`, `historical_top_keywords`, `top_performing_organic_keywords`, `top_performing_sponsored_keywords` instead.

**Who chose the sources decides the order.** If *you* defaulted them, widen first and send checkpoint ④ against the post-widen numbers — asking before widening spends a round trip the widening might have made unnecessary. If the *user* picked the set, they already gave you a right-column answer: don't silently override it. Show the thin bank and the warning, and put the additions in that message as a suggestion for them to accept.
