# IT Helpdesk Assistant Genie — Spec (v1)

**Scope:** Ticket status & updates only, via an existing Workato connector on the Genie platform (e.g. Glean, ServiceNow Now Assist, Moveworks-style agent builder).

---

## 1. Purpose

Give employees a fast, self-serve way to check the status of their IT tickets — "where's my request," "what's happening with INC0012345," "any open tickets for me" — without opening the ITSM portal or emailing the helpdesk. Everything else (password resets, access requests, new ticket creation) is explicitly out of scope for v1 and listed under Roadmap.

## 2. How it fits together

```
 Employee ──chat──▶ Genie (platform agent)
                        │
                        │ invokes
                        ▼
                 Workato Connector
                 (pre-authenticated recipe)
                        │
                        ▼
              ITSM system (ServiceNow / Jira SM / etc.)
```

The Genie platform already has the Workato connector configured, so this spec assumes:
- Auth/OAuth between Workato and the ITSM system is handled outside the Genie (connector-level).
- The Genie calls Workato **recipes exposed as callable actions**, not raw ITSM APIs directly.

### Recipes/actions this Genie needs from Workato

| Action name | Input | Output | Notes |
|---|---|---|---|
| `get_ticket_by_id` | ticket number (e.g. `INC0012345`) | status, priority, assignee, last update, short description | Primary lookup |
| `list_tickets_by_requester` | requester email/SSO ID (from session identity, never typed by user) | array of open/recent tickets | Used for "my tickets" / "any updates for me" |
| `search_tickets_by_keyword` | free text (e.g. "VPN", "laptop") | array of matching tickets for that requester | Fallback when user doesn't have a ticket number |

All three should be scoped so they can **only return tickets belonging to the authenticated requester** — see Guardrails.

## 3. Conversation design

### Sample intents & utterances
- **Check by ID:** "What's the status of INC0012345?", "Any update on ticket 48213?"
- **List my tickets:** "What tickets do I have open?", "Show my recent requests"
- **Search by topic:** "Did my VPN issue get picked up?", "Status on the laptop request I put in last week"
- **Escalate:** "This has been open for a week, can someone look at it now?"

### Slots
| Slot | Required for | Source |
|---|---|---|
| `ticket_id` | Check by ID | Parsed from message (regex for common ID formats) |
| `requester_identity` | All lookups | Session/SSO — **never** asked of the user |
| `keyword` | Search by topic | Parsed from message |

### Sample exchange
> **User:** any update on my VPN ticket?
> **Genie:** Found it — **INC0012345**, "VPN access not working after password reset."
> Status: **In Progress** → assigned to Network Support, last updated 2 hours ago.
> Want me to nudge the assignee or open the full ticket?

### Fallback (no match)
> I couldn't find a ticket matching "VPN" under your account. Want me to list all your open tickets instead, or file a new one? (Filing isn't something I can do yet — I'll hand you to the portal for that.)

## 4. Guardrails

1. **Identity-bound lookups only.** The Genie must resolve the requester from the authenticated session (SSO), never from a name/email typed in chat — prevents one employee from fishing for another's ticket data.
2. **Read-only in v1.** No closing, reassigning, or editing tickets. No adding comments on the user's behalf without explicit confirmation shown back to them first.
3. **No PII beyond what's needed.** Return ticket subject, status, priority, assignee team (not assignee's personal contact info unless the requester is the ticket owner).
4. **Escalation path.** If a ticket is flagged urgent/blocked or the user expresses frustration/urgency, offer a clear path to a human agent (link or handoff), not just a canned "sorry."
5. **No hallucinated status.** If the Workato call fails or times out, say so plainly — never guess a status.

## 5. Error handling

| Failure | Genie response |
|---|---|
| Workato timeout/connector down | "I can't reach the ticketing system right now — try again in a few minutes, or check the portal directly: [link]." |
| Ticket ID not found | "I don't see a ticket with that number under your account — double check the ID, or I can search by keyword instead." |
| Ticket belongs to someone else | Treated identically to "not found" (no confirmation that the ID exists at all, to avoid leaking existence of other people's tickets). |
| Multiple keyword matches | List them briefly (ID + subject + status) and ask which one. |

## 6. Security & compliance

- Workato connector uses least-privilege OAuth scope: read-only ticket access.
- No ticket data is retained by the Genie platform beyond the session unless the platform's standard chat logging applies (flag this to security/compliance review).
- Audit log: every ticket lookup should log requester identity + ticket ID + timestamp on the Workato/ITSM side (standard connector behavior).

## 7. Success metrics

- % of ticket-status questions resolved without a human agent touch
- Median time-to-answer vs. portal lookup
- Escalation rate (how often users still need a human after asking the Genie)
- Deflection of "any update?" follow-up emails to the helpdesk queue

## 8. Roadmap (not in v1)

- Password reset / access requests (needs write-scope + step-up auth)
- New ticket creation (needs structured intake + category routing)
- Proactive notifications ("your ticket was just updated") rather than pull-only
- SLA breach warnings surfaced proactively to requester

---

*A working demo of the v1 chat experience (with simulated ticket data) is provided alongside this spec.*
