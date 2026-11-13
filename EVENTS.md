# EVENTS.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 2 should be written before you run the replay. That is the
part a TA will ask about.

Keep it short and specific. Point at operations, call sites, and what a caller
gets back.

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

One row per operation. Read the code, not the file
names. "As wired today" is what the code does now. "Your call" is what you
would do if this were your service. List up to eight operations, and remove
the rows you don't use.

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Validate, conflict-check, and save the booking (`BookingService.book`, `bookingService.ts:65-95`) | Synchronous, in `book()` | Synchronous | The caller's whole question is "do I have the room?", and only the store can answer it before we return. |
| 2 | Write the booking's row to the search index (`searchIndex.indexBooking`, `bookingService.ts:94`) | Synchronous, in `book()`, before the store write | Event: subscriber on `booking.confirmed` (and a remove on `booking.cancelled`) | The room-finder is a read model; booking never reads it, so its outage must not block a booking. |
| 3 | Audit record (`audit.record` in `book()`/`cancel()`) | Synchronous, right after the store write | Synchronous | It is the record of what the service did, local and cannot fail, so it belongs next to the write it describes. |
| 4 | Confirmation / cancellation email (`EmailConfirmationHandler`) | Event, on `booking.confirmed` and `booking.cancelled` | Event | The organizer can wait seconds for mail, and a relay outage must not undo a booking. |
| 5 | Receipt rendering (`ReceiptGenerator`) | Event, on `booking.confirmed` | Event | Downloaded later; keyed by booking id, so a late or repeated render is harmless. |
| 6 | Schedule / drop the reminder (`ReminderScheduler`) | Event, on `booking.confirmed` and `booking.cancelled` | Event | Due 30 minutes before the start, long after the request returns; keyed by booking id. |
| 7 | Operations dashboard numbers (`StatsUpdater`) | Event, on both booking events | Event | Recounts the store each time, so it only has to be eventually right and nobody waits on it. |
| 8 | Pay the refund (`RefundLedger` → `PaymentGateway.refund`) | Amount computed synchronously in `cancel()`; ledger write and payment on `refund.requested` | Event (amount stays synchronous) | The cancellation must stand even if the provider is down, but the handler must be safe to run twice (milestone 2). |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**First operation and your call.** Search indexing (row 2), moved to the bus.
Booking only needs the service's own list of bookings, not search. But today
the service updates search before saving the booking, so when search is down,
booking fails. If search were updated by an event afterward, a search outage
would just leave the room-finder page out of date until the updates are
replayed.

**Second operation and your call.** Paying the refund (row 8), kept on the bus.
Someone cancelling needs the room released right away, not the money. If the
payment happened during the cancel, a payment provider outage would block the
cancellation. On the bus, the cancel goes through and the failed payment can be
retried later, as long as a retry doesn't pay twice (milestone 2).

### The incident

**The call behind `TICKET.md`.** `src/service/bookingService.ts`, in
`BookingService.book`, line 94: `this.searchIndex.indexBooking(booking, room, now);`.
When search is down, this line throws, nothing catches it, and the booking
fails with a generic error before anything is saved. Cancellations kept working
because cancelling never touches search.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** Name what you expect
to change in the ledger, at the payment provider, and in the bus log. Write it
down, then run the demo.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

**Your prediction against the run.** Where they agree and where they do not.

### Your fix

**The diff.** Paste it.

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

**What stays the same across its three states.** Name the numbers you
compared.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

### Your recovery

**What, if anything, the failed delivery left undone.**

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.
