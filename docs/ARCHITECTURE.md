# Architecture

## High-level

```
Webhook (POST { url })
        |
        v
Set node  ---  pull body.url into a clean field
        |
        v
HTTP Request  ---  GET https://r.jina.ai/<url>, returns plain text
        |
        v
Claude chain  ---  one call, structured JSON output (3 formats)
        |
        v
Code node  ---  parse JSON, fall back gracefully if parse fails
        |
        v
Google Docs  ---  create one doc per run, formatted with section dividers
        |
        +--> Slack notify (doc link)
        +--> Webhook response (doc id, received: true)
```

Eight nodes plus a sticky README. The flow is deliberately linear: receive, fetch, generate, parse, store, notify. No branches, no loops, no sub-workflows. The only fan-out is at the end where the Google Doc result feeds both the Slack node and the webhook response in parallel.

## Components

### Repurpose URL Webhook (`Webhook` node)

POST endpoint at `/webhook/repurpose`. Response mode is `responseNode`, meaning the workflow controls the HTTP response and can return the Google Doc ID after the doc is actually created. This makes the webhook callable by other systems that want to know "where did the draft land."

The webhook expects a JSON body with a single `url` field. There is no auth on the webhook by default. See [SECURITY.md](SECURITY.md) for how to add a shared-secret header.

### Extract URL (`Set` node)

Pulls `body.url` into a top-level `url` field. This is a one-line transform that exists for one reason: it gives the rest of the workflow a clean, predictable shape regardless of how the upstream caller nested its payload. If you change the input contract (say, accept `article_url` instead of `url`), this is the only node you edit.

### Fetch Article via Jina.ai (`HTTP Request` node)

GETs `https://r.jina.ai/<url>` and asks for a plain text response. Jina's reader endpoint takes any public URL and returns clean Markdown-ish text stripped of nav, ads, and clutter. No API key is required at the free tier (200 requests per IP per day).

This is the single biggest design choice in the workflow. See "Why Jina free reader vs paid scrapers" below.

### Claude - Repurpose Model (`Anthropic Chat Model` node)

Configures the LM the chain will use. Model is `claude-sonnet-4-6`, max tokens 2000. This node is a config carrier, not part of the data flow; it plugs sideways into the Chain LLM node via the `ai_languageModel` connection.

### Generate Repurposed Content (`Chain LLM` node)

The brain of the workflow. One Claude call with a structured prompt that asks for a JSON object with three keys:

- `linkedin_post` (150-250 words)
- `newsletter_section` (200-300 words)
- `tweet_thread` (5-7 numbered tweets)

The article body is interpolated from the upstream Jina.ai node. The voice instructions sit at the top of the prompt; this is the field every operator must edit before the workflow produces anything usable.

### Parse Repurposed Content (`Code` node)

Strips any markdown fencing (` ```json ... ``` `) the model might wrap around the response, then `JSON.parse`. If parsing fails, it falls back to dumping the raw text into `linkedin_post` rather than crashing the run. The node also stamps `source_url` and `created_at` onto the output so the Google Doc has its provenance.

The fallback matters: an unparseable response is recoverable (you read the raw text in the doc and re-run), a thrown error halts the workflow.

### Create Google Doc (`Google Docs` node)

Creates a fresh document per run, titled `Repurposed Content - YYYY-MM-DD`, in a folder you specify. The body uses plain text dividers (`---`) between the three sections. One doc per run, not one doc per source URL; simpler to audit and share.

### Slack - Doc Ready (`Slack` node)

Posts to a channel ID with the source URL and a clickable Google Doc link. Lightweight notification, not a full preview, the doc is the review surface.

### Webhook Response (`Respond to Webhook` node)

Returns `{ received: true, doc_id: <id> }` as JSON to the original webhook caller. This lets upstream systems chain on the doc ID if they want to.

## Design decisions worth calling out

### Why Jina free reader vs paid scrapers

Three options were on the table:

1. Direct HTTP fetch with a real `User-Agent` cheap, but breaks on JS-heavy sites and Cloudflare-protected pages
2. Firecrawl / Browserless reliable, but adds a paid dependency and a credential
3. Jina.ai reader free tier handles 90% of articles, no key required

For a template aimed at "I want to try this without paying for a fourth SaaS," Jina wins. The fallback when Jina fails is documented in the troubleshooting table: swap the HTTP node, the rest of the flow doesn't care.

### Why one Claude call vs three (one per format)

The single-call structured-output pattern saves money (one prompt, one set of input tokens) and keeps the three outputs internally consistent (same article context, same voice instructions). The cost is loss of per-format prompt control, you can't tune the LinkedIn voice independently of the newsletter voice without splitting later.

For a starter template, one call is the right default. Operators who outgrow it can split into three calls and the rest of the workflow downstream of the Code node doesn't change.

### Why Google Docs vs direct posting

Auto-publishing is the wrong default for AI-generated content. The Google Doc is a review surface: open, edit, copy-paste into LinkedIn / Twitter / your newsletter. Anyone who really wants auto-publish for one format can branch off the Code node and add a Buffer node, but the doc copy stays as an audit trail.

Docs over Sheets because the content is prose, not tabular. Docs over Notion because Docs has a simpler OAuth story for first-time setup; Notion is a one-line swap if you prefer it.

### Why the Webhook Response returns the doc ID

The workflow is callable. Some operators wire this into a Telegram bot, a Slack slash command, or a frontend "repurpose this URL" button. Returning the doc ID means the caller can show a "draft ready, open it here" link without polling.

### Why the prompt asks for `tweet_thread` as a single string, not an array

A string round-trips through JSON cleanly; an array of tweet objects invites the model to over-structure the output. The downstream Google Doc renders the string as-is. If you want the array form (for, say, posting straight to X), parse the numbered tweets in the Code node.

## Performance notes

| Step | Latency expectation |
|---|---|
| Webhook receive | Instant |
| Jina.ai reader | 1-5 sec depending on source size |
| Claude generate (sonnet-4.6, ~2k output tokens) | 8-15 sec |
| Code parse | <50 ms |
| Google Doc create | 1-2 sec |
| Slack notify | <1 sec |
| Total end-to-end | 12-25 sec for a typical 1500-word article |

Cost per run: roughly 0.05 USD on Claude Sonnet 4.6 at 2025 pricing, plus zero for Jina free tier. If you batch this over many URLs, budget accordingly and see [SECURITY.md](SECURITY.md) for cost-control patterns.

## Observability

- **n8n Executions panel** is the primary debugging surface. Each webhook hit shows up there with the JSON body it received.
- The **sticky README inside the workflow** carries the live setup notes. Edit it as you customize so future-you knows what's wired where.
- For longer-running deployments, consider adding a final node that writes one row per run (timestamp, source URL, doc ID, success/error) to a Google Sheet or Airtable. The executions panel only retains 30 days by default.

## See also

- [SECURITY.md](SECURITY.md) - threat model, layered defenses, what to lock down before exposing the webhook publicly
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md) - patterns shared across every template in the collection
