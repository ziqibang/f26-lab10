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

**Green, twice.** Same test, same aim (`hasOverlap`, call 1), with
`proposed-fix.diff` applied through `git apply` and not committed. The gate fired
in both runs: the parked check at `RaceTest.java:40` passed. The invariant held
for this schedule: `assertEquals(1, ...)` passed with one CONFIRMED booking.

After the patch `book()` calls `hasOverlap`, then `hasOverlap` again, then
`insert`. The patch adds the second check and changes nothing else.

#### Patched Run 1

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
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 11 source files with javac [debug release 21] to target\classes
[INFO] 
[INFO] --- resources:3.4.0:testResources (default-testResources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\test\resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ lab10-reservations ---
[INFO] Recompiling the module because of changed dependency.
[INFO] Compiling 3 source files with javac [debug release 21] to target\test-classes
[INFO] 
[INFO] --- surefire:3.2.5:test (default-test) @ lab10-reservations ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running edu.cmu.cs214.lab10.harness.RaceTest
2026-11-06T19:50:27.137Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:50:27.151Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.080 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.579 s
[INFO] Finished at: 2026-11-06T14:50:27-05:00
[INFO] ------------------------------------------------------------------------
```

#### Patched Run 2

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
2026-11-06T19:50:32.078Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
2026-11-06T19:50:32.094Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.107 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.819 s
[INFO] Finished at: 2026-11-06T14:50:32-05:00
[INFO] ------------------------------------------------------------------------
```

#### What this result does and does not prove

Both runs log `bk-1` confirmed, then one `booking.rejected`. A is parked until B
has finished, so the confirmation is B's and the rejection is A's. A's stale
`false` from call 1 still reached `book`, but A's new second check saw `bk-1` and
rejected.

**It proves** that one interleaving is closed: B running completely between A's
first check and A's second check no longer double-books.

**It does not prove the race is fixed.** The aim parks the first check, whose
answer no longer leads straight to the insert. The patch adds a check but does
not make the last check and the insert one atomic step: `hasOverlap` and `insert`
are still separately locked calls, so a gap remains after the second check. This
test cannot see that gap, because it never pauses there.

The other green results do not settle it either. The full suite passing (12 of
12) only adds this one schedule to tests that run one caller at a time. The
teammate's clean stress runs cannot show the gap is gone, since a shorter gap
would also fail less often. Stress results on the patch are in the verdict.

The gap after the second check is exercised in the re-aimed run below.

**The re-aimed run.** Method name, call index, the pasted result, and one line
on why that aim tests the patch rather than sidestepping it.

#### Prediction for the re-aimed run (written before the test existed)

Aim: `new PausingStore(real, "hasOverlap", 2)`, on patched code. Both threads call
`book("WEH-5409", 540, 600)`.

| Step | Thread A | Thread B |
|---|---|---|
| t1 | `hasOverlap`, call 1, returns `false`. `nextId()` gives `bk-1`. | |
| t2 | `hasOverlap`, call 2 (the re-check), computes `false`. The gate parks A holding it. **The re-check and insert are separated from here until t6.** | |
| t3 | | `hasOverlap`, call 3, returns `false`. `nextId()` gives `bk-2`. |
| t4 | | `hasOverlap`, call 4, returns `false`. `insert(bk-2, CONFIRMED)` runs. |
| t5 | | Logs `booking.confirmed booking=bk-2`. `book` returns CONFIRMED. |
| t6 | The test calls `release()`. The re-check returns the stale `false`, although `bk-2` holds the slot. | |
| t7 | `insert(bk-1, CONFIRMED)` runs. Nothing rechecks. | |
| t8 | Logs `booking.confirmed booking=bk-1`. `book` returns CONFIRMED. | |
| | **End state:** two CONFIRMED bookings, `bk-1` and `bk-2`, for WEH-5409 540..600. | |

Why call 2 is A's re-check and not one of B's calls: the count runs across all
threads, but B does not exist yet. A starts alone and the test blocks on
`awaitPaused()` until the gate fires. So calls 1 and 2 can only be A's, and B's
two checks are calls 3 and 4. The gate is one shot, so B passes through.

Expected log lines, in this order, with no `booking.rejected`,
`booking.waitlisted` or `booking.promoted` line (timestamps will vary):

```
<timestamp> INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
<timestamp> INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
```

The ids are the reverse of milestone 1: `nextId()` runs before the re-check, so A
takes `bk-1` before parking, yet B logs first.

Expected result: the parked check passes, then the end-state assertion fails with
`expected: <1> but was: <2>`.

#### Re-aimed result

`RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed`, aim `hasOverlap` call 2,
on patched code: **red, twice.** The parked check passed, so A held its re-check
answer, and the end-state assertion failed at `RaceTest.java:75` with
`expected: <1> but was: <2>` and `Errors: 0`. The test took 0.054 s and 0.039 s,
so no timeout path was taken.

This tests the patch rather than sidestepping it because it pauses at the patch's
own re-check, the last read before the insert, and lets that answer go stale.

The run matched the prediction, including the order: `bk-2` (B) logs first and
`bk-1` (A) second, with no rejected, waitlisted or promoted line. One point the
prediction did not cover: in both runs the two lines carry the same millisecond,
closer to the ticket's 1 ms gap than milestone 1's 8-9 ms.

#### Re-aimed Run 1

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
2026-11-06T19:52:53.670Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T19:52:53.670Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.081 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed -- Time elapsed: 0.054 s <<< FAILURE!
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at edu.cmu.cs214.lab10.harness.RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed(RaceTest.java:75)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Failures: 
[ERROR]   RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed:75 expected: <1> but was: <2>
[INFO] 
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.316 s
[INFO] Finished at: 2026-11-06T14:52:53-05:00
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

#### Re-aimed Run 2

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
2026-11-06T19:52:57.861Z INFO booking.confirmed booking=bk-2 room=WEH-5409 start=540 end=600
2026-11-06T19:52:57.861Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0, Time elapsed: 0.079 s <<< FAILURE! -- in edu.cmu.cs214.lab10.harness.RaceTest
[ERROR] edu.cmu.cs214.lab10.harness.RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed -- Time elapsed: 0.039 s <<< FAILURE!
org.opentest4j.AssertionFailedError: expected: <1> but was: <2>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at edu.cmu.cs214.lab10.harness.RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed(RaceTest.java:75)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Failures: 
[ERROR]   RaceTest.aReCheckBeforeInsertStillLeavesOneConfirmed:75 expected: <1> but was: <2>
[INFO] 
[ERROR] Tests run: 1, Failures: 1, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.070 s
[INFO] Finished at: 2026-11-06T14:52:57-05:00
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

**Your verdict.** Accept or reject, and what your milestone-1 result does and
does not show about the patch.

**REJECT.** The re-aimed test reaches two CONFIRMED bookings for the same room and
time on every run, so the patch does not fix RES-4471.

- **What it closes:** one schedule. B running completely between A's first check
  and A's re-check is now caught, which is why the milestone-1 test goes green.
  That result shows only this schedule is closed.
- **What it leaves open:** the gap between the re-check and the insert.
  `hasOverlap` and `insert` are still two separately locked calls, so any answer
  can go stale before the write. Adding a read makes the gap shorter, and no number
  of reads removes it. The patch's comment, "no window left", is wrong.
- **Why the teammate's evidence missed it:** the suite they ran had no concurrent
  test at all, so it could not see any race. Stress runs depend on timing, and the
  patch shrinks the gap, so bad rounds become rare rather than impossible. On
  patched code, 3 runs of 100 rounds and 2 of 2000 gave 4 clean runs and 1 bad round
  in 4,300, against 21 in 300 before the patch. Clean runs are what you would expect
  to see from a patch that does not work.
- **`book()` vs `cancelAndPromote`:** still open for the same reason. Nothing
  makes `book()`'s last check and its insert one step, so a promotion can land
  between them. In a one-off check, not committed, with X holding 600..660 and W
  waitlisted for 540..660, I parked `book(540, 600)` at call 2 and cancelled X. W
  was promoted, A then inserted `bk-1`, and both were CONFIRMED over 540..599.

A real fix has to make the check and the insert one atomic step, covering every
writer of CONFIRMED, not add more reads.

---

## Milestone 3: the fix you would ship

**Which option, and what you changed.** Name the lock or the atomic operation
and the two operations it now covers. Paste the diff of your change and the
passing run of your milestone-1 test and the full suite.

**Option 2: an atomic check-and-insert in the store.** `BookingStore.insertIfFree`
is a new plain interface method, and `InMemoryBookingStore` implements it as one
`synchronized` method. The lock is that store's monitor, the same lock as every
other store method, and it now covers the two operations together: the overlap
check and the insert. `book()` calls `insertIfFree` once instead of `hasOverlap`
then `insert`. `PausingStore` passes it through its usual gate.

Why the store and not a lock in the service:

- **Shared store.** `BookingService`'s constructor says a store may be shared by
  several service instances. A lock on a service would not stop two services
  racing. The store's lock does: in a one-off check, not committed, two services
  on one store under the same schedule left 1 CONFIRMED.
- **Promotion race closed too.** `cancelAndPromote` already runs under this lock,
  so a promotion can no longer land between `book`'s check and insert. It happens
  wholly before or wholly after, and each side sees the other. In the same one-off
  check, with X holding 600..660 and W waitlisted for 540..660: cancelling X after
  `book(540, 600)` left W waiting, and cancelling X first promoted W and rejected
  the booking. Each left 1 CONFIRMED over those minutes.

The check reuses `overlapsConfirmed`, the rule promotion already uses. Building the
interface change alone failed on both implementers, `PausingStore` and
`InMemoryBookingStore`, as a plain method should.

Behaviour change: `nextId()` now runs before the store call, so a booking rejected
for a held slot uses up an id. Ids can now skip, but stay unique. `RaceTest` shows
this: A is `bk-1` and B's rejected attempt used `bk-2`. Confirmed and rejected logs
and results are otherwise unchanged.

#### `git diff` of `src/main` against the initial commit

```diff
diff --git a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
index 5e6b639..9c65544 100644
--- a/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/harness/PausingStore.java
@@ -53,8 +53,8 @@ public final class PausingStore implements BookingStore {
     /**
      * @param delegate     the real store, which does all the work
      * @param targetMethod the {@link BookingStore} method to park inside, by
-     *     name: {@code hasOverlap}, {@code insert}, {@code bookingsForRoom}, or
-     *     {@code cancelAndPromote}
+     *     name: {@code hasOverlap}, {@code insert}, {@code insertIfFree},
+     *     {@code bookingsForRoom}, or {@code cancelAndPromote}
      * @param targetCall   which invocation of that method to park, counting
      *     from 1 across all threads
      */
@@ -109,6 +109,11 @@ public final class PausingStore implements BookingStore {
         });
     }
 
+    @Override
+    public boolean insertIfFree(Booking booking) {
+        return gate("insertIfFree", () -> delegate.insertIfFree(booking));
+    }
+
     @Override
     public List<Booking> bookingsForRoom(String roomId) {
         return gate("bookingsForRoom", () -> delegate.bookingsForRoom(roomId));
diff --git a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
index 514c3e1..77c4a9d 100644
--- a/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
+++ b/src/main/java/edu/cmu/cs214/lab10/service/BookingService.java
@@ -22,10 +22,12 @@ import java.util.concurrent.atomic.AtomicLong;
  * <p>Thread safety: safe for concurrent use. The service owns no mutable
  * booking state of its own. The room catalog is built once in the constructor
  * and never written again, and every read and write of booking data goes
- * through {@link BookingStore}, whose methods are each synchronized. Since the
- * store is thread safe, the operations built on top of it are too. The only
- * other field that changes after construction is {@link BookingStats}, and its
- * numbers are documented as approximate.
+ * through {@link BookingStore}, whose methods are each synchronized. Each store
+ * call is atomic but a sequence of calls is not, so a decision that must hold at
+ * the moment of a write has to be one store call, as {@link #book} does with
+ * {@link BookingStore#insertIfFree}. The only other field that changes after
+ * construction is {@link BookingStats}, and its numbers are documented as
+ * approximate.
  */
 public final class BookingService {
 
@@ -67,11 +69,10 @@ public final class BookingService {
         if (startMinute < 0 || endMinute <= startMinute) {
             return reject(roomId, startMinute, endMinute, "empty or reversed slot");
         }
-        if (store.hasOverlap(roomId, startMinute, endMinute)) {
+        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
+        if (!store.insertIfFree(booking)) {
             return reject(roomId, startMinute, endMinute, "slot already held");
         }
-        Booking booking = new Booking(nextId(), roomId, startMinute, endMinute, BookingStatus.CONFIRMED);
-        store.insert(booking);
         stats.recordConfirmed();
         BookingLog.confirmed(booking);
         return BookingResult.confirmed(booking);
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
index 3ae3544..bc5aedb 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/BookingStore.java
@@ -28,6 +28,16 @@ public interface BookingStore {
     /** Stores the booking. Ids are assumed unique; the store does not check. */
     void insert(Booking booking);
 
+    /**
+     * Stores the booking only if the room has no CONFIRMED booking touching its
+     * range, as one indivisible step: no other call can change the room between
+     * the check and the insert.
+     *
+     * @param booking the booking to store
+     * @return true when the booking was stored, false when the range was taken
+     */
+    boolean insertIfFree(Booking booking);
+
     /**
      * Every booking held for the room, in insertion order, in every status.
      *
diff --git a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
index f7fd85f..d7c8833 100644
--- a/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
+++ b/src/main/java/edu/cmu/cs214/lab10/store/InMemoryBookingStore.java
@@ -36,6 +36,15 @@ public final class InMemoryBookingStore implements BookingStore {
         bookings.add(booking);
     }
 
+    @Override
+    public synchronized boolean insertIfFree(Booking booking) {
+        if (overlapsConfirmed(booking)) {
+            return false;
+        }
+        bookings.add(booking);
+        return true;
+    }
+
     @Override
     public synchronized List<Booking> bookingsForRoom(String roomId) {
         List<Booking> forRoom = new ArrayList<>();
```

#### `twoCustomersBookingTheSameSlotLeaveOneConfirmed` Run 1

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
[INFO] Recompiling the module because of changed source code.
[INFO] Compiling 11 source files with javac [debug release 21] to target\classes
[INFO] 
[INFO] --- resources:3.4.0:testResources (default-testResources) @ lab10-reservations ---
[INFO] skip non existing resourceDirectory C:\Users\ziqix\downloads\f26-lab10\src\test\resources
[INFO] 
[INFO] --- compiler:3.13.0:testCompile (default-testCompile) @ lab10-reservations ---
[INFO] Recompiling the module because of changed dependency.
[INFO] Compiling 3 source files with javac [debug release 21] to target\test-classes
[INFO] 
[INFO] --- surefire:3.2.5:test (default-test) @ lab10-reservations ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running edu.cmu.cs214.lab10.harness.RaceTest
2026-11-06T19:57:25.165Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:25.181Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.079 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  3.187 s
[INFO] Finished at: 2026-11-06T14:57:25-05:00
[INFO] ------------------------------------------------------------------------
```

#### `twoCustomersBookingTheSameSlotLeaveOneConfirmed` Run 2

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
2026-11-06T19:57:29.289Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:29.306Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.078 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.076 s
[INFO] Finished at: 2026-11-06T14:57:29-05:00
[INFO] ------------------------------------------------------------------------
```

#### `aReCheckBeforeInsertStillLeavesOneConfirmed` Run 1

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
2026-11-06T19:57:33.395Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:33.417Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.083 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.107 s
[INFO] Finished at: 2026-11-06T14:57:33-05:00
[INFO] ------------------------------------------------------------------------
```

#### `aReCheckBeforeInsertStillLeavesOneConfirmed` Run 2

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
2026-11-06T19:57:37.489Z INFO booking.rejected room=WEH-5409 start=540 end=600 reason=slot already held
2026-11-06T19:57:37.506Z INFO booking.confirmed booking=bk-1 room=WEH-5409 start=540 end=600
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.086 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.067 s
[INFO] Finished at: 2026-11-06T14:57:37-05:00
[INFO] ------------------------------------------------------------------------
```

#### Full suite

```text
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.051 s -- in edu.cmu.cs214.lab10.domain.BookingTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.076 s -- in edu.cmu.cs214.lab10.harness.RaceTest
[INFO] Tests run: 7, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.016 s -- in edu.cmu.cs214.lab10.service.BookingServiceTest
[INFO] Tests run: 13, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

**Is your milestone-1 test still testing anything?** After your fix, describe
what that test now exercises. Did `PausingStore` pause a call during the green run? How do
you know?

**Yes, but less than before, and not atomicity.**

How the tests changed: `book()` no longer calls `hasOverlap`, so the old aims
would never fire. Both tests now aim at `insertIfFree`, call 1, and keep their
names, the two-thread schedule, the gate assertion and the end-state assertion
(exactly 1 CONFIRMED). With the same aim they are now the same test under two
names. Milestone 2's target, a re-check, no longer exists.

Did it pause? Yes, in every green run. The gate assertions at `RaceTest.java:40`
and `:65` passed, which needs A to have parked. The logs agree: B's
`booking.rejected` comes before A's `booking.confirmed` although A's insert came
first, because A was parked before logging.

What the pause can show now: while A is parked, holding no lock, a second booker
runs freely, sees A's booking and is rejected, and the end state has 1 CONFIRMED.

What it cannot show: that the check and insert are atomic. `PausingStore` parks
after the real call returns, so A has finished the whole atomic step before the
pause. No wrapper can reach inside it. I confirmed this by temporarily removing
`synchronized` from `insertIfFree`: **both race tests stayed green.** The stress
runner did notice: all 3 runs of 2000 rounds threw
`ConcurrentModificationException`, and one reported 3 bad rounds. (My output filter
cut off the counts for the other two.)

So atomicity rests on the method being `synchronized`, which is checked by reading
it. The stress results support that: 3 runs of 100 rounds and 2 of 2000 on the fix
gave 0 bad rounds in 4,300, and that runner caught the lock's removal. As milestone 2
showed, though, clean stress runs alone prove nothing.

**What runs one at a time now that ran in parallel before.** Name concretely which callers
queue behind which. "Nothing, it is just correct now" does not count as an
answer.

Before, a booker held the store's lock twice, for the check and then for the
insert, and other callers could run in between. Now the check and insert are one
hold. Concretely:

- Two `book()` calls for the same slot queue: B's check waits until A's check and
  insert are both done, so B sees A's booking.
- Web and kiosk callers, and any other service instance on the same store, queue
  in the same line, because the lock belongs to the store.
- `cancelBooking` and `book` queue behind each other, as whole steps.
- `joinWaitlist` and `listBookings` wait while a `book` holds the lock.

**Cost.** All bookings on the store, for every room, serialize on one lock. A
booking for GHC-4301 waits behind one for WEH-5409, although they cannot conflict.
Each hold also scans the whole list, every room and every status, and the list only
grows because cancelled bookings are kept. So the hold gets longer as data builds
up, and bookings run one at a time however many cores the server has. This is not a
new lock, and the per-booking work is about the same as before, but the check and
insert no longer leave a gap for others to run in. I did not measure it.

It would matter when many bookings arrive together against a large list, such as
the rush just after a slot opens, which is when this bug appeared. For the campus
volumes in the ticket it should be negligible. If it ever showed up, the next steps
would be indexing by room or one lock per room.

**What you left unfixed.** For anything your review flagged that you chose not
to fix, one line each on why.

- **`BookingStats` lost counts:** left as is. Documented as approximate, nothing
  reads them, and exact counting is not worth the contention. Counting the store
  gives exact numbers.
- **Per-service `sequence` ids:** left as is. Each instance is unique on its own,
  but two services sharing a store can both issue `bk-1`, and the store does not
  check. It is not this ticket. It needs its own fix if several instances really
  do share a store, for example ids issued by the store.
- **`cancelBooking` double logging:** left as is. A repeat or unknown cancel still
  logs and counts `cancelled`. This is cosmetic and does not touch booking data.
- **`hasOverlap` and `insert` stay public:** no production code calls `hasOverlap`
  now, and `insert` is used only by `joinWaitlist`. Nothing stops a future caller
  rebuilding the race or inserting CONFIRMED directly, so this is worth a cleanup.
