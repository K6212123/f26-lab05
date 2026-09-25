# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplication over reuse. The complete pricing policy--including the
premium surcharge, long-booking discount, evening discount, thresholds, and order of
rounding--is implemented twice.

**Classic or agent-specific.** Agent-specific. This matches the lecture's "duplication
over reuse" smell. The likely cause is missing context: the author of the reporting code
rebuilt pricing without reusing either `ReservationManager.calculatePrice` or the price
already recorded on each `Booking`.

**Where in the code.** `src/reportGenerator.ts`, `ReportGenerator.priceOf` (lines
104--117), duplicates `ReservationManager.calculatePrice`/`applyDiscounts` and their
constants in `src/reservationManager.ts` (lines 11--15 and 140--158).

**The principle it violates.** Information hiding and a single source of truth. A pricing
policy should be owned behind one interface; reporting should consume the booking's
authoritative price rather than know every pricing rule.

**What it makes expensive.** Any pricing change, such as moving the evening cutoff or
adding a weekend discount, now requires coordinated edits in two files. If the reporting
copy is missed, `revenue()` is the first thing to break: its totals disagree with the
`priceCents` charged and printed on receipts. It can also retroactively reprice old
bookings under today's rules instead of reporting their stored price.

### Smell 2

**The smell.** Speculative over-abstraction. The notification code has a plugin registry,
builder type, registration and discovery functions, a factory, and configuration for a
system whose only supported channel is email.

**Classic or agent-specific.** Agent-specific. This is the lecture's "speculative
over-abstraction" smell, likely produced by an underspecified request: without an actual
requirement for multiple notification plugins, the author guessed at future generality.

**Where in the code.** `src/notifications/notifierFactory.ts`, especially
`registerChannel`, `registeredChannels`, and `createNotificationChannel`. `ChannelName`
is only `'email'`, and the only registration is the module-level email registration on
line 42.

**The principle it violates.** YAGNI and "decouple what varies." No demonstrated channel
variation justifies paying for a registry; the existing `NotificationChannel` interface
is already the useful boundary.

**What it makes expensive.** A simple change to notification configuration has to be
understood and threaded through the shared config, registry, builder, and factory.
Adding a real second channel is not plug-and-play either: it still requires editing the
closed `ChannelName` union and the common `NotifierConfig`, so the speculative machinery
adds review and test surface without localizing that change.

### Smell 3

**The smell.** Primitive obsession, specifically a data clump. A time interval is
represented everywhere by two raw numbers, and functions repeatedly pass separate
`start` and `end` values instead of using a domain type such as `TimeSlot`.

**Classic or agent-specific.** Classic. This is the lecture's primitive-obsession
example: related primitive values travel together even though they have rules and
behavior that belong on a real type.

**Where in the code.** `src/availability.ts`, especially `isSlotFree`, whose four
parameters are `slotStart`, `slotEnd`, `bookingStart`, and `bookingEnd` (lines 7--16).
The same start/end pair also appears in `findFreeSlots`, `freeMinutes`, `Booking`, and
`ReservationRequest`.

**The principle it violates.** Information hiding and making invalid states
unrepresentable. A `TimeSlot` should own the invariant that its end is after its start
and the behavior for overlap and clipping; callers should not reconstruct those rules
from unrelated numbers.

**What it makes expensive.** Adding a rule that applies to every interval--for example a
cleaning gap between bookings, multi-day reservations, or a different time unit--requires
finding and changing every function that manipulates the two numbers. Today it is also
possible to call `isSlotFree(660, 540, 540, 660)` with a reversed slot or accidentally
swap booking and candidate values, and the type checker cannot detect either mistake.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Duplication over reuse in revenue reporting. I chose it
because pricing is a business rule with an existing authoritative result on `Booking`,
and leaving a second implementation creates a direct risk that reports disagree with
what customers were charged.

**What changed.** In `src/reportGenerator.ts`, `ReportGenerator.revenue` now reads
`booking.priceCents` instead of calling its own `priceOf` implementation. I removed that
private method, its `durationOf` helper, the five duplicated pricing constants, and the
now-unused `Booking` import. Revenue totals now report the price stored when the booking
was created rather than independently recalculating it.

**What you deliberately did not touch.** My scope line was the reporting copy of the
pricing policy: replace its source of price and delete only code made dead by that
replacement. I did not redesign `ReservationManager.calculatePrice`, introduce a new
pricing service, change occupancy/window filtering, or address the notification and
time-primitive smells. Those changes are not required to establish one authoritative
price and would enlarge a behavior-preserving fix.

**How you know behavior is preserved.** All 39 existing tests pass and `npm run
typecheck` passes, with zero test edits. The suite covers base, premium, long-booking,
and evening prices; revenue totals, averages, per-room totals, and exclusion of cancelled
bookings; and occupancy behavior. The reporting test also compares revenue with the
created bookings' `priceCents`. It would not catch a future pricing rule that computes
the wrong value consistently before storing it, nor reporting behavior for malformed
bookings inserted directly into storage with an invalid `priceCents`.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Name it.

**The decomposition.** What are the pieces, what does each own, and where do the rules live?

**One cost.** Something this actually costs. "No real downside" is not a cost.

### Proposal B (not coded)

**The problem.**

**The decomposition.**

**One cost.**

### The thing that looks smelly but is fine

**What it is.** File and method.

**Why it is fine.** Defend it with properties of the code, not with its line count.

**What would flip your verdict.** Name the change that would turn this into a real problem.
