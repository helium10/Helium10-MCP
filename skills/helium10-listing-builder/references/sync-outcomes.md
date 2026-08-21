# Sync Outcomes and Errors

Publishing is always last. If a rewrite happens after the user authorized publishing, the authorization died with the old text — re-confirm against what will actually be written. The flow diagram shows the rewrite loop before ⑤ for that reason: there is no "publish, then rewrite" ordering.

Read this after calling `sync_listing_to_amazon`. The single most important thing here: **`ACCEPTED_UNCONFIRMED` is not a failure, and none of these errors are retryable as-is.** Retrying a submission Amazon already accepted writes twice.

## The three outcomes

| Result | What it means | What you do |
|---|---|---|
| `sync_status: CONFIRMED` | Server waited ~3s, re-read via SP-API, content matched | Tell the user it's confirmed live |
| `sync_status: ACCEPTED_UNCONFIRMED` | Amazon accepted it; the ~3s read-back couldn't confirm yet | **Not a failure. Never auto-retry.** The window is short and Amazon propagation often exceeds it, so this is common. Tell them to verify in Seller Central shortly |
| `H10.LB_SP_API_REJECTED` / `H10.LB_SYNC_VERIFY_MISMATCH` | Amazon returned INVALID, or read-back contradicted the submission | **Never blindly retry.** Inspect `error.details.issues`, fix content via Step 5, then resubmit |

## Ownership and auth errors

None of these get better by trying again — each needs something changed outside this tool:

| Error | What to do |
|---|---|
| `H10.LB_ASIN_NOT_OWNED` | Fix the ASIN, or have them connect that seller account in Helium 10 |
| `H10.LB_SKU_MISMATCH` | Get the real SKU from Seller Central. Don't retry the old one |
| `AUTH.INVALID_SP_TOKEN` / `AUTH.NO_CONNECTED_TOKEN` | Have them re-authorize or connect the Amazon account in Helium 10 |

## Always diff `synced_fields[]` against what you sent

This is the one silent failure in the response. `synced_fields[]` lists what actually went through; a field you passed can be missing from it with **no error anywhere**. If nobody compares the two, the user believes content is live that isn't.

Check every field you passed. `title` and `item_highlight` travel as a pair, so a `title` that arrives without its highlight leaves the highlight stale.

**If a field you sent is absent from `synced_fields[]`, sending it again is not a retry.** The no-retry rule above is about resubmitting a payload Amazon already accepted — that writes twice. A field that was never written has no first write yet, so submitting it alone is the correct fix, not a duplicate. Confirm the exact text with the user, then send just that field.

One caveat on timing: while `sync_status` is `ACCEPTED_UNCONFIRMED` you cannot yet distinguish "never submitted" from "submitted but not visible to the read-back." Have the user check that field in Seller Central first, and only push it if it's genuinely still the old value.

## Non-blocking issues

Amazon may attach WARNING/INFO `issues` to an accepted submission — show them, but they don't block anything. They're often independently actionable (an image-count warning, say), so surface them rather than swallowing them.
