# Service Ticket Chat — PRD

Roadmap item 01 of the Service Tickets POD: *the customer cannot see any progress.*
When the assignee on a service ticket is set or changes, trigger the customer chat.

**Read it:** https://ashish-raj-wiom.github.io/service-ticket-handler-notice-prd/

| File | What it is |
|---|---|
| `Service_Ticket_Handler_Notice_PRD.md` | The PRD. Wiom Template v3. **The single source of truth.** |
| `Service_Ticket_Handler_Notice_Tradeoffs.md` | The twelve decisions behind it, the measurements they were made against, and the code facts the spec rests on. |
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

v0.1 Draft. Lint-clean against the Wiom PRD checklist — 0 errors, 0 warnings, 1 recorded override.

Four things block sign-off:

1. No engineering reviewer named.
2. No consulted parties named for TAS, chat or IVR.
3. No design exists for the chat or its contact card.
4. M2 has no baseline — `NO_TIMES_CUSTOMER_CALLED` is null for every row, so repeat-contact
   rate cannot be read before or after.

The `AI-generated content for review` section at the foot of the PRD lists every value that was
filled rather than decided. Those are the PM's worklist; the page badges them.
