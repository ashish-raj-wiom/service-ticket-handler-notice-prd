# Service Ticket Chat — PRD

Roadmap item 01 of the Service Tickets POD: *the customer cannot see any progress.*
When the assignee on a service ticket is set or changes, trigger the customer chat.

**Read it:** https://ashish-raj-wiom.github.io/service-ticket-handler-notice-prd/

| File | What it is |
|---|---|
| `Service_Ticket_Handler_Notice_PRD.md` | The PRD. Wiom Template v3. **The single source of truth.** |
| `Service_Ticket_Handler_Notice_Tradeoffs.md` | The 16 decisions behind it, the measurements they were made against, and the code facts the spec rests on. |
| `index.html` | Renders the markdown live from this repo. Holds no content of its own. |

## How to change the document

Edit the markdown, commit, push. The page re-renders itself — there is no build step and no
generated copy to keep in sync.

```bash
git add -A && git commit -m "PRD: <what changed>" && git push
```

GitHub caches raw files for a few minutes, so a fresh push can take a moment to appear.

`index.html` fetches the `.md` files at load time and renders them in the browser. It contains
no document text. If the two ever disagree, the markdown is right and the page has a bug.

## Status

**v1.0 — signed off 14 Sep 2026.** Reviewer: Akash. Consulted: Rahul (CSP execution), Akash
(customer chat).

Lint clean against the Wiom PRD checklist: every lettered obligation, MUST NOT, transition and
guardrail is covered by an acceptance criterion, and one override is recorded — no committed
delivery window, measured through MQ-4 instead.

Nothing in the PRD is unconfirmed: the review section is gone and no placeholder values remain.
