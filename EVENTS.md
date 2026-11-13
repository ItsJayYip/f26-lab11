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

Each redelivery repeats the refund: one more ledger entry and one more
settlement every time, so the organizer gets paid again. The bus log has no
new messages, but the refund message's delivery count goes up.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** Where they agree and where they do not.

They agree on everything. Each redelivery added a ledger entry (led-2, led-3)
and a settlement (stl-2, stl-3) for bk-1001, so the organizer got $255.00 back
on an $85.00 refund. The bus log has no new messages, msg-3 shows 3
deliveries, and there are no dead letters.

### Your fix

**The diff.** Paste it.

```diff
diff --git a/src/handlers/refundLedger.ts b/src/handlers/refundLedger.ts
index 600ffca..6d6d054 100644
--- a/src/handlers/refundLedger.ts
+++ b/src/handlers/refundLedger.ts
@@ -19,11 +19,20 @@ export interface LedgerEntry {
  * payment provider for the same amount. The ledger is append only: entries are
  * numbered in the order they were written, and nothing edits or removes one
  * after the fact.
+ *
+ * The bus can deliver the same message more than once, so the handler
+ * remembers message ids. A message gets at most one ledger entry and at most
+ * one payment. If the payment failed on an earlier delivery, a later delivery
+ * of the same message pays without writing a second entry.
  */
 export class RefundLedger implements Subscriber<RefundRequested> {
   readonly name = 'RefundLedger';
 
   private readonly entries: LedgerEntry[] = [];
+  /** Message ids that already have a ledger entry. */
+  private readonly recorded = new Set<string>();
+  /** Message ids whose payment went through. */
+  private readonly paid = new Set<string>();
   private readonly gateway: PaymentGateway;
 
   constructor(gateway: PaymentGateway) {
@@ -32,15 +41,22 @@ export class RefundLedger implements Subscriber<RefundRequested> {
 
   handle(message: Message<RefundRequested>): void {
     const event = message.payload;
-    this.entries.push({
-      entryId: `led-${this.entries.length + 1}`,
-      bookingId: event.bookingId,
-      organizer: event.organizer,
-      amountCents: event.amountCents,
-      reason: event.reason,
-      recordedAtMinute: message.publishedAtMinute,
-    });
+    if (this.paid.has(message.id)) {
+      return;
+    }
+    if (!this.recorded.has(message.id)) {
+      this.entries.push({
+        entryId: `led-${this.entries.length + 1}`,
+        bookingId: event.bookingId,
+        organizer: event.organizer,
+        amountCents: event.amountCents,
+        reason: event.reason,
+        recordedAtMinute: message.publishedAtMinute,
+      });
+      this.recorded.add(message.id);
+    }
     this.gateway.refund(event.bookingId, event.amountCents);
+    this.paid.add(message.id);
   }
 
   /** Every ledger entry, oldest first. */
```

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

It uses the message id: an already-paid message is skipped, and one whose
payment failed gets paid without a second ledger entry. It does not protect
against the same refund sent as a new message (new id), or against the
remembered ids being lost on a restart.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**What stays the same across its three states.** Name the numbers you
compared.

In all three states: 1 ledger entry (led-1, $85.00), 1 gateway call, 1
settlement (stl-1, $85.00), $85.00 returned in total, and 0 dead letters. Only
the delivery count for msg-3 goes up, to 3.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

```
armed: call 1 to the payment provider will throw
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3
the cancel call itself returned: booking is cancelled

=== after the failed delivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 0
money returned in total: $0.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=1

the refund message is in the log and in the dead-letter list. Milestone 3 asks what you do with it.
```

### Your recovery

**What, if anything, the failed delivery left undone.**

The organizer was never paid. The booking is cancelled and the ledger says
$85.00 was refunded (led-1), but the payment provider has no settlement and $0.00
went back. The refund message msg-3 is on the dead-letter list.

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

I redelivered msg-3 by adding `app.bus.redeliver` at the end of `main()` in
`src/demos/failureDemo.ts`. The redelivery saw the ledger entry already existed,
so it only made the payment. The ledger still has one entry, there is now one
$85.00 settlement, and the old dead letter stays on the list as a record of the
failed attempt.

State after the redelivery:

```
=== after the redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}
```

Bus log line for the refund message:

```
  msg-3 refund.requested deliveries=2
```

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.

Leaving the message on the dead-letter list for a person to handle would mean
someone has to read it and pay the refund by hand, so the organizer waits until someone gets to it, and it is only safe if
that person checks that the payment never went through so they don't pay twice.
