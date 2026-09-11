# RoomReserve Critique

---

## Milestone 1: The design as it is

**Data model.** A booking is not a class. It is two independently-keyed map entries in
`InMemoryStore`: an interval `long[]{start, end}` (minutes since midnight) in
`slotsByRoomDate: Map<String, List<long[]>>` (key `"room|date"`), and a booker name in a
separate `bookerBySlot: Map<String, String>` (key `"room|date|start|end"`)
(`InMemoryStore.java:13-14`). The two maps must stay in agreement on room/date/start/end,
but nothing enforces that beyond three independent string-concatenation call sites that
build the same key format (`addSlot` lines 17/29, `removeSlot` lines 34/42, `bookerFor`
line 58). A booking's only identity is its exact interval — there is no booking ID.

**Operations.** All four live on `RequestHandler`, taking and returning `String`:
`createBooking(room, date, start, end, user)` (`RequestHandler.java:12`),
`cancelBooking(room, date, start, end)` (`:45`),
`rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd)` (`:67`),
`listBookings(room, date)` (`:103`). `"HH:MM"` strings are parsed to minutes at the top of
each method.

**Structure.** `ReservationApp` (`ReservationApp.java`) is a demo `main` that calls
`RequestHandler` directly. `RequestHandler` (`RequestHandler.java:10`) holds one
`private final InMemoryStore store`, parses request strings, and — for `createBooking`
only — inlines its own overlap check before touching storage. `InMemoryStore`
(`InMemoryStore.java`) owns the two maps above and exposes
`addSlot`/`removeSlot`/`slotsFor`/`bookerFor`/`hasSlot`/`bookingCount`. `BookingPolicy`
(`BookingPolicy.java`) is fully implemented (business hours, max length, overlap, via
`validate()`) but is **never instantiated or called by any other class** — confirmed by
grepping all of `src/main` for `BookingPolicy`: every match is inside `BookingPolicy.java`
itself. This contradicts DESIGN.md's component table and flow diagram (`DESIGN.md:16-47`)
and its explicit claim that "every booking request is validated by `BookingPolicy` before
it reaches storage" (`DESIGN.md:51`). In the real code, `RequestHandler` talks to
`InMemoryStore` directly, and `BookingPolicy` is dead code.

**The no-double-booking invariant.** For a given `(room, date)`, no two stored intervals
may overlap. Every place this is actually checked:

- `RequestHandler.createBooking`, lines 30-36 — the only real overlap check in the live
  system. Walks `store.slotsFor(room, date)` and rejects when
  `startMinutes < slot[1] && slot[0] < endMinutes`, before calling `store.addSlot`. This
  reimplements — independently, without calling — `BookingPolicy.overlaps`
  (`BookingPolicy.java:45`).
- `InMemoryStore.addSlot`, lines 23-27 — checks only for an **exact duplicate interval**
  (`slot[0]==start && slot[1]==end`). A partially overlapping interval passes freely. Not
  an overlap check.
- `BookingPolicy.validate`, lines 19-35 — a correct overlap (+ hours + length) check, but
  dead code, as established above; not an enforcement point in the running system.
- `RequestHandler.rescheduleBooking`, lines 67-101 — **no overlap check at all.**

Reschedule trace, entry point to storage:

1. `RequestHandler.rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd)`
   (`:67`).
2. Parses all four `"HH:MM"` strings to minutes (`:69-88`); malformed input returns
   `"ERROR: time must look like HH:MM"`.
3. Checks only `newEndMinutes > newStartMinutes` (`:89`) — no business-hours, max-length,
   or overlap check.
4. `store.bookerFor(room, date, oldStartMinutes, oldEndMinutes)`
   (`InMemoryStore.java:57`) — exact-key lookup in `bookerBySlot`; `null` returns
   `"ERROR: no booking for ... at ..."` and stops.
5. `store.removeSlot(room, date, oldStartMinutes, oldEndMinutes)`
   (`InMemoryStore.java:33`) — removes the old interval from both maps unconditionally;
   the boolean return value is discarded (`RequestHandler.java:98`).
6. `store.addSlot(room, date, newStartMinutes, newEndMinutes, user)`
   (`InMemoryStore.java:16`) — only rejects an exact-duplicate new interval; otherwise
   appends to `slotsByRoomDate` and writes `bookerBySlot`. Return value also discarded
   (`:99`).
7. Returns `"OK: moved ..."` (`:100`) unconditionally, even if step 6 silently failed.

Net: `BookingPolicy` is never on this path, and the new interval is never checked against
any *other* booking for the room/date — reschedule can create an overlapping
double-booking with no rejection.

---

## Milestone 2: Two design problems

### Problem 1

**The problem.** Misplaced responsibility. DESIGN.md assigns "the rules" to
`BookingPolicy` as their single owner, but validation logic is actually reimplemented
inline, partially, at each call site instead.

**Where in the code.** `RequestHandler.createBooking` lines 30-36 inline an overlap-only
check (duplicating, not calling, `BookingPolicy.overlaps` at `BookingPolicy.java:45`) and
skip business-hours/max-length entirely. `RequestHandler.rescheduleBooking` lines 67-101
perform no rule check at all. `BookingPolicy.java` sits unused.

**What it makes expensive.** DESIGN.md's promise of "a single place to edit when the
rules change" (`DESIGN.md:59-61`) — e.g. the planned per-building closing time — is false
today: a rule change requires auditing every operation's method body for a hand-rolled
copy of the rule rather than editing one class. It has already caused a real bug: reschedule
enforces none of the three rules, including no-double-booking.

### Problem 2

**The problem.** Representational gap. There is no `Booking` type; a single logical
booking is split across two independently-keyed maps, identified only by its exact
interval.

**Where in the code.** `InMemoryStore.java:13-14` (`slotsByRoomDate`, `bookerBySlot`),
with the same `"room|date|start|end"` key format independently reconstructed by string
concatenation in `addSlot` (`:17,29`), `removeSlot` (`:34,42`), and `bookerFor` (`:58`).

**What it makes expensive.** Nothing enforces that the two maps stay in sync.
Concretely: `rescheduleBooking` discards both `removeSlot`'s and `addSlot`'s return values
(`RequestHandler.java:98-99`) — if `addSlot` fails (e.g. the new interval exactly matches
another existing booking), the old booking is already gone from both maps but no new one
was recorded, yet the handler still reports `"OK: moved ..."` and the user's booking
silently vanishes. The lack of a booking ID also means every operation must address a
booking by its exact original interval, which only gets harder once "recurring bookings"
(`DESIGN.md`'s planned next) need an identity beyond a single room+date+interval tuple.

---

## Milestone 3: Two alternative decompositions

### Alternative A

**The decomposition.** Keep the three-layer split, but make it real. `RequestHandler`
only parses/formats. A `Booking` value type (`room, date, start, end, user`) replaces the
two parallel maps. Every mutating operation (`create`, `cancel`, `reschedule`) is required
to pass through `BookingPolicy` — expanded into the sole validation gate, injected into
`RequestHandler`, with no other path to `InMemoryStore` — before storage is touched.
Reschedule's overlap check excludes the booking being moved.

**One tradeoff.** `cancelBooking` now pays for hours/length checks that never applied to
it, and "exclude the old booking from its own overlap check" adds a special case to what
DESIGN.md sold as "a single comparison." This is a discipline fix, not a rethink — on its
own it doesn't help with per-room-varying rules or recurring bookings.

### Alternative B

**The decomposition.** Aggregate-oriented. A `RoomCalendar` (one per room+date, or a
`Room` owning its calendars) owns its own `Booking`s and exposes `book`, `cancel`,
`reschedule` methods that enforce hours/length/overlap as *its own* invariant — the
aggregate cannot be mutated into an invalid state. `RequestHandler` becomes a thin adapter
that looks up the right aggregate and calls it; there is no free-standing policy class or
free-standing store — the rule logic and the state it protects live in the same object.

**One tradeoff.** The invariant becomes impossible to bypass by construction, which is
strictly stronger than Alternative A. But a rule that isn't about one room (e.g. a holiday
closure across all rooms) has no obvious owner, and unit-testing "the rules" in isolation
(a plain list of intervals in, a verdict out — what `BookingPolicy` already supports)
gets harder once the rules are entangled with aggregate construction and storage.

### Preference

Alternative A, given DESIGN.md's own "planned next" (per-building hours, recurring
bookings, a DB-backed store): keeping validation as one separate, swappable, easily
unit-tested class scales better to configurable, shared rules than pushing logic onto N
per-room aggregates. I would switch to Alternative B if the actual future need turns out
to be genuinely different logic per entity, not just different constants — e.g. some rooms
requiring an approval workflow and others not — where a shared policy class would
otherwise accumulate per-room conditionals instead of each room owning its own rule.
