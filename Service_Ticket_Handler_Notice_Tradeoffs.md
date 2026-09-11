# Service Ticket Handler Notice — tradeoff log

Running log from the interview of 11 Sep 2026. Becomes the tradeoffs register at finalise.
Every row is a decision Ashish made against presented options.

| # | Decision point | Chosen | Rejected options | Why (PM's stated reason) | Date |
|---|---|---|---|---|---|
| 1 | What fires the customer-facing notice | Every change of who is working on it: CSP self-assign, technician assigned, technician swapped, job recalled — 95% coverage | Technician-assigned only (39%); every state change; assigned + a time-based fallback | "Whenever the CSP assigns himself or a technician or changes him, customer should know about it." | 11 Sep 2026 |
| 2 | Delivery surface | Customer app chat, for every ticket regardless of creation channel | Chat only for chat-originated tickets; chat + WhatsApp/SMS fallback; chat now, other channels as V2 | "For all the service tickets where CSP assigns someone to work on it, irrespective of from where the ticket is created." | 11 Sep 2026 |
| 3 | Message volume, given 72% of tickets close within 5 min of the first progress event | Send everything, no suppression | Hold briefly and drop if resolution beats it; one message updated in place; only message tickets past an age threshold | Chose full transparency over noise reduction. | 11 Sep 2026 |
| 4 | What the notice tells the customer | Technician assigned + name + calling CTA; PIN where the CSP is in the IVR masked-calling cohort | Name without contact; no person named; name plus deadline/ETA | Reassurance plus a way to reach the person, without committing to a deadline the two systems disagree about. | 11 Sep 2026 |
| 5 | Calling CTA outside the masked-calling cohort | Fall back to the CSP's direct number | No CTA; fall back to the call centre; hold the message until a PIN exists | Chose reach over safety from the stale-contact bug. **Mitigation derived, not overridden:** PRD mandates the live gateway CSP record, names `t_account_mapping1` as forbidden (G3), and carries AC-GRD-1 so the item-06 bug cannot ride in silently. | 11 Sep 2026 |
| 6 | What the customer sees when the CSP recalls a job from a technician | Same message as a fresh assignment — one template for all four triggers | Neutral message with no name; silent; a distinct CSP-named variant | Fewest templates, one code path. | 11 Sep 2026 |
| 7 | Double-tap / duplicate event | One notice per actual change of person | One per event always; one per person per ticket | "Double tap should not send the same message two times. Whenever there is an assignee or if a change in assignee, message should be triggered." | 11 Sep 2026 |
| 8 | Delivery speed | Best effort, no committed window | Within 1 minute; within 5 minutes | A progress message does not warrant retry/recovery engineering. **Recorded as Override O1**, with latency measured through MQ-4 so the decision stays reversible. | 11 Sep 2026 |
| 9 | Handler's name cannot be resolved at send time | Send without a name, keep the CTA | Do not send; substitute the CSP's name | The weaker half of the message still helps; naming the wrong person does not. | 11 Sep 2026 |

## Measurements the decisions were made against

Taken 11 Sep 2026 from `PROD_DB.CSP_TAS_SERVICE_CSP_TAS_SERVICE.RESTORE_EXECUTION_CANDIDATES`
and `PROD_DB.PUBLIC.SERVICE_TICKET_MODEL`, 30-day window.

| Figure | Value |
|---|---|
| Restore candidates | 50,047 (~1,670/day) |
| Ever reached `ASSIGNED_TECHNICIAN` | 19,645 — 39.3% |
| Ever reached `ACCEPTED` | 47,555 — 95% |
| Ever reached `IN_PROGRESS` | 0 — `START_WORK` is dead in production |
| Reassigned to a second technician | 819 — 4.17% of assigned |
| Median create → assign technician | 76 min |
| Median create → completed | 196 min |
| Completed within 5 min of first progress event | 72.3% |
| Partner-assigned tickets originating in `CUSTOMER_CHAT` | 20.7% |
| `NO_TIMES_CUSTOMER_CALLED` | null for every row — M2 has no baseline |

## Code facts the PRD rests on

| Fact | Where |
|---|---|
| `ES_RESTORE_TECHNICIAN_ASSIGNED` already carries `technicianId`, `customerId`, `customerMobile`, `ticketId` | `csp-tas-service/…/restore/domain/event/outbound/EsRestoreTechnicianAssigned.java` |
| `ES_RESTORE_TASK_ACCEPTED` and `ES_RESTORE_TASK_RECALLED` carry **no** customer identity — must be widened for the 95% case | same folder |
| Both existing consumers of the assigned event are CSP-side; notification-service resolves recipients as `cspId × role` and has no customer recipient concept | `csp-notification-service/…/config/EventRoleMappingProperties.java:189` |
| Unprompted chat push already exists | `booking-service-java/…/service/IMessageOrchestratorService.java:14` |
| Masked-call PIN lookup already exists; `masked_call_available` is the cohort flag | `booking-service-java/…/service/impl/CustomerIvrService.java:19` |
| booking-service has **no** CSP identity or contact source today | `booking-service-java/src/main/java/com/wiom/client/` |
| Reassignment works only because `GUARD-ES-RESTORE-06` has zero callers; wiring it up would break decision 1 | `csp-tas-service/…/restore/domain/service/RestoreCandidateGuards.java:69` |
