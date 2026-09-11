# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** What is a booking, in the code? What types hold it, and what has to stay
in agreement for a booking to make sense?
- A booking is a **value**: a tuple `(String room, String date, String start, String end, String user)`.
- `start` and `end` need to follow the `HH:MM` format at the user endpoint (enforced in `RequestHandler.createBooking()`), but internally they are subsequently converted into minutes since midnight.
- Some more invariants about valid bookings mentioned in the design doc are enforced in `BookingPolicy.validate()`.

**Operations.** What can a caller do, and what goes in and out?
- Create, cancel, reschedule or list bookings.
- For each operation, the params are different combinations of the elements of the tuple that define a booking.

**Structure.** What classes exist, what does each own, and who holds a reference to whom?
- `ReservationApp` is the user endpoint, which just creates an instance of `RequestHandler`.
- `RequestHandler` owns operations, (ideally) approves them with a `BookingPolicy` instance, and stores them in an `InMemoryStore` instance.
- `BookingPolicy` owns the booking invariants in the design doc.
- `InMemoryStore` owns the lowest level of storage.

**The no-double-booking invariant.** Where is it enforced? Name every place a check
happens, say what each one actually checks, and trace one reschedule request through the
code from the entry point to storage.
- Enforced ONLY in the body of `RequestHandler.createBooking()`, when it should be that `createBooking()` and `rescheduleBooking()` delegate the check to `BookingPolicy`.

---

## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** Name it, using the vocabulary from lecture (milestone 2 in the
handout names the three).
- MISSING BOUNDARY between `RequestHandler` and `InMemoryStore`.

**Where in the code.** File and method.
- In `RequestHandler`: initialization of `InMemoryStore store`, and all references to it in the file.

**What it makes expensive.** A concrete future change, or something that already goes
wrong today. What breaks first?
- According to the stack specified in the design doc, requests should not go from `RequestHandler` to `InMemoryStore` before passing through `BookingPolicy` to be validated. The current issue heavily risks invalid bookings skipping `Booking Policy` entirely because there is a direct reference to the store in the request handler.

### Problem 2

**The problem.**
- MISPLACED RESPONSIBILITY in `RequestHandler`.

**Where in the code.**
- `RequestHandler.java:30-37`.

**What it makes expensive.**
- `RequestHandler` is trying to achieve the entire function of `BookingPolicy`, which ends up going wrong in the current code because the check only happens when creating a booking but not rescheduling a booking. It is easier, more modular/efficient, etc. to follow the design doc and delegate this responsibility to an instance of `BookingPolicy` instead.

---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?
- Have *everything* in a single class. The `RoomReserve` service has enough simplicity and few enough lines of code (~300) that this would not be unreasonable, especially if we anticipate no new features. We just have to put slightly more care than in each of the current classes into only exposing what is necessary for the caller and nothing else. (Maybe make just the operations public and everything else private.)

**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.
- If we anticipate the addition of new features (which the design doc does), then this design would fall short of the more modular one in the design doc, since the all the new logic will become increasingly overwhelming to maintain.

### Alternative B

**The decomposition.**
- Take an approach similar to the current implementation, but install a new helper class that handles input/output formatting of booking parameters (room, date, start, end, user). For example, the `room` and `user` parameters are currently case-sensitive, which we probably don't want. And some of the other formatting checks are still underspecified: any timestamp of the form `X:Y` can still be valid even if `X` and `Y` have more or less than 2 digits.
- A huge advantage is that this absolves all other classes of this responsibility, as well as future features (such as accommodating for times past midnight). 

**One tradeoff.**
- If normalizing all input parameters to an internal representation (e.g. "weh5302" -> "WEH 5302"), extra care must be taken so that, when end-users call `listBookings()`, they recognize what is supposed to correspond to their original input.
- More straightforwardly, though, this adds more complexity to the stack, the main 3 components will all have a connection to this new class and hence break the purely linear path from before.

### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.
- I would pick Alternative B, under the current implementations and according to the design doc, because the current code is buggy while the design doc still anticipates more features. Even if I weren't asked to come up with this design, I would still probably arrive at this conclusion naturally, because delegating formatting to its own module facilitates both debugging the current code and implementing the features the design doc speaks of. Again, if there were no new features added in the future, Alternative A could suffice.
