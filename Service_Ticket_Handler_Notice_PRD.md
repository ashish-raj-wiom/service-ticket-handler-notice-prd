# Service Ticket Chat — telling the customer who is working

| | | | |
|---|---|---|---|
| **Owner** — Ashish Raj (PM) | **Reviewer** — Akash | **Status** — Draft | **Sign-off** — Pending |
| **Version** — v0.2 · 14 Sep 2026 | **Consulted — CSP execution (TAS)** — Rahul | **Consulted — Customer chat** — Akash | |

---

## 1. Objective & Definition of Success

**Objective.** A customer with an open service ticket can tell, without calling anyone, that a named person is now working on their problem — and can reach that person in one tap.

**Boundary.** This spec governs the chat message sent to a customer when the **assignee** of their service ticket is set or changes. It covers restore service tickets on every creation path — IVR, customer chat and Kapture-direct alike — and it is the customer's only new surface: the chat thread.

It leaves unchanged: how a ticket is created, classified, deadlined, verified or closed; the CSP app and the actions in it; the existing CSP-side and technician-side notifications that already fire on the same events; the complaint chat intake gates and the resolution message the customer already receives. The masked-calling PIN workflow is reused unchanged: this spec decides when it fires, never what it says (§4, message 2). The contact-card workflow is **not** reused unchanged — R2e and G2 require two changes to it, set out in §9.

**Shifting is out of scope, and must be filtered out.** The `ES_RESTORE_TECHNICIAN_ASSIGNED` event does not distinguish task family, so without an explicit check a shifting customer would receive this feature by default — and the chat message copy is complaint-framed ("आपकी शिकायत" / "Your complaint… resolving your issue"), which is wrong for a request to move a connection. Shifting already has its own customer workflows (`ticket_type_5_created`, `ticket_type_5_tat_breached`, `ticket_type_5_resolved`) and is already barred from the PIN and card path, which accepts only `NBREC`, `INSTALL` and `RESTORE`. Excluding it here is therefore consistent with how the estate already treats it. R1 MUST NOT (d) makes the exclusion a requirement rather than an omission. **This leaves a known, deliberate gap**: 94% of shifting candidates are assigned a technician, a median 4.4 hours after creation, against a 96-hour TAT, and that customer is told nothing — the same problem this PRD solves for restore. It is V2 work, not a non-problem. Customers whose app is too old to receive a chat see nothing new, and this spec does not add another channel to reach them (M1's ceiling, not a defect).

### Guardrails — promises that hold on every path

| ID | Guardrail | One line | Anchors |
|---|---|---|---|
| G1 | **Current person only** | The newest card in the thread names the person assigned at the moment its action happened, so the name a customer is holding is never a superseded one. | R1a · R2a · AC-CHANGE-1 · AC-RACE-2 · MQ-2 |
| G2 | **Every card works** | A contact card appears only when a masked route or a live direct number actually resolved; otherwise the chat goes out without one. | R2a · R2b · R2c · R2d · AC-CARD-4 · MQ-3 |
| G3 | **Live contact source only** | Names and numbers come from the live contact record (§8) — never from an older copy of those details held elsewhere. | R2 MUST NOT (c) · AC-GRD-1 · MQ-3 |
| G4 | **One action, one chat** | Every assign action triggers exactly one chat — never two for a single action, and never none because the assignee is unchanged. | R1a · R1e · AC-DUP-1 · AC-CHANGE-3 · MQ-1 |
| G5 | **Silence after closure** | No chat is triggered by an action that happens after the ticket reaches a terminal state. | R3a · AC-CLOSE-2 · MQ-1 |
| G6 | **The card reaches the person it names** | The call action connects the customer to the assignee named on that card — never to someone else standing in for them. | R2a · R2c · R2 MUST NOT (b) · AC-CARD-2 · AC-GRD-3 · MQ-3 |

### Success metrics

| ID | Metric | Baseline | Target | Source |
|---|---|---|---|---|
| M1 | Of tickets that reach an assignee and whose customer can receive a chat ⚠️ *AI GENERATED — review*, the share where at least one chat arrived before the resolution message | 0% — new capability | **> 99%** | MQ-1 |

**Counted alongside M1, not inside it:** the share of tickets reaching an assignee whose customer could not receive a chat at all. They are outside M1's denominator, so this keeps them from disappearing (MQ-1).

**Invariant (not a metric):** G3 cards carrying a name or number from anything but the live contact record = 0, zero tolerance. Monitored via MQ-3, not trended.

---

## 2. User Stories & Rules

| ID | Story | MUST | MUST NOT |
|---|---|---|---|
| R1 | As a customer with an open service ticket, I want to know who is working on it and when that changes, so that I stop calling to find out. | **(a)** Trigger the chat every time the assignee is set or changes — the CSP taking it himself, a technician assigned, a technician swapped, the job recalled, or the same person assigned again. **(b)** Use the same text every time, whoever the assignee is. **(c)** Do this for every service ticket, whatever channel it was created on. **(d)** Send it in the customer's own language, one only. **(e)** One chat per action: an action that reaches the system more than once — a double tap, a retry, a redelivered event — still produces one chat. | **(a)** Send anything before a person has taken the ticket. A ticket merely sitting with the CSP has no assignee. **(b)** Send two chats for one action. **(c)** Stay silent on a real action because the person happens to be unchanged. **(d)** Send anything for a candidate whose task family is not `RESTORE`. The check is explicit: the triggering event does not carry the distinction, so silence here would send complaint wording to a customer who only asked to move their connection. |
| R2 | As a customer, I want to reach the person working on my ticket in one tap, without hunting for a number. | **(a)** Alongside every chat, send a contact card naming the assignee, carrying a call button. **(b)** Where masked calling is available for the ticket, put the PIN message before the card and route the call through the masked number. **(c)** Where it is not, the card calls the **assignee's own** direct number, read from the live contact record. **(d)** Where no route resolves at all, send the chat with no card. **(e)** Where the assignee's name cannot be found but a route can, send the card with its call button and no name. | **(a)** Show a card with no working route behind it (G2). **(b)** Name one person on the card and connect the customer to another (G6). **(c)** Read the name or number from any source other than the live contact record (G3). **(d)** Put a different person's name, or a stand-in name, on the card in place of the assignee's. |
| R3 | As a customer, I want the updates to stop when my ticket is done, so that a closed ticket never looks live. | **(a)** Trigger no chat from an action that happens after the ticket reaches a terminal state. | Hold back a chat that was triggered before closure merely because it will land after the resolution message. |

---

## 3. System Behaviour

### 3a. System flow chart

```mermaid
flowchart TD
    A["CSP acts on the ticket: takes it himself, assigns a technician, swaps technician, or recalls"] --> A2{"Task family on the candidate?"}
    A2 -- "SHIFTING — a move request" --> A3["Nothing sent — outside this spec (§1 Boundary, R1 MUST NOT (d))"]
    A2 -- "RESTORE — a service complaint" --> B{"Ticket already in a terminal state?"}
    B -- "Yes" --> C["T6 — no chat message"]
    B -- "No" --> D{"Has the customer been told about anyone yet?"}
    D -- "No" --> D2{"Did the CSP take it himself?"}
    D2 -- "Yes" --> E["T1 — first chat message, CSP is the assignee"]
    D2 -- "No" --> F["T2 — first chat message, technician is the assignee"]
    D -- "Yes" --> G{"Is this the same action reaching us again?"}
    G -- "Yes" --> H["T4 — one action, one chat: nothing further sent"]
    G -- "No" --> I["T3 — chat triggered, card names the assignee"]
    J["Ticket reaches COMPLETED or CANCELLED"] --> K["T5 — closed, no further chat messages"]
    E --> L{"Chat message reaches the customer?"}
    F --> L
    I --> L
    L -- "No" --> M["T7 — undelivered, recorded"]
    L -- "Yes" --> O{"Any call route for this assignee?"}
    O -- "No" --> P["T1 / T2 / T3 side-effect — chat message only, no card"]
    O -- "Masked calling available" --> Q["T1 / T2 / T3 side-effect — chat message, PIN message, card"]
    O -- "Direct number only" --> R["T1 / T2 / T3 side-effect — chat message, card, no PIN"]
```

**Precedence:**

- **P1 — Closure wins a tie.** An assign action and the ticket's closure landing at the same instant resolve closure first; the assign action is then dispatched fresh and falls to T6, generating no chat message (AC-RACE-1).
- **P2 — Order of action, not order of arrival.** A chat message generated by an action that happened before closure is delivered even if it lands in the chat after the resolution message. Chat messages are never dropped for arriving late (AC-RACE-2).

### 3b. State transition table — canon

Lifecycle of **who the customer has been told about** — one per ticket, created the first time we tell them who is working on it. The ticket's own lifecycle — creation, classification, deadline, verification, closure — and the CSP's execution states are out of scope; they appear here only as triggers.

**Precondition for every row below:** the candidate's task family is `RESTORE`. Shifting candidates are handled by the same service, on the same records, and are separated only by that field — a `SHIFTING` candidate never enters this lifecycle at all (R1 MUST NOT (d), §3a).

| ID | From | Action / Trigger | Rule / Check | To | Side-effects |
|---|---|---|---|---|---|
| T1 | — | CSP takes the ticket himself | Ticket not in a terminal state | Told about the CSP | Chat triggered (R1a); contact card sent naming the CSP, preceded by the PIN message inside the cohort (R2a, R2b) or on its own outside it (R2c); the CSP is now who the customer has been told about (R1a). |
| T2 | — | CSP assigns a technician, no chat sent yet | Ticket not in a terminal state | Told about the technician | Chat triggered (R1a); contact card sent naming the technician, per R2a–R2d; the technician is now who the customer has been told about (R1a). |
| T3 | Told about X | Any further assign action: a different technician, the CSP recalling the job off X, **or X assigned again** | Ticket not in a terminal state | Told about the person just assigned | Chat triggered again, same copy (R1a, R1b); **a fresh contact card sent naming the new assignee**, route re-resolved for them (R2a–R2d, G6), so the newest card in the thread is always the current person (G1); the new person is now who the customer has been told about (R1a). |
| T4 | Told about X | One assign action reaching the system a second time — a double tap, a client retry, a redelivered event | The action has already been processed | Told about X (unchanged) | **No second chat** (R1e, G4) — the first one already went. Recorded as a duplicate so MQ-1 can tell it from a failure. Telling one action's duplicate from two real actions is the implementer's; the promise is one chat per action. |
| T5 | Told about X, or nobody yet | Ticket reaches COMPLETED or CANCELLED | — | Closed | No side-effect of its own. Chats already triggered still deliver (P2, R3 MUST NOT). |
| T6 | Closed | Any later assign, swap or recall action | — | Closed | **No chat triggered** (R3a, G5). |
| T7 | Any step that would trigger a chat | The chat cannot be delivered — the customer's app is too old to receive it, there is no chat identity, or the send fails | — | Unchanged | **Envelope:** the customer receives nothing and is told nothing later; no recovery is promised (Override O1). The miss is recorded against the ticket and counts as a failure for M1 (MQ-1). Who the customer has been told about does **not** change, so the next assign action still triggers a chat (R1a). |

**What T1, T2 and T3 send.** Each triggers the chat, then the contact card — with the PIN message between them inside the masked-calling cohort, so two messages outside it and three within. This mirrors the existing create-ticket flow, which sends its text and then requests the masked-call connection separately. The copy is fixed and identical on all three; it never names anyone. The assignee's name is carried on the card.

**Degradation inside T1, T2 and T3** — variants of the same emission, not separate states:

| Condition | Customer receives |
|---|---|
| Masked calling available for the ticket (R2b) | Chat message · PIN message · card naming the assignee, calling through the masked number |
| Masked calling unavailable (R2c) | Chat message · card naming the assignee, calling their own direct number — no PIN message |
| Assignee's name unresolved (R2e) | Chat message · card with the call action but no name shown |
| No route resolves at all (R2d) | Chat message only, no card (G2) |

---

## 4. Screen Requirements

**Experience intent:** the customer should feel accompanied — a real person, named, now has this, and reaching them is one tap away.

**Master design file:** ⚠️ *AI GENERATED — review* **Not yet created.** No design file exists. The copy below is the message already in production; the card's visual treatment is the existing contact card used by the create-ticket flow.

Each time the assignee is set or changes, the customer gets **two or three messages**, in this order:

| # | Message | When |
|---|---|---|
| 1 | Chat message | Always (R1a) |
| 2 | PIN message | Only inside the masked-calling cohort (R2b) |
| 3 | Contact card — assignee's name + call action | Whenever a route resolved (R2a, R2d) |

### Message 1 — the chat

**States:** sent (an assignee was set or changed — T1, T2, T3) · not sent (no assignee yet, ticket terminal, or an app too old to receive it — T6, T7)
**Freshness:** appended when generated. No delivery window is committed — see Override O1. Latency is measured, not promised (MQ-4).

Copy in production, one language per customer, not both:

> **Hindi** — आपकी शिकायत (टिकट ID: `<ticket_id>`) के लिए **एक ख़ास इंजीनियर असाइन हो गया है** और वो आपकी समस्या पे जोर शोर से काम कर रहे है।
>
> **English** — Your complaint (Ticket ID: `<ticket_id>`) has been **assigned to a dedicated engineer**, and they are working hard on resolving your issue.

| Element | Source / Routes to | Logic |
|---|---|---|
| Field — ticket id | the service ticket the chat message belongs to | Always shown, so a customer with more than one open ticket can tell them apart. |
| Field — body copy | fixed copy above | Identical on T1, T2 and T3 — first chat message, technician swap and recall read the same (R1b). **Carries no name**: the person is named on the card, not here. |
| Check — language | customer's language preference | One language is sent, never both (R1d). |

### Message 2 — the PIN message (masked calling only)

**This spec does not define this message.** It is the live masked-calling workflow, already in
production and already sent by the create-ticket flow. This feature triggers it, unchanged, on
every chat where masked calling is available. Its text, its PIN and its behaviour are owned by
that workflow, not here (§1 Boundary).

**States:** sent (masked calling available for this ticket — R2b) · not sent (unavailable — R2c)
**Freshness:** sent with the card, immediately after the chat.

Illustrative only — what that workflow sends today, quoted so the reader knows what the customer
sees between the chat and the card:

> इंजीनियर से कनेक्ट करने के लिए कॉल के वक़्त PIN पूछा जा सकता है। आपका PIN: `<pin>`

| Element | Source / Routes to | Logic |
|---|---|---|
| Trigger — masked-call workflow | the existing live workflow, invoked per ticket | Invoked on every chat message where masked calling resolved (R2b), including a swap (T3). This spec owns *when* it fires; the workflow owns what it says. |

### Message 3 — the contact card

**States:** named with call action (name and route both resolved) · unnamed with call action (name unresolved, route resolved — R2e) · absent (no route resolved — R2d)
**Freshness:** sent immediately after the chat, or after the PIN message where that is sent. A swap (T3) sends a **fresh card**; the newest card in the thread always names the current assignee (G1).

| Element | Source / Routes to | Logic |
|---|---|---|
| Field — assignee name | live contact record for the assignee (§3b, §8) | The CSP's name when the CSP took it himself, the technician's when one is assigned. Omitted — never substituted — when it cannot be resolved (R2e). |
| Action — call | masked number where available (R2b), else the assignee's own direct number (R2c) | Offered only when a route resolved (R2d, G2). Connects to the person named on this card and no one else (G6). |
| Check — route resolution | — | If no route resolves, the card is not sent at all; the chat still goes (R2d). |
| Check — contact source | — | Name and number came from the live contact record; a value from any older copy is never rendered (G3). |

---

## 5. Configurability

**This feature introduces no configurable parameters.**

Three numbers touch it, and none of them is ours to set:

| The number | Who owns it |
|---|---|
| The minimum app version that can receive a chat at all | The chat platform's own rollout gate. Below it the customer gets nothing (T7) — a ceiling on M1, not a setting of ours. |
| The PIN, and the ticket type used to request it | The masked-calling workflow, reused unchanged (§1 Boundary, §4, message 2). |
| Delivery speed | Not committed at all. Best effort, measured through MQ-4 and never promised — see Override O1. |

There is no cap on how many chats one ticket may send. A chat follows every genuine change of assignee by decision, and duplicates are suppressed by person rather than by count (R1e, G4).

---

## 6. Measurement

| ID | The system must be able to answer… | Feeds |
|---|---|---|
| MQ-1 | For each service ticket: how many chats were triggered, who each card named, whether each was delivered, was a duplicate, or failed — and whether the customer could receive one at all. | M1 · G4 · G5 |
| MQ-2 | For each card: whether the person it named was the assignee at the moment the action occurred. | G1 |
| MQ-3 | For each card: whether it was sent, whom it named, which route it carried (masked, direct, or none), which record the name and number came from, and whether the call reached the person named. | G2 · G3 invariant · G6 |
| MQ-4 | For each delivered chat message: the time between the CSP's action and the chat appearing in the thread. | Override O1 — measured, never committed |

---

## 7. Acceptance Criteria

Every criterion below runs on one scenario, so nothing has to be held in your head:

| | |
|---|---|
| Customer | **Sunita Devi**, account `WN4471203` |
| Ticket | **1787745414303000**, raised 11 Sep 2026 09:14 IST |
| CSP | **Ramesh Kumar** — own number `9812345670` |
| Technicians | **Imran Sheikh**, **Vikas Yadav** |
| Masked calling | DID `08047106321`, PIN `015564` |

The groups run in the order the customer experiences them.

### START — the first chat on a ticket (T1, T2)

*Proves a chat is triggered the first time someone takes the ticket, however the ticket arrived, and never before.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-START-1 | **Given** no chat has been sent on the ticket, **When** Ramesh takes it himself at 09:31, **Then** a chat is triggered, its text carries no name, and the card that follows names Ramesh. | R1a · T1 | Settled |
| AC-START-2 | **Given** no chat has been sent on the ticket, **When** Ramesh assigns Imran at 09:31 without taking it himself first, **Then** a chat is triggered, its text carries no name, and the card that follows names Imran. | R1a · T2 | Settled |
| AC-START-3 | **Given** the ticket was created by a call-centre agent in Kapture and Sunita has never used chat, **When** Ramesh assigns Imran, **Then** the chat still reaches her. | R1c · T2 | Settled |
| AC-START-4 | **Given** the ticket is open and untouched since 09:14, **When** two hours pass with no CSP action, **Then** nothing has been sent at any point. | R1 MUST NOT (a) · T1 · T2 | Settled |
| AC-START-5 | **Given** Sunita's language preference is Hindi, **When** Imran is assigned, **Then** she gets the Hindi text only, and not the English one as well. | R1d | Settled |

### CARD — what arrives with the chat (R2)

*Proves how many messages arrive, what the card names, and where its call button goes.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-CARD-1 | **Given** masked calling is available for the ticket, **When** Imran is assigned, **Then** Sunita gets three messages in order — the chat, the PIN message showing `015564`, and a card naming Imran — and no personal mobile number appears in any of them. | R2a · R2b · T2 | Settled |
| AC-CARD-2 | **Given** masked calling is **not** available for the ticket, **When** Imran is assigned, **Then** Sunita gets two messages — the chat, then a card naming Imran whose call button dials `9812345670`, Imran's own number — and no PIN message. | R2c · G6 · T2 | Settled |
| AC-CARD-3 | **Given** masked calling is not available, **When** Ramesh takes the ticket himself, **Then** the card names Ramesh and dials Ramesh's own number. | R2a · R2c · G6 · T1 | Settled |
| AC-CARD-4 | **Given** neither masked calling nor any direct number resolves for Imran, **When** he is assigned, **Then** the chat is still sent and no card is sent at all. | R2d · R2 MUST NOT (a) · G2 · T2 | Settled |
| AC-CARD-5 | **Given** Imran's name cannot be found but his number can, **When** he is assigned, **Then** the card is sent with a working call button and no name on it — and no other person's name in its place. | R2e · R2 MUST NOT (d) · T2 | Settled |

### CHANGE — a further assign on the same ticket (T3)

*Proves every later assign sends again — a different person, the job coming back, or the same person once more.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-CHANGE-1 | **Given** Sunita has a card naming Imran, **When** Ramesh swaps to Vikas at 11:02, **Then** a chat is triggered again, a fresh card naming Vikas appears below it, and the newest card in her chat names Vikas rather than Imran. | R1a · R2a · T3 · G1 | Settled |
| AC-CHANGE-2 | **Given** Sunita has a card naming Imran, **When** Ramesh recalls the job off Imran at 11:02 and it returns to him, **Then** a chat is triggered with the same text and a fresh card naming Ramesh appears. | R1a · R1b · T3 | Settled |
| AC-CHANGE-3 | **Given** Sunita has a card naming Imran from 09:31, **When** Ramesh assigns Imran again at 11:40 as a fresh action, **Then** a chat is triggered again and a fresh card naming Imran appears. | R1a · R1 MUST NOT (c) · T3 · G4 | Settled |
| AC-CHANGE-4 | **Given** masked calling is available for the ticket and Sunita already has a card naming Imran, **When** Ramesh swaps to Vikas, **Then** the PIN message is sent again alongside the fresh card. | R2b · T3 | Settled |

### DUP — one action reaching us more than once (T4)

*Proves a double tap or a retry costs the customer nothing.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-DUP-1 | **Given** Ramesh taps assign-Imran once at 09:31:00, **When** that single action reaches the system five times within ten seconds, **Then** exactly one chat and one card naming Imran reach Sunita, and the repeats are recorded as duplicates rather than as failures. | R1e · T4 · G4 | Settled |

### CLOSE — the ticket finishes (T5, T6)

*Proves the chat stops when the work does.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-CLOSE-1 | **Given** Sunita has a card naming Imran, **When** the ticket is marked resolved at 12:15 and closes, **Then** her chat holds that chat and card, then the resolution message, and nothing further is triggered for the ticket. | R3a · T5 | Settled |
| AC-CLOSE-2 | **Given** the ticket closed at 12:15, **When** an assign-Vikas action arrives at 12:18, **Then** nothing is sent and Sunita's chat is unchanged. | R3a · T6 · G5 | Settled |

### WF — whole journeys

*Proves the pieces hold together across a ticket's life, including the short one.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-WF-1 | **Given** the ticket raised at 09:14, **When** Ramesh takes it at 09:31, assigns Imran at 09:48, swaps to Vikas at 11:02, and it resolves at 12:15, **Then** Sunita's chat holds three chats each with its own card — naming Ramesh, then Imran, then Vikas — followed by the resolution message, and the newest card names Vikas. | T1 · T3 · T5 · G1 · G4 | Settled |
| AC-WF-2 | **Given** the ticket raised at 09:14, **When** Ramesh takes it at 09:31 and marks it resolved at 09:33, **Then** Sunita gets the chat and Ramesh's card, then the resolution message about two minutes later, and neither is held back because of the other. | T1 · T5 · P2 | Settled |

### FAIL — the chat cannot be delivered (T7)

*Proves a failed send is recorded honestly and never becomes permanent silence.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-FAIL-1 | **Given** Sunita's app is too old to receive a chat, **When** Imran is assigned, **Then** she gets nothing, nothing is queued for a later app upgrade, the ticket is recorded as having a failed chat, and it counts against M1. | T7 | Settled |
| AC-FAIL-2 | **Given** the chat and card for Imran failed to reach Sunita, **When** Ramesh takes any further assign action — Vikas, or Imran again — **Then** a fresh chat and card are attempted. | R1a · T7 | Settled |

### RACE — two things at once (P1, P2)

*Proves the order things happened decides the outcome, not the order they arrive in.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-RACE-1 | **Given** Sunita's newest card names Imran, **When** a swap to Vikas and the ticket's closure land at the same instant, **Then** closure resolves first, the swap sends nothing, and the last name she has been given is still Imran. | P1 · T6 · G5 | Settled |
| AC-RACE-2 | **Given** the swap to Vikas happened at 11:02:00 and the ticket closed at 11:02:01, **When** the Vikas chat reaches Sunita at 11:02:09, after the resolution message, **Then** it is still delivered. | P2 · R3 MUST NOT · G1 | Settled |

### REG — what must not change (§1 Boundary)

*Proves this feature only adds — nothing that works today starts behaving differently.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-REG-1 | **Given** Imran is assigned to the ticket, **When** the assignment happens, **Then** the notifications Ramesh and Imran already get on the CSP and technician apps fire exactly as before, unchanged in content and timing. | §1 Boundary | Settled |
| AC-REG-2 | **Given** Sunita opens chat and reports a new internet problem while the ticket is already open, **When** the complaint flow runs, **Then** she gets the existing open-ticket reply and engineer callback exactly as today. | §1 Boundary | Settled |
| AC-REG-3 | **Given** a candidate whose task family is `SHIFTING`, **When** a technician is assigned to it, **Then** nothing from this feature is sent, and that customer's existing shifting messages fire exactly as they do today. | R1 MUST NOT (d) · §1 Boundary | Settled |
| AC-REG-4 | **Given** the same shifting candidate, **When** its assignment event reaches this feature, **Then** it is dropped once the task family is read — the event arriving is not by itself a reason to send. | R1 MUST NOT (d) | Settled |

### GRD — checks that run on live traffic

*Proves the guardrails hold in production, not only in a test.*

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-GRD-1 | **Given** an older copy of the contact details elsewhere still shows `Suresh Yadav / 9811111111` for Sunita's connection while the live contact record for Imran holds `9812345670`, **When** a card with a direct-number call button is sent, **Then** it names Imran and dials `9812345670`, and the source recorded against the card is the live contact record. | G3 · R2 MUST NOT (c) · MQ-3 | Settled |
| AC-GRD-2 | **Given** every card sent over a full day of live traffic, **When** MQ-2 is run across them, **Then** each card named the person assigned at the moment of its own triggering action. | G1 · MQ-2 | Settled |
| AC-GRD-3 | **Given** every card sent over a full day of live traffic, **When** MQ-3 is run across them, **Then** each card's call button reached the person named on that card. | G6 · R2 MUST NOT (b) · MQ-3 | Settled |

---

## 8. Glossary

| Term | Meaning | Owner (domain) |
|---|---|---|
| Assignee | **Canonical definition:** the single person currently responsible for doing the work on a service ticket — either the CSP who took it himself, or the technician he assigned. All other mentions cite this definition. | CSP execution |
| Who the customer has been told about | **Canonical definition:** the assignee named on the newest card the customer has received. It can lag the real assignee between an action and the chat that follows it. This is what §3b tracks, per ticket, and it carries the person, whether they are the CSP or a technician. It records what the customer knows; it is **never** a reason to hold a chat back (R1a). | — |
| Chat message | **Canonical definition:** the fixed-copy message triggered when the assignee is set or changes (§4, message 1). The same on a first send, a swap and a recall; it names no one. | — |
| Contact card | **Canonical definition:** the message carrying the assignee's name and a call action (§4, message 3), sent alongside every chat where a route resolves. The only place a person is named. | — |
| Live contact record | **Canonical definition:** the record where a person's own name and mobile number are maintained and kept current — the CSP's and the technician's alike, in one place. Any other copy of those details held elsewhere in the estate is stale by definition and is never read on this path (G3). | CSP identity |
| Masked calling | A call route where the customer dials a shared number and enters a PIN to be connected to the assignee, so neither party sees the other's number. Availability is decided per ticket by the IVR service, not by this spec. | IVR / masked calling |
| Terminal state | A ticket state from which no further work happens — resolved-and-closed, or cancelled. Used by R3a and G5 as the point chats stop. | CSP execution |

---

## 9. Notes for System Capabilities

What the platform must be able to do for this feature to exist. Whether these are one system or several, and how they interact, is the implementer's design.

| Capability | Needed by |
|---|---|
| Observe every change of assignee on a service ticket — the CSP taking it himself, a technician assigned, a technician swapped, a job recalled — and carry enough identity with each to reach that ticket's customer. Today only the technician-assignment signal carries customer identity; the self-assign and recall signals carry none. | T1 · T2 · T3 · R1a · R1a |
| Recognise one assign action reaching the system more than once, so a double tap or a retry sends a single chat — without mistaking two real actions for one. | T4 · R1e · G4 |
| Resolve the assignee's display name **and their own direct number** from their identifier at send time — the same lookup for a CSP and for a technician — and proceed without a name when it cannot be resolved. | R2c · R2e · G6 · T1 · T2 · T3 |
| Read those details only from the live contact record, and record which source each card used. | G3 · AC-GRD-1 · MQ-3 |
| Trigger the existing masked-calling workflow for a ticket, and tell whether masked calling is available for it. The workflow itself is reused, not rebuilt. | R2b |
| Push an unprompted message into a customer's chat thread, keyed to their account, for any customer whose app can receive one, whatever channel their ticket came from — in the customer's own language. | R1c · T1 · T2 · T3 |
| Send a contact card carrying a name and a call action, and re-send a fresh one whenever the assignee changes. | R2a · G1 · G6 · T3 |
| Show no name on the card when the person cannot be resolved, rather than a stand-in, **and** send no card at all when no route resolves. The existing card workflow does neither today: it substitutes a default name, and it sends a card with an empty number. Both are changes to that workflow, not reuses of it. | R2e · R2 MUST NOT (d) · R2d · G2 |
| Read the candidate's task family at the point the assignment event is handled, and act only on `RESTORE`. The event does not carry the field, and the two families share a service and a record shape, so nothing else distinguishes them. | R1 MUST NOT (d) · AC-REG-3 · AC-REG-4 |
| Record, per chat message, whether it was delivered, suppressed as a duplicate, or failed — and which call route and contact source it carried. | MQ-1 · MQ-2 · MQ-3 · MQ-4 |

---

## AI-generated content for review

| Location | What was generated | Basis |
|---|---|---|
| §1 M1 — denominator | "tickets that reach an assignee **and whose customer can receive a chat**" | You set the target at > 99%. Customers on an app too old to receive a chat would otherwise make that unreachable through no fault of the build, so they sit outside the denominator and are counted separately. Say if you want them counted as misses instead. |
| §4 — Master design file | "No design file exists" | The chat message and PIN copy are production copy you supplied; the card is the existing create-ticket contact card. What is missing is a design file, not a design. |

---

## Overrides

| Rule overridden | What was done instead | Rationale | Approved by |
|---|---|---|---|
| L8 / J2 — every customer-facing moment sits inside a C-id window with a specified customer-visible state | No delivery window is committed, and no configurable parameter defines one. The chat is best-effort; latency is measured through MQ-4 but never promised, and T7 promises the customer no recovery. | PM decision in interview: a progress message does not warrant the retry and recovery engineering a committed window would force. Measuring it keeps the decision reversible — if MQ-4 shows chats arriving after resolution often enough to matter, a committed window can be added without reopening the rest of the spec. | Ashish Raj, 11 Sep 2026 |
