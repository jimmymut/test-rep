# Warranty Action Hub: Reference guide

As of 1 October 2026, describing the code on `main` (including the "finalise all the feedbacks" work merged that morning).

The Warranty Action Hub replaces the team's Monday.com board. It shows every open and paid warranty claim at Taverna's two CDJR stores, who owns each one, and what has to happen next. It reads claims from DealerConnect and Reynolds by itself, so nobody types claims in. People add only what no feed knows: owners, comments, holds, hand-entered RO numbers and sign-offs.

---

## 1. Who uses it

Only people who hold a warranty role can open the hub or its API. There is no administrator exception: an Administrator without a warranty role is refused, and is never offered as an owner. Each person holds one hub role, given in Taverna OS user management.

| Role | What it is for | What it can do beyond reading and commenting |
| --- | --- | --- |
| Submitter | Sends claims to Stellantis | Set a claim's Current Owner, mark an RO Transmitted, flag an LOP issue |
| Fixer | Accountable for the claims they own; the rotation hands them work | Set a Current Owner, flag an LOP issue |
| Shop Foreman | Fixes labour-operation (LOP) problems | Set a Current Owner, flag and release an LOP issue |
| Manager | Runs the board | Everything above, plus change a Claim Owner, spread work across Fixers, set status colours, sign off claim differences |
| Accounting | Clears payment differences | Set a Current Owner, sign off claim differences |

Anyone on the hub can comment, reply, like, enter or correct an RO number, export, and use every filter.

---

## 2. Where the data comes from

Two databases sit behind the hub.

- **The source database** (`WAH_PAID_CLAIMS_DATABASE_URL`, read-only) holds what the scrapers and report importers bring in from DealerConnect and Reynolds.
- **The app database** holds everything people do on the hub.

The board's view of the source data is cached for 5 minutes, so a new scrape shows within 5 minutes of landing.

### From DealerConnect (Stellantis), scraped by the DealerConnect source

| Table | What it gives the hub | Notes |
| --- | --- | --- |
| `dealerconnect_claim_entry` | Open claims: number, VIN, type, status (Rejected, RA Submitted, Hold), amount, process date | A per-store snapshot, replaced on every scrape. A claim Stellantis pays drops out here and reappears as paid. |
| `dealerconnect_pending_claims` | Pending amounts per claim | Used when present. |
| `dealerconnect_claim_activity` | Paid claims and their history (first-scraped date, chargebacks) | Flagged stale after 6 hours without a refresh. |
| `paid_claim`, `paid_claim_lop`, `paid_claim_part` | The Claim Acknowledgement: what each claim was paid, per labour operation and per part | Flagged stale after 48 hours. Used to split a difference into labour, parts and other. |

### From Reynolds (the DMS), report files imported by Data Studio

| Report | What it gives the hub |
| --- | --- |
| RMI 1266 | The RO's warranty total, split into labour and parts. Only ROs opened in the last 3 months. This is the "Expected" figure. |
| RMI 1267 | RO lines with op codes. Used to tell recall work apart and as a second copy of RO dates. |
| RMI 1268 (Whole RO by Close Date) | Every RO closed in the last 3 months, with its close date and department type (S service, B body shop, P parts). Used to drop body shop ROs and to find older ROs. |
| Open RO report (`rmi_open_ro`) | Every RO still open and ready to send, its department, and the customer-pay invoice date. Gives Pending its "Not Submitted" rows before a claim exists. Dropped into the RMI folder several times a day. |
| `z_jeep_hub_ro_history` | ROs kept from earlier feeds, older than any report's 3 months. |

### From Monday.com

The team's old board, read through Monday's API every 3 hours (at 20 past the hour) and on demand. The hub never writes back to Monday.

### Stored by the hub itself (app database)

Owners, the rotation pointer, comments and replies, likes and views, files, the activity log, notifications, held row positions, status colours, RO workflow states (Transmitted, LOP Issue), hand-entered RO numbers, claim difference sign-offs, and the Monday copy.

### How a claim gets its RO number

Neither DealerConnect feed carries an RO number, so the hub works one out. A badge on the row says how sure it is.

| Confidence | Meaning |
| --- | --- |
| High | The claim number and the VIN both match this RO in Reynolds. No badge. |
| Entered by hand | Someone on the hub typed or corrected the RO number. It outranks every automatic match. |
| Medium | Matched by the VIN or by the claim number, not both. |
| Low | Read from the claim number alone (Plantation only, where claims are numbered after the RO). No report confirms it. |
| No match | No RO found. Enter it by hand. |

Plantation (dealer 61008) numbers a claim after its RO: `E62645` is the warranty claim on RO 62645. Fort Lauderdale (61083) does not.

### What is left out on purpose

- **New vehicle prep claims** (`K-New Vehicle Prep`) and the placeholder `PREPNV`.
- **Body shop ROs**, by RMI 1268's department type B.

**Known gap.** RMI 1268 lists an RO only once it closes. An open body shop RO coming from the Open RO report can still appear on Pending until it closes. On 29 September there were 61 such ROs. The Open RO report's own department column ("Body Shop") would close this gap.

---

## 3. The sections

The left rail (on a phone, the strip of buttons above the board) has six sections. A paid RO moves through them by how its paid total compares with the RMI 1266 warranty total.

```
                       claims still being worked
  Open RO report  ──►  PENDING  (Not Submitted, Transmitted, LOP Issue,
  DealerConnect  ──►            Rejected, RA, Hold)
                          │
                          │ every claim on the RO is Paid
                          ▼
            paid total vs RMI 1266 warranty total
            ┌──────────────┬───────────────────┬──────────────────┐
         equal to       within 2%,          more than 2% off    no warranty
         the cent       not exact                                total known
            │              │                    │                   │
            ▼              ▼                    ▼                   ▼
          PAID      MINOR ADJUSTMENT     CLAIM DIFFERENCE     PAID ▸ No warranty
        (Archived)  Accounting clears    Accounting or a        total
                    ("Adjusted")         Manager approves
                          │              or records it
                          └──────┬───────────┘
                                 ▼ signed off
                         PAID (Archived), with the decision on it
```

| Section | What is on it |
| --- | --- |
| Pending | Every RO still being worked: from the Open RO report (not sent yet) and from DealerConnect (sent, not paid). |
| Minor adjustment | Paid, within 2% of the warranty total but not to the cent. Waits for Accounting to clear it. |
| Paid | Archived: paid and matching, or signed off. No warranty total: paid, but RMI 1266 has no total to compare against. |
| Claim difference | Paid, more than 2% away from the warranty total. Waits for Accounting or a Manager. |
| Monday board | A read-only copy of the old Monday board, with its comments and files. |
| Settings | Status colours. Only Managers see it. |

**Sign-off.** An RO's sign-off holds only at the figure it was made at. A later payment or chargeback that changes the difference sends the RO back for review. A signed-off RO can be reopened.

**Claim difference board.**
- **Columns.** Expected (the RMI 1266 total), Paid, and Difference with its percent in brackets.
- **Colours.** Underpaid shows in red. Overpaid shows in amber, marked "over".
- **Order.** It opens with the largest difference first.
- **Totals card.** The card above shows Expected, Paid, Underpaid and Overpaid across every page, following the search and filters.
- **Breakdown.** The row and the drawer split a difference into labour hours, parts and other, from the Claim Acknowledgement.

---

## 4. Statuses and the workflow

Some statuses come from DealerConnect. The rest are set by people on the hub, per repair order.

| Status | Set by | Meaning |
| --- | --- | --- |
| Not Submitted | The Open RO report, or a release from LOP Issue | Ready to send, not sent yet. |
| Transmitted | Submitter or Manager | Sent to Stellantis. This is when the RO gets its Claim Owner. |
| LOP Issue | Anyone on the roster except Accounting | Held for the Shop Foreman over its labour operations. All foremen are told. |
| Rejected | DealerConnect | Stellantis rejected the claim. |
| RA | DealerConnect ("RA Submitted") | Resubmitted after a rejection. |
| Hold | DealerConnect | Stellantis suspended the claim for review (code AB4). Nothing to do until they move. |
| Paid | DealerConnect claim activity | Paid. The RO leaves Pending once all its claims are paid. |

**Moves people make**, from the drawer's grey bar:

| Move | Who | Allowed from |
| --- | --- | --- |
| Mark transmitted | Submitter, Manager | Not Submitted |
| Flag LOP issue | Submitter, Fixer, Shop Foreman, Manager | Anything except Paid or LOP Issue |
| Release to Not Submitted | Shop Foreman, Manager | LOP Issue |

**When DealerConnect overrides a manual status.**
- Paid always wins over a status set by hand.
- Any other DealerConnect status also wins, if the hub first saw it after the manual status was set. For example, a hold on a rejected claim survives while it still reads Rejected, but not once it comes back RA.
- An RO on Transmitted or LOP Issue stays on the board after it leaves the Open RO report.

**Needs Resubmit.** This is not a status. It's what the Current Owner cell says for a claim that is Rejected and held by nobody.
- When DealerConnect newly rejects a claim, the hub clears its Current Owner. The Claim Owner stays.
- Fixers see "Needs Resubmit" and take the claim into their own name to resubmit it.
- A claim that was already Rejected keeps whoever has since taken it.

---

## 5. Owners and the rotation

Every claim has two owners, kept separate on purpose.

- **Claim Owner.** The Fixer accountable end to end. It is sticky: once set it stays through every resubmission. Only a Manager can change it, and the lock icon shows that.
- **Current Owner.** Whoever must act next. It moves freely, and anyone on the roster can set it.

**When a claim gets its Claim Owner**, in this order:
1. **Already has one.** Nothing changes.
2. **Already paid.** A claim that arrives paid never gets one.
3. **A Fixer holds it.** If its Current Owner is a Fixer, that Fixer becomes Claim Owner. The rotation doesn't advance.
4. **Its RO has one.** A claim on an RO that already has a Claim Owner takes that owner, so a resubmission doesn't use up someone's turn.
5. **Otherwise the rotation picks.** It takes the next Fixer in turn.

**When the rotation runs:**
- **At transmit.** When someone marks an RO Transmitted, the person who sent it becomes Current Owner if they're on the roster.
- **On arrival.** When a claim reaches DealerConnect without being transmitted on the hub. This applies only to newly arrived claims, never the backlog.
- **Never for PDI-only ROs.**

**Spread across Fixers.** A Manager can select claims and press "Spread across Fixers" in the selection bar. The selected claims are dealt out in turn, which handles a backlog.

**Hand-off when someone leaves.** If an admin removes a person's warranty role or deactivates them while they own pending claims, the save is refused until the admin chooses where those claims go: to a named Fixer, or back to the rotation. Paid claims keep the old name as history.

---

## 6. The drawer (click any RO or claim)

The header shows the RO number, the store and the VIN (click to copy), plus two buttons: "Monday history", when Monday had this RO, and "Notify team". Below it:

- **The grey bar** holds the workflow moves (section 4).
- **RO number** shows the confidence badge and "Change". Anyone on the hub can enter or correct it. Entering a blank clears it, and the change is logged.
- **Claim difference sign-off** appears on paid ROs that need one. Accounting or a Manager picks a decision, with an optional note:
  - Approved, recorded by accounting, or adjusted by accounting.
  - Reopen undoes a decision.

**Updates tab**, the conversation:
- **Posts.** Words, @mentions and up to 5 files of 15 MB each. Enter posts, and Shift+Enter starts a new line.
- **@Name** tells that person, and **@Team** tells everyone on the hub.
- **Replies.** Replies sit under a post, one level deep as on Monday. They take mentions and files. The post's author is told about a reply.
- **Likes.** Like any post or reply, and click again to take it back. Hover to see who liked it.
- **Seen by.** Each post has an eye with a count. Opening the Updates tab records you as having seen what others wrote. Your own posts never count you.
- **Live.** Posts, replies, likes and views show on every open hub within a moment, with no reload.
- **Threads.** A comment on a claim also shows on its RO's thread.

**Activity log.** Every status change (from DealerConnect or a person), owner change, RO number entry and sign-off is listed, with who did it, their role at the time, and the exact time. It also shows the RO's dates from Reynolds:
- RO opened.
- RO closed.
- Customer-pay invoiced, from the Open RO report.

**Members.** Everyone who can open the hub, with their role.

### Notifications and e-mail

- **The bell** lists mentions, team notices, replies to your posts, assignments to you, and LOP issues if you're a foreman. Tabs: All, Mentioned (which includes team notices and replies) and Assigned to me. You can mark a notice read without opening it. Clicking a notice opens its drawer.
- **Live delivery.** Notices arrive within a moment over a live connection, with a fallback check every minute.
- **E-mail** goes only to people who haven't had the hub open in the last 3 minutes. It is Taverna-branded and sent from the company Microsoft 365 mailbox. Its button opens the hub straight onto that claim's drawer, after sign-in if needed.

---

## 7. Toolbar and table

| Feature | What it does |
| --- | --- |
| Store switch | All stores, CDJRF-PLANT or CDJR-FTL. Applies to every section at once and is remembered. Counts, totals and export follow it. |
| Search | Matches the RO number, claim number, VIN and owner. A claim-number search shows just that claim. |
| Person | Shows claims owned by one person (Claim or Current Owner), or by former members. |
| Filter | Status, claim type, RO link confidence, and dates: Date imported, Closed or Paid date, with presets for the last 7, 30 or 90 days. |
| Sort | By any column. Claim difference opens sorted by largest difference. |
| Export | A CSV of the whole board in view: every page, under the current search and filters, one row per claim. |
| Hide | Hide columns. The checkbox and RO number can't be hidden. |
| Group by | RO (parent rows with their claims) or Claim (one row per claim). |

**Columns:**

| Column | What it shows |
| --- | --- |
| RO number | With its comment count and confidence badge |
| Store | The store |
| VIN | The VIN, which copies on click |
| Date imported | When the hub first saw the claim; hover for the RO open date |
| Last activity | The latest comment, status change or owner change made by a person, for example "3 days ago". Hover for what it was. |
| Warranty total | The RMI 1266 total |
| Claim owner, Current owner | The two owners |
| Status | The status, in the board's colours |
| Close | The RO's close date |

Rows alternate in a faint stripe. On a phone, the RO number stays fixed while the rest scrolls sideways.

**Selection bar.** Ticking rows opens it at the bottom: bulk assign an owner, spread across Fixers (Managers only), or export the selection.

**Row positions.** On an unsorted, unfiltered board you can drag a row, or use its handle, to hold it at a position for the whole team. A pin shows a held row.

**Pages.** The board shows 25 rows a page by default, with 50, 100 or 250 to choose from. The hub remembers the section you were on.

---

## 8. Other parts

- **Monday board.**
  - Every item from the old board: RO, group, status, Claim owner and Next action in Monday's colours, date added, RO total, VIN and update count.
  - Clicking an item opens its comments as they were on Monday, with replies and files. Images show inline.
  - Search and a group filter.
  - Links both ways with the live RO.
  - Owners from Monday were copied onto Pending ROs that had none. This matched by first name, skipped anything ambiguous, and showed "Monday import" in the log.
- **Settings.** Managers pick each status's colour, from Monday's palette or a custom one. Reset returns to the default.
- **How this works.** The help panel, from the rail, or from the "?" on a phone.
- **Hidden for now.** The Duplicate check and Submitter queue sections are in the code but hidden from the rail until they read live data.

---

## 9. Demo walkthrough

1. **Pending.** Open the hub. Point out the counts, the store switch, and the RO confidence badges.
2. **Drawer.** Click an RO with comments. Show the Updates tab: post with an @mention, reply to it, like it, and point at the seen-by eye.
3. **Activity log.** Show the Reynolds dates (opened, closed) among the DealerConnect status changes and owner changes.
4. **LOP hold.** Use a Submitter or Manager account: flag an LOP issue and show the foreman's notice. Then release it to Not Submitted and mark it Transmitted. A Claim Owner appears from the rotation.
5. **Claim difference.** Show the totals card, the largest gaps first, and an overpaid row in amber. Open one, show the labour and parts breakdown, and sign it off with a note.
6. **Minor adjustment.** Show a small difference waiting for Accounting.
7. **Monday board.** Show the old conversation on an RO, then jump to its live row.
8. **Phone.** Open the hub on a phone to show the section strip and the fixed RO column.

### Questions you may get

| Question | Answer |
| --- | --- |
| Why is this RO missing? | It may be a body shop RO, a new vehicle prep claim, or not yet in DealerConnect or the Open RO report. Check the store switch and filters too. |
| Why does a row say "No RO"? | Neither Reynolds report matched it. Enter the RO by hand in the drawer. |
| How fresh is it? | DealerConnect and the Open RO report arrive several times a day. The board picks up new data within 5 minutes. |
| Who gets a new claim? | The rotation, at transmit or on first arrival, following the order in section 5. |
| Can Monday still be edited? | Monday is read-only from the hub. It refreshes every 3 hours while the team still uses it. |
| Why is a claim "Needs Resubmit"? | DealerConnect rejected it and nobody has taken it yet. |
| Why didn't someone get an e-mail? | They had the hub open within the last 3 minutes, so the bell reached them. Or they have no e-mail address on file. |
| Can we see the Stellantis rejection code? | Not yet. The DealerConnect feed doesn't carry codes. |
