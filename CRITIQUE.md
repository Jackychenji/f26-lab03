# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

Line numbers below refer to the starter commit. Behaviors marked "(observed)" were
checked by running the compiled classes from `jshell`; the repository code was not
changed.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** There is no `Booking` type. A booking is a `long[2]` (`{startMinute,
endMinute}`) living in `InMemoryStore.slotsByRoomDate`, a `Map<String, List<long[]>>`
keyed by the string `room + "|" + date` (`InMemoryStore.java:13,17`). The booker is
*not* in that array: it lives in a second map, `bookerBySlot: Map<String, String>`,
keyed by `room + "|" + date + "|" + start + "|" + end` (`InMemoryStore.java:14,29`).
So one conceptual booking = one array in the first map + one entry in the second map,
glued together by rebuilding the same string key. That key is rebuilt by hand in five
methods (`addSlot`, `removeSlot`, `slotsFor`, `bookerFor`, `hasSlot`).

A booking's identity is its exact `(room, date, start, end)` tuple; there is no id. Two
bookings with the same interval cannot coexist (`addSlot:23-27`), and cancel/reschedule
find a booking only by its exact interval, not by who made it.

What has to stay in agreement: (1) every `long[]` in `slotsByRoomDate` has exactly one
matching key in `bookerBySlot`, and vice versa; (2) the key string is built identically
at every site; (3) the two maps are updated together. Nothing enforces (1): `slotsFor`
returns the live internal list (`InMemoryStore.java:54`), so a caller can append to it
and create a slot with no booker (observed: after
`store.slotsFor(r,d).add(new long[]{600,660})`, `bookerFor` returns `null` and
`bookingCount()` reports 2).

`room` and `date` are unparsed strings; `"not-a-date"` is accepted as a date
(observed). Times arrive as `"HH:MM"` and are parsed inside each handler method into
minutes since midnight; hour and minute are not range-checked (`25:00` parses to 1500).

**Operations.** Four public methods on `RequestHandler`, all `String` in / `String`
out, with the only protocol being an `"OK: "` or `"ERROR: "` prefix:

| Method | In | Out |
| --- | --- | --- |
| `createBooking(room, date, start, end, user)` | five strings | `OK: booked ...` or an `ERROR:` (bad time format, end ≤ start, overlap, exact duplicate) |
| `cancelBooking(room, date, start, end)` | four strings, **no user** | `OK: cancelled ...` or `ERROR: no booking ...`. Anyone can cancel anything. |
| `rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd)` | six strings, **no user** | `OK: moved ...` or `ERROR:` (bad format, end ≤ start, no such booking). Booker is carried over by lookup. |
| `listBookings(room, date)` | two strings | multi-line listing, or `no bookings for ...` |

`ReservationApp.main` is the only caller; `RequestHandlerTest` is the only test and it
only goes through these strings.

**Structure.** Four classes, no interfaces.

- `ReservationApp` holds a `RequestHandler` (`ReservationApp.java:9`).
- `RequestHandler` holds a `private final InMemoryStore store = new InMemoryStore()`
  (`RequestHandler.java:10`). It constructs its own storage; nothing can be injected.
  It owns: time parsing (copied three times, lines 13-25, 46-57, 69-88), the overlap
  rule (30-36), response formatting, and `minutesToText`.
- `InMemoryStore` owns the two maps plus `hasSlot` and `bookingCount`, which nothing
  calls.
- `BookingPolicy` owns business hours, max length, and an overlap loop
  (`validate`, lines 19-35). **No class holds a reference to it and no code calls it.**
  `grep -rn BookingPolicy src/` finds only its own file.

So the real call graph is `ReservationApp → RequestHandler → InMemoryStore`. The
`BookingPolicy` box in DESIGN.md's diagram is not on the path. Consequently the
business-hours and max-length rules that DESIGN.md says are enforced are not
(observed: `createBooking(..., "22:00", "23:00", ...)` and an `08:00–13:00` five-hour
booking both return `OK`).

**The no-double-booking invariant.** Every place a check happens:

1. `RequestHandler.createBooking`, lines 30-36. A real overlap test
   (`start < slot[1] && slot[0] < end`) over `store.slotsFor(room, date)`. **This is the
   only live enforcement in the system, and it only guards `create`.**
2. `InMemoryStore.addSlot`, lines 23-27. Checks *exact equality* of start and end only.
   `{540,600}` vs `{570,630}` passes. On the create path it is redundant with (1); on the
   reschedule path it is reached, but its `false` return is discarded
   (`RequestHandler.java:99`).
3. `BookingPolicy.validate`, lines 29-33. A correct overlap loop. Dead code.
4. `InMemoryStore.hasSlot`, lines 61-72. Exact match. Dead code.
5. `RequestHandler.rescheduleBooking`: **no overlap check of any kind.**

Trace of one reschedule, `ReservationApp.java:21`:
`handler.rescheduleBooking("WEH-5302", "2026-09-11", "10:00", "11:30", "13:00", "14:30")`

1. `RequestHandler.rescheduleBooking` (67): split all four strings on `:` (69-76), parse
   to minutes (81-88): old = 600–690, new = 780–870.
2. Line 89: `newEnd <= newStart`? No. (Nothing checks hours or length.)
3. Line 93: `store.bookerFor(room, date, 600, 690)` → `bookerBySlot.get("WEH-5302|2026-09-11|600|690")` → `"bo"`.
4. Line 98: `store.removeSlot(room, date, 600, 690)` → walks the list under
   `"WEH-5302|2026-09-11"`, removes `{600,690}` by exact match, removes the booker key.
   Return value ignored.
5. Line 99: `store.addSlot(room, date, 780, 870, "bo")` → exact-duplicate scan only,
   appends `{780,870}`, puts the booker key. Return value ignored.
6. Line 100: return `"OK: moved ..."`.

Between steps 3 and 5 nothing compares 780–870 with the room's other slots. Observed
consequences: rescheduling `09:00–10:00` onto `11:30–12:30` while `11:00–12:00` is
booked returns `OK` and the listing then shows both (a double booking). Rescheduling
onto an interval that already exists exactly returns `OK: moved` while `addSlot`
returned `false`, so the old booking is deleted and the new one never stored: the
booking is silently lost. Rescheduling to `21:00–23:59` returns `OK`.

The tests stay green because every test in `RequestHandlerTest` that touches overlap
goes through `createBooking`; `rescheduleMovesABooking` only moves into an empty slot.

---

## Milestone 2: Two design problems

### Problem 1

**The problem.** Misplaced responsibility. The no-overlap rule is implemented inside the
request-parsing layer instead of in the component the design assigns it to, and the
component that is supposed to own the rules (`BookingPolicy`) is not wired in at all.

**Where in the code.** `RequestHandler.createBooking`, lines 30-36 (the inline overlap
loop); `RequestHandler.rescheduleBooking`, lines 93-99 (the same operation with no rule
at all); `RequestHandler.java:10`, which builds an `InMemoryStore` and never a
`BookingPolicy`. `BookingPolicy.validate` has zero callers.

**What it makes expensive.** Already wrong today: because the rule is a loop pasted into
one method rather than a step every mutation must pass through, the second mutating
operation simply did not get it, and `reschedule` double-books, drops bookings, and
books outside business hours (all observed above). The next change DESIGN.md names,
"a per-building closing time", is the one that gets ugly: an engineer follows the
design doc, edits `BookingPolicy.CLOSING_MINUTE` into a per-building lookup, runs the
green suite, and ships a change that alters nothing, because the hours rule has never
executed. Every new rule (recurring-booking limits, per-user quotas) has to be added to
`create` and `reschedule` separately, and the second copy is the one that gets missed.

### Problem 2

**The problem.** Representational gap. DESIGN.md's unit of thought is "a booking: room,
date, interval, booker" that can be created, moved, and cancelled as one thing. The code
has no such thing: a booking is a bare `long[]` in one map plus a `String` in a second
map, related only by a hand-assembled string key.

**Where in the code.** `InMemoryStore.java:13-14` (the two parallel maps),
`addSlot:28-29` and `removeSlot:41-42` (every mutation must touch both maps in step),
and `RequestHandler.rescheduleBooking:93-99`, where "move a booking" has to be
expressed as look-up-booker, remove, add, across two maps with no rollback.

**What it makes expensive.** Already wrong today: the exact-duplicate reschedule loses
the booking (remove succeeds, add fails, caller cannot tell), because a "move" is not
one operation on one object but three on two maps. Future change: the planned
"bookings that cross midnight" and "recurring bookings" have nowhere to go. A
cross-midnight booking is two different `room|date` keys with no link between them,
so cancel and reschedule would have to find and update both halves by rebuilding keys
at all five key-building sites. A recurrence has no field to live in, because the
booking is a two-element array. Both changes turn into edits to string-key plumbing
in `InMemoryStore` and to each handler method rather than a field on a type.

(Also noticed, not counted as one of the two: a missing boundary. `slotsFor` returns
the store's live list, `RequestHandler` hard-codes `new InMemoryStore()`, and there is
no store interface. The planned database-backed store therefore means editing
`RequestHandler`, and nothing can be substituted in tests.)

---

## Milestone 3: Two alternative decompositions

### Alternative A: one booking type, one admission point, a store behind an interface

**The decomposition.**

- `Booking` (a record: room, date, start, end, user). The single representation of a
  booking; it is what the store holds and what the service passes around.
- `BookingStore` (interface: `add(Booking)`, `remove(Booking)`,
  `forRoomDay(room, date)` returning an unmodifiable list). `InMemoryStore` implements
  it with one map `room|date → List<Booking>`; the second map disappears. A
  database-backed store is a second implementation.
- `ReservationService` owns the four operations in domain types (minutes and
  `Booking`, not strings). Every mutation, create and reschedule alike, goes through
  one private method `admit(candidate, ignoring)` which calls `BookingPolicy.validate`
  against `store.forRoomDay(...)` minus the booking being moved. Reschedule is
  "admit the new booking ignoring the old one, then swap"; if admission fails nothing
  is touched.
- `BookingPolicy` keeps all three rules and is called only from `admit`.
- `RequestHandler` shrinks to parsing (`parseTime` written once) and formatting, and
  constructs nothing: it takes a `ReservationService`, which takes a `BookingStore`
  and a `BookingPolicy`.

Rules live in `BookingPolicy`; the only door to storage is `admit`.

**One tradeoff.** Two new types and an extra layer in a 300-line prototype, and
`ReservationService`'s four signatures mirror `RequestHandler`'s, so for a while there
are two parallel "create/cancel/reschedule/list" surfaces to keep in step. The
`BookingStore` interface has to be designed before the database implementation exists,
so its first real consumer will force it to churn.

### Alternative B: the room-day owns its calendar and its invariant

**The decomposition.**

- `RoomCalendar`: one object per (room, date). It privately owns that day's bookings
  and is the only code that can add, move, or remove one. `add` and `move` do the
  overlap test themselves and refuse rather than mutate. Because the collection is
  never exposed, the no-overlap invariant is enforced by the object that holds the
  data, and there is no path that bypasses it.
- `InMemoryStore` becomes `Map<room|date, RoomCalendar>` plus `calendarFor(room, date)`.
- `BookingPolicy` keeps the per-request rules that need no existing data (business
  hours, max length) and is called by the handler before it touches a calendar.
- `RequestHandler` parses, asks the policy, then asks the calendar; it no longer
  contains any rule.

Rules are split on purpose: shape-of-request rules in `BookingPolicy`, the
consistency invariant in `RoomCalendar`.

**One tradeoff.** The rules now live in two places, which is exactly what DESIGN.md
argued against ("a single place to read when someone asks what the service allows").
And the aggregate boundary bakes in "one room, one day": a booking that crosses
midnight (on the planned list) spans two `RoomCalendar`s, so the invariant that this
design makes unbypassable would have to be re-established across objects, undoing the
main benefit.

### Preference

Alternative A, on the condition that the "Planned next" list in DESIGN.md is real. Three
of its four items push toward A: a database-backed store needs the `BookingStore`
interface; recurring and cross-midnight bookings need a `Booking` type with room for
new fields; and the reschedule defects show that anything with two mutating operations
needs one admission point, not two copies of a loop. Under A the per-building
closing-time change is one edit to `BookingPolicy` that actually runs.

I would pick B instead if RoomReserve stays a per-room-per-day prototype with a small
team and the only real pain is today's reschedule bug. B is the smaller change: no new
interface, no service layer, and it makes the double-booking bug structurally
impossible rather than merely checked. The condition that flips me is the
cross-midnight item: the moment a booking can span two days, B's aggregate boundary is
in the wrong place, and A's single `Booking` plus one `admit` point is the design that
still holds.
