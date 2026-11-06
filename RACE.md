# RACE.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 1 is written BEFORE you run your race test. A TA
may ask about it. Commit it on its own so the order is visible.

Keep it short and specific. Point at methods, call sites, and the invariant.

---

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

Everything your review turned up that touches shared state, with a one-line
verdict for each. Add or remove rows as needed.

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book`: `store.hasOverlap` (line 70), then `store.insert` (line 74) | **Explains the ticket.** Check-then-act across two separately locked calls: the lock is dropped between them, so two threads can both see "free" and both insert CONFIRMED. |
| 2 | `book` check vs `cancelAndPromote` (`InMemoryBookingStore:53`) | **Racy, not harmless, but not the ticket.** Same stale check: a promotion can land between `book`'s check and insert, giving two CONFIRMED. The ticket excludes it (no waitlisted booking in any pair), but a fix must cover it. |
| 3 | `InMemoryBookingStore.bookings` and its `synchronized` methods (`hasOverlap`, `insert`, `bookingsForRoom`, `cancelAndPromote`, `size`) | Safe per call: all access is under the store's lock, helpers `indexOf`/`overlapsConfirmed` run only inside `cancelAndPromote`, and the list never escapes. This atomicity does not extend across calls (see 1). |
| 4 | `BookingService.joinWaitlist` (insert at line 97) | Safe: one atomic insert, no prior check to go stale, and the WAITLISTED entry blocks nothing, so it cannot create a CONFIRMED booking. |
| 5 | `BookingService.cancelBooking` (lines 114-117) | Safe for bookings: cancel and promote are one locked step. A concurrent double cancel logs and counts `cancelled` twice because the outcome is not checked, which is cosmetic. |
| 6 | `BookingStats` plain `long` counters | Racy but harmless: `++` loses updates, as documented. Nothing in the service reads them, and the ticket's duplicates are in the booking data, not the dashboard. |
| 7 | `BookingService.sequence` (`AtomicLong`) | Safe: `incrementAndGet` gives unique ids within one service. Caveat: the counter is per service while the store can be shared, so two service instances would both issue `bk-1`. Not this ticket, whose ids differ. |
| 8 | `BookingService.catalog` (`HashMap`) | Safe: filled in the constructor, assigned to a `final` field, then only read. |
| 9 | `BookingLog` (static `TIMESTAMP`, `System.out`) | Safe: the formatter is immutable and each event is one `println`. Lines may appear slightly out of order, which is harmless. |
| 10 | `Booking`, `Room`, `BookingResult`, `BookingStatus` | Safe: immutable records and an enum; a status change creates a new value in the store under its lock. |
| 11 | Harness: `PausingStore` counter and latches; `StressRunner` `System.setOut` | Safe and not production code: atomic and latch-based; `setOut` runs on the main thread around the rounds. |

**Which one explains the ticket, and how you know.** Row 1, `book`. The
invariant is that a room has at most one CONFIRMED booking covering any minute,
and the ticket's violation is in the booking data, so the cause must be
something that writes CONFIRMED records. That removes most rows: counters, ids,
logs and the catalog never write a status, and `joinWaitlist` writes only
WAITLISTED. Only two sites create CONFIRMED:
`book` (`BookingService:73`) and promotion (`InMemoryBookingStore:71`). The
ticket removes promotion: neither booking in any pair was waitlisted, and the
log shows two `booking.confirmed` lines with no `booking.promoted`. So `book`
ran twice and both calls got `false` from `hasOverlap`. Run one after the other,
that is impossible: nothing cancelled the first booking, so the second check
would have seen it and rejected. So the second `hasOverlap` must have run before
the first `insert`, inside the gap between lines 70 and 74. `overlaps` is correct
for identical ranges and `hasOverlap` holds the lock, so the check itself is not
wrong; its answer goes stale. The ticket's other facts fit: confirmations 1 ms
apart, in the rush after a slot opens, no exceptions (both threads follow the
success path), two different customers. The suite stays green because no test
starts a thread, and laptops on a count of three cannot hit a gap that small.
The false claim is in `BookingService`'s javadoc: "Since the store is thread
safe, the operations built on top of it are too." Each call is atomic; the pair
is not.

### Predicted interleaving (write this before you run your race test)

Two threads, one row per step, in the form from lecture. Describe what each
thread does at that step and leave the other column empty. Add or remove rows
as your prediction needs.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `book("WEH-5409", 540, 600)` calls `hasOverlap`, call 1. The store is empty, so the real store computes `false` and releases its lock. | |
| t2 | `PausingStore` parks A after the delegate returned, holding `false` before `book` sees it. **The check and insert are separated from here until t6.** | |
| t3 | | Same `book` call. `hasOverlap`, call 2, passes straight through and returns `false`: nothing is stored yet. |
| t4 | | `nextId()` gives `bk-1`. `insert(bk-1, CONFIRMED)` runs and returns (void). |
| t5 | | Logs `booking.confirmed booking=bk-1`. `book` returns CONFIRMED. |
| t6 | The test calls `release()`. `hasOverlap` returns the stale `false` to `book`, although `bk-1` now holds the slot. | |
| t7 | `nextId()` gives `bk-2`. `insert(bk-2, CONFIRMED)` runs and returns (void). Nothing rechecks. | |
| t8 | Logs `booking.confirmed booking=bk-2`. `book` returns CONFIRMED. | |
| | **End state:** two CONFIRMED bookings, `bk-1` and `bk-2`, for WEH-5409 540..600. Invariant broken. | |

**Aim: `new PausingStore(real, "hasOverlap", 1)`.** Start A alone and wait on
`awaitPaused()`, so A is certain to be call 1. Only then start B and join it,
then `release()` and join A. B is call 2, and the gate is one shot, so B is
never held. That order leaves one possible schedule: B's check and insert both
fall inside A's gap.

The pause comes after the real `hasOverlap` runs, and that is what makes this
aim work. A already holds its `false`, so the answer is stale by the time A uses
it. A pause before the call would let A read the store after B's insert, get
`true`, and be rejected, which is the correct behaviour. The pause also sits in
the wrapper, outside the real store's `synchronized` method, so A holds no lock
while parked and B is not blocked.

**Expected log lines**, in this order, with no `booking.rejected`,
`booking.waitlisted` or `booking.promoted` line (timestamps will vary):

```
<timestamp> INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
<timestamp> INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
```

B logs first and takes `bk-1`, even though A entered `book` first, because
`nextId()` runs after the check.

**Expected failure.** Count the bookings in `real.bookingsForRoom("WEH-5409")`
that are CONFIRMED and overlap 540..600, then:

```java
assertEquals(1, confirmed);
```

```
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
```

If the run turned out different from your prediction, add one line on what
you missed. Leave the table as you first wrote it.

### Your race test

**How you aimed `PausingStore`, and the result.** Method name and call index, why
that pair pins the schedule you predicted, and the output of two separate
runs (assertion failure and log lines).

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green or red. Paste the result.

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.
