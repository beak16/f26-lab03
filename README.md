# Lab 3 Starter: RoomReserve

RoomReserve is a small room-reservation service. Callers create, cancel, reschedule,
and list bookings for a room on a day. It ships with a design document, about 300 lines
of code, a green test suite, and CI.

You write no code in this lab. You read the system and critique its design.

**Read `DESIGN.md` first, then read the code.** Your critique goes in `CRITIQUE.md`.

## Build and test

```
mvn test
```

Everything is green. You can also run the demo script:

```
mvn compile
java -cp target/classes edu.cmu.cs214.roomreserve.ReservationApp
```

## Where things are

- Design document: `DESIGN.md`
- Code: `src/main/java/edu/cmu/cs214/roomreserve/`
- Tests: `src/test/java/edu/cmu/cs214/roomreserve/RequestHandlerTest.java`
- Your writeup: `CRITIQUE.md`
- Setup: `SETUP.md`

See the Lab 3 handout on the course page for the three milestones you show a TA.

## AI tool usage

Claude Code, using the Sonnet 5 model (model id: `claude-sonnet-5`).
