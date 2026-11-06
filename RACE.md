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

`RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed` uses
`new PausingStore(real, "hasOverlap", 1)`. Call 1 is A's check: B does not start
until A has parked, and `assertTrue(gated.awaitPaused(), ...)` fails the test if
A never parks. The gate parks A after the real call, holding `false`. B then
checks and inserts inside that gap, and A inserts on the stale answer. Both runs
get past the parked check at `RaceTest.java:40` and are red on the invariant
assertion at `RaceTest.java:49`, with `Errors: 0`. The test took 0.072 s and
0.061 s, far below the 2 s `PausingStore` timeout, so no timeout path was taken.

To confirm the parked check can fail, I temporarily aimed at `hasOverlap` call 3,
which never happens. That run failed at line 40 with `customer A never parked in
hasOverlap call 1`, after the 2 s timeout, and never reached the invariant
assertion. The aim was then restored.

These two runs replace the ones in commit 5cfa649, which predate the parked
check.

#### Run 1

```text
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< edu.cmu.cs214:lab10-reservations >------------------
[INFO] Building lab10-reservations 1.0.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- resources:3.4.0:resources (default-resources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\main\resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ lab10-reservations ---
[INFO] Nothing to compile - all classes are up to date.
[INFO] 
[INFO] --- resources:3.4.0:testResources (default-testResources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\test\resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ lab10-reservations ---
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 3 source files with javac [debug release 21] to target\test-classes
[INFO] 
[INFO] --- surefire:3.2.5:test (default-test) @ lab10-reservations ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running edu.cmu.cs214.lab10.harness.RaceTest
2026-11-06T19:47:17.329Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:47:17.338Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.106 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed -- Time elapsed: 0.072 s <<< FAILURE!
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed(RaceTest.java:49)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Failures: 
[ERROR]   RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed:49 expected: <1> but was: <2>
[INFO] 
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.341 s
[INFO] Finished at: 2026-11-06T14:47:17-05:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:3.2.5:test (default-test) on project lab10-reservations: There are test failures.
[ERROR] 
[ERROR] Please refer to C:\Users\ziqix\downloads\f26-lab10\target\surefire-reports for the individual test results.
[ERROR] Please refer to dump files (if any exist) [date].dump, [date]-jvmRun[N].dump and [date].dumpstream.
[ERROR] -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
```

#### Run 2

```text
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------< edu.cmu.cs214:lab10-reservations >------------------
[INFO] Building lab10-reservations 1.0.0
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- resources:3.4.0:resources (default-resources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\main\resources
[INFO] 
[INFO] --- compiler:3.13.0:compile (default-compile) @ lab10-reservations ---
[INFO] Nothing to compile - all classes are up to date.
[INFO] 
[INFO] --- resources:3.4.0:testResources (default-testResources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\test\resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ lab10-reservations ---
[INFO] Nothing to compile - all classes are up to date.
[INFO] 
[INFO] --- surefire:3.2.5:test (default-test) @ lab10-reservations ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running edu.cmu.cs214.lab10.harness.RaceTest
2026-11-06T19:47:22.332Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:47:22.340Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.098 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed -- Time elapsed: 0.061 s <<< FAILURE!
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at edu.cmu.cs214.lab10.harness.RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed(RaceTest.java:49)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Failures: 
[ERROR]   RaceTest.twoCustomersBookingTheSameSlotLeaveOneConfirmed:49 expected: <1> but was: <2>
[INFO] 
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.491 s
[INFO] Finished at: 2026-11-06T14:47:22-05:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:3.2.5:test (default-test) on project lab10-reservations: There are test failures.
[ERROR] 
[ERROR] Please refer to C:\Users\ziqix\downloads\f26-lab10\target\surefire-reports for the individual test results.
[ERROR] Please refer to dump files (if any exist) [date].dump, [date]-jvmRun[N].dump and [date].dumpstream.
[ERROR] -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
```

#### Prediction vs what ran

Log lines, Run 1 and Run 2: `bk-1` confirmed first (thread B), then `bk-2`
(thread A), with no `booking.rejected`, `booking.waitlisted` or
`booking.promoted` line. This matches the prediction.

The log lines do not name threads. The test fixes the attribution: A is held
until `release()`, which runs only after `customerB.join`, and neither bounded
wait expired, so the first line must be B's and the second A's. Steps marked
"check" also rest on a throwaway program, not committed, that ran the same
schedule once on the same production code and printed each thread's state and
result.

| Step | Predicted | What ran |
|---|---|---|
| t1-t2 | A makes call 1 and parks holding `false` | Match: the test's parked check passed in both runs. Check: `awaitPaused=true`, `hasOverlapCalls=1`, A was `TIMED_WAITING`. |
| t3 | B's `hasOverlap` is call 2 and passes through | Match (check): `hasOverlapCalls=2`, and B was not held. |
| t4-t5 | B inserts and logs `bk-1` | Match: `bk-1` is logged first in both runs. Check: B finished in 32 ms with A still parked, and B's result was CONFIRMED `bk-1`. |
| t6-t8 | After `release()`, A inserts and logs `bk-2` on its stale `false` | Match: `bk-2` is logged second in both runs. Check: A's result was CONFIRMED `bk-2`. |
| Log lines | `bk-1` (B), then `bk-2` (A), nothing else | Match in Run 1 and Run 2. |
| Assertion | `expected: <1> but was: <2>` | Match in Run 1 and Run 2, exact text. |
| End state | Two CONFIRMED bookings | Match: the assertion counted 2. Check: the store held `bk-1` and `bk-2`, both CONFIRMED for 540..600. |

Differences: none in order, ids, outcomes or values. Two points the prediction
did not cover:

- The two log lines are 9 ms apart in Run 1 and 8 ms apart in Run 2, against 1 ms
  in the ticket. The prediction gave no gap. Here the gate holds A until B
  finishes, so this gap reflects the test's schedule, not production timing.
- The prediction did not include the parked check. I added it after the first
  two runs, because `awaitPaused()` could otherwise time out unnoticed.

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
