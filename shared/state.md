# Shared state (avoid repeating news)

State layer, fetched at run time. Runs are stateless, so the only memory an email has of what it already said is the copy Resend stored of the **last** send. Each run reads that one email, avoids repeating what it covered, and reports only what has changed on continuing stories. Then it embeds a fresh manifest in the email it sends, which becomes the memory for the next run.

Only ONE prior email is ever read: the most recent send for this routine. There is no multi-day history.

Follow this at TWO points in a run:
- **Before research** — do the Read step, and apply the dedup/delta rules while researching.
- **During assembly** — do the Emit step to build `{{MANIFEST}}` for `shared/format.md`.

<applies_to>
Only emails whose `brief.md` says to follow this file. An email that does not use state tracking substitutes `{{MANIFEST}}` with an empty string (see `shared/format.md`).
</applies_to>

<read_last_email>
Runs that send via inline recipients (`to`, i.e. `mcp__resend__send-email`). If the email sends to an `audience_id` via a broadcast, this Read step does not apply — skip it and send a full digest (see the audience note below).

1. `mcp__resend__list-emails` to get recent sends.
2. Filter to THIS routine: keep only emails whose `subject` equals this email's `config.yaml` `subject` AND whose `from` and recipient match. If `list-emails` does not return enough fields to match, `get-email` the most recent handful and match on those. (The manifest's own `email` field is the authoritative tie-breaker.)
3. Take the SINGLE most recent match. `get-email` it and read its `html`.
4. Extract the manifest: the text between the `DIGEST_MANIFEST_V1` and `END_DIGEST_MANIFEST` sentinels, and parse the JSON.
5. If there is no prior send, or it has no manifest, or the JSON will not parse: proceed with a normal full digest and no dedup. Do not stop the run.

Reading email history is read-only and safe. If `list-emails` / `get-email` are not approved yet, the first run may prompt for approval; approve them once.
</read_last_email>

<dedup_and_delta>
Apply while researching, using the parsed `items` from the last email:

- **Drop repeats.** Exclude any story whose canonical `url` or `slug` matches an item already in the manifest. Do not re-report it.
- **Delta continuing stories.** If a story maps to an existing `topic_key`, do NOT restate it. Report only what has materially changed since that item's `state` line (new figures, new developments), framed as a follow-up. If nothing material has changed, omit the story entirely.
- Genuinely new stories (no matching `url`/`slug`/`topic_key`) are reported in full as normal.

`topic_key` is a short, stable, lower-kebab-case handle for a running story (e.g. `us-cpi-june`, `iran-hormuz`, `openai-gpt-pricing`). Reuse the prior run's `topic_key` for the same story so it can be matched next time.
</dedup_and_delta>

<emit_manifest>
Build the manifest for what THIS run actually included, and pass it as `{{MANIFEST}}` to the `shared/format.md` shell. It is an HTML comment placed just before `</body>`; it renders nothing and is read back by the next run.

One `items` entry per story included in this email (skip stories you omitted). Keep `state` to a single line capturing the key facts/figures you reported, so the next run can compute a delta against it.

`run_date` is today's date as `YYYY-MM-DD` (bash: `date +%F`).

```html
<!--DIGEST_MANIFEST_V1
{
  "email": "<config.yaml name>",
  "run_date": "YYYY-MM-DD",
  "items": [
    {
      "topic_key": "us-cpi-june",
      "slug": "june-cpi-cools-to-3.5pct",
      "url": "https://www.wsj.com/economy/june-cpi-...",
      "state": "CPI 3.5% YoY; July hike odds 42%->17%; yields & USD lower"
    }
  ]
}
END_DIGEST_MANIFEST-->
```

If a run legitimately has no stories to record (e.g. a quiet day with nothing sent), still emit the comment with an empty `items` array so the next run reads a valid, empty manifest.
</emit_manifest>

<notes>
- Only the last email is read, so memory is exactly one day. A story that ran, went quiet for a day, then resurfaced can repeat — accepted trade-off for keeping memory short. (To close that gap without reading more than one email, a future version could carry forward still-fresh prior items into each new manifest.)
- Manual "Run now" tests send real emails that become the state the next run reads. Testing twice in a row will make the second run dedup against the first test — expected.
- Audience/broadcast sends: this file targets the inline-`to` send path. If an email switches to an `audience_id` broadcast, retrieval must move to the Resend broadcasts API; until then, such emails should not rely on state tracking.
</notes>
