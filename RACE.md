# RACE.md

## Milestone 1: from the ticket to a test that fails every time

### Suspect list

| # | What you found (method, field, or class) | Verdict, in one line |
|---|---|---|
| 1 | `BookingService.book`: `store.hasOverlap` (line 70), then `store.insert` (line 74) | **Explains the ticket.** Check-then-act across two locked calls: two threads can both see "free" and both insert. |
| 2 | `book` check vs `cancelAndPromote` (`InMemoryBookingStore:53`) | **Racy, not harmless, not the ticket.** A promotion can land between check and insert; the ticket excludes it (no waitlisted booking in any pair). |
| 3 | `InMemoryBookingStore.bookings` and its `synchronized` methods (`hasOverlap`, `insert`, `bookingsForRoom`, `cancelAndPromote`, `size`) | Safe per call: all access is under one lock. That atomicity does not extend across calls. |
| 4 | `BookingService.joinWaitlist` (insert at line 97) | Safe: one atomic insert, and WAITLISTED blocks nothing. |
| 5 | `BookingService.cancelBooking` (lines 114-117) | Safe for bookings: one locked step. A double cancel logs and counts twice, which is cosmetic. |
| 6 | `BookingStats` plain `long` counters | Racy but harmless: lost updates are documented and nothing reads them. |
| 7 | `BookingService.sequence` (`AtomicLong`) | Safe within one service; two services on one store would both issue `bk-1`. Not this ticket. |
| 8 | `BookingService.catalog` (`HashMap`) | Safe: filled in the constructor, `final`, then only read. |
| 9 | `BookingLog` (static `TIMESTAMP`, `System.out`) | Safe: immutable formatter, one `println` per event. |
| 10 | `Booking`, `Room`, `BookingResult`, `BookingStatus` | Safe: immutable records and an enum. |
| 11 | Harness: `PausingStore` counter and latches; `StressRunner` `System.setOut` | Safe, and not production code. |

**Which one explains the ticket, and how you know.** Row 1, `book`.

1. The violation is two CONFIRMED records in the booking data, so only code that
   writes CONFIRMED can cause it. Counters, ids, logs and the catalog do not.
2. Only `book` (`BookingService:73`) and promotion (`InMemoryBookingStore:71`) do.
3. The ticket removes promotion: no booking in any pair was waitlisted, and the log
   has no `booking.promoted`. So `book` issued both.
4. In sequence, the second check would see the first booking. So the second
   `hasOverlap` ran before the first `insert`, inside the gap between them.

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

**How you aimed `PausingStore`, and the result.** `hasOverlap`, call 1. A starts
alone and the test asserts `awaitPaused()`, so call 1 is A's, and A parks after the
real call holding `false`. B then checks and inserts inside that gap. Red on the
invariant in both runs, 0.072 s and 0.061 s, far below the 2 s timeout.

```text
--- Run 1 ---
2026-11-06T19:47:17.329Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:47:17.338Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0 | BUILD FAILURE
--- Run 2 ---
2026-11-06T19:47:22.332Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:47:22.340Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0 | BUILD FAILURE
```

Prediction vs actual: matched in order, ids, outcomes and assertion text. Log lines
do not name threads, but A is held until after B is joined, so the first line is B's.
The prediction did not cover the gate assertion, added later, or the 8-9 ms log gap.

---

## Milestone 2: the patch a teammate sent

**Your milestone-1 test on the patched code.** Green, twice. The patch adds a second
`hasOverlap` just before the write, so `book()` calls `hasOverlap`, `hasOverlap`,
`insert`. The gate fired (`RaceTest.java:40` passed) and 1 CONFIRMED remained.

```text
--- Run 1 ---
2026-11-06T19:50:27.137Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:50:27.151Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
--- Run 2 ---
2026-11-06T19:50:32.078Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:50:32.094Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
```

Green proves one schedule is closed: B running entirely between A's two checks.
It does not prove the race is fixed: that aim parks the first check, and the second
check and the insert are still two locked calls, so a gap remains that it never hits.

**The re-aimed run.** `hasOverlap`, call 2. A starts alone and B starts only after
`awaitPaused()`, so calls 1-2 are A's and B's are 3-4. It tests the patch because it
pauses at the patch's own re-check. Predicted before the test existed: `bk-2`, then
`bk-1` (`nextId()` runs before the re-check), then `expected: <1> but was: <2>`. Matched.

```text
--- Run 1 ---
2026-11-06T19:52:53.670Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T19:52:53.670Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0 | BUILD FAILURE
--- Run 2 ---
2026-11-06T19:52:57.861Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T19:52:57.861Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
Tests run: 1, Failures: 1, Errors: 0, Skipped: 0 | BUILD FAILURE
```

**Your verdict.** **REJECT.** The re-aimed test reaches two CONFIRMED on every run.
The patch closes one schedule and leaves the gap between re-check and insert open.
The suite had no concurrent test, and stress depends on timing: on the patch, 1 bad
round in 4,300, against 21 in 300 before, so clean runs were expected. `book()` vs
`cancelAndPromote` stays open for the same reason (reproduced in a one-off check).

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Option 2: `BookingStore.insertIfFree`,
`synchronized` on the store's monitor, covers the overlap check and the insert as
one step. `book()` calls it once in place of `hasOverlap` then `insert`.

- **Why the store:** services may share a store, so a service lock would not stop
  them racing. One-off check: two services on one store left 1 CONFIRMED.
- **Promotion race closed:** `cancelAndPromote` uses the same lock, so it lands
  wholly before or after. One-off check: both orders left 1 CONFIRMED.
- **Behaviour change:** a rejected booking uses up an id, so ids can skip.

Code hunks, with javadoc omitted. Full diff: `git diff a8dc8f5 afa318b -- src/main`.

```diff
--- BookingService.book()
-        if (store.hasOverlap(roomId, startMinute, endMinute)) {
+        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
+        if (!store.insertIfFree(booking)) {
             return reject(roomId, startMinute, endMinute, "slot already held");
         }
-        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
-        store.insert(booking);
         stats.recordConfirmed();
--- BookingStore
+    boolean insertIfFree(Booking booking);
--- InMemoryBookingStore
+    @Override
+    public synchronized boolean insertIfFree(Booking booking) {
+        if (overlapsConfirmed(booking)) {
+            return false;
+        }
+        bookings.add(booking);
+        return true;
+    }
--- PausingStore
+    @Override
+    public boolean insertIfFree(Booking booking) {
+        return gate("insertIfFree", () -> delegate.insertIfFree(booking));
+    }
```

```text
--- twoCustomersBookingTheSameSlotLeaveOneConfirmed, Run 1 ---
2026-11-06T19:57:25.165Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:25.181Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
--- twoCustomersBookingTheSameSlotLeaveOneConfirmed, Run 2 ---
2026-11-06T19:57:29.289Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:29.306Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
--- aReCheckBeforeInsertStillLeavesOneConfirmed, Run 1 ---
2026-11-06T19:57:33.395Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:33.417Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
--- aReCheckBeforeInsertStillLeavesOneConfirmed, Run 2 ---
2026-11-06T19:57:37.489Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:37.506Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
--- Full suite ---
Tests run: 13, Failures: 0, Errors: 0, Skipped: 0 | BUILD SUCCESS
```

**Is your milestone-1 test still testing anything?** Yes, but not atomicity. `book()`
no longer calls `hasOverlap`, so both tests aim at `insertIfFree`, call 1, keeping
their names, schedule, gate and end-state assertions. They are now the same test
under two names. The pause fired in every green run: the gate assertions at
`RaceTest.java:40` and `:65` passed, and B's rejection logs first because A was parked.

It can show that a booker running while A is parked is rejected and 1 CONFIRMED
remains. It cannot show the check and insert are atomic: `PausingStore` parks after
the real call returns, so A has already finished the step. With `synchronized`
temporarily removed, both race tests stayed green, while all 3 stress runs of 2000
rounds threw `ConcurrentModificationException` and one reported 3 bad rounds (the other
counts were not captured). Atomicity rests on reading the `synchronized` method,
supported by 0 bad rounds in 4,300 on the fix.

**What runs one at a time now that ran in parallel before.** Check and insert are one
lock hold, so other callers can no longer run between them. A second `book()` waits
for the first's check and insert. Web, kiosk and other services on the store share
that line, and `cancelBooking`, `joinWaitlist` and `listBookings` queue behind it too.

**Cost:** all bookings on the store, for every room, serialize on one lock, so a
GHC-4301 booking waits behind a WEH-5409 one although they cannot conflict. Each hold
scans the whole list, which only grows. Not measured. It would matter when many
bookings hit a large list at once, as just after a slot opens; per-room locks would help.

**What you left unfixed.**

- `BookingStats` lost counts: documented as approximate, and nothing reads them.
- Per-service `sequence` ids: not this ticket; needs store-issued ids if services share.
- `cancelBooking` double logging: cosmetic, with no effect on booking data.
- `hasOverlap` and `insert` stay public: a future caller could rebuild the race.
