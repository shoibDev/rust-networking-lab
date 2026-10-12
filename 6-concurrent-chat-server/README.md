# Assignment 06 — Concurrent Chat Server

**Estimated effort:** 24–36 hours. **Result:** a Tokio TCP chat service with a documented JSON-line protocol, bounded queues, independent readers/writers, and clean shutdown.

## Prerequisites and learning objectives

Complete projects 02–05. Study Rust Book material on traits, ownership across threads, `Send`/`Sync`, `Arc`, and channels, then Tokio's spawning, I/O, channels, and select tutorials. Understand futures, `async`/`await`, task cancellation, bounded channels, and the difference between concurrency and parallel execution.

You must explain why a slow reader can hurt a naive broadcast loop, why holding a lock during network I/O is dangerous, and how a queue limit turns memory pressure into an explicit policy.

## CLI and demonstration

Server defaults:

```bash
cargo run --quiet -- server --bind 127.0.0.1:7002 --max-clients 32 --queue-capacity 32 --idle-timeout-ms 300000
```

Client in terminal B:

```bash
cargo run --quiet -- client --connect 127.0.0.1:7002 --name alice
```

Client in terminal C:

```bash
cargo run --quiet -- client --connect 127.0.0.1:7002 --name bob
```

Client user input: ordinary lines become chat messages; `/who` requests the roster; `/quit` leaves. Stdin EOF also requests quit. Unknown slash commands produce a local diagnostic and are not sent. The client prints received server objects as JSONL to stdout; any prompt or connection diagnostic goes to stderr.

Use numeric IPv4 endpoints, max-clients 1–1,024, queue capacity 1–1,024, idle timeout 1–600,000 ms. Support server port 0 in tests. A client connect/handshake deadline is 5 seconds. Rejected handshake or unexpected connection failure yields exit 1; completed quit yields exit 0.

## Chat/1 — framing and messages

Each wire message is one UTF-8 JSON object followed by exactly one LF. Maximum frame size is 4,096 bytes **including LF**. Reject invalid UTF-8/JSON, missing LF beyond the limit, raw CR in the frame, unknown message types, and extra/missing fields. Do not use an unbounded `read_line`.

The first message must arrive within 5 seconds and be:

```json
{"type":"join","name":"alice"}
```

Names normalize to lowercase and match `[a-z][a-z0-9_-]{0,15}`. Names are unique among registered clients. Reserve capacity for unregistered accepted connections too; an incomplete handshake must not bypass the limit.

Client commands after successful join:

| Type | Exact fields | Meaning |
| --- | --- | --- |
| `say` | `type`, `text` | Broadcast text |
| `who` | `type` | Request sorted roster |
| `quit` | `type` | Graceful departure |

Text is 1–1,000 UTF-8 bytes and contains no NUL, CR, or LF after JSON decoding. Measure bytes, not Unicode characters. Escaping in JSON does not evade the decoded-byte limit.

Server messages:

| Type | Fields beyond `type` |
| --- | --- |
| `welcome` | `name`, `users` (sorted names including the new user) |
| `joined` | `seq`, `name` |
| `message` | `seq`, `from`, `text` |
| `left` | `seq`, `name`, `reason` |
| `users` | `users` (sorted names) |
| `bye` | `reason` |
| `error` | `code`, `message` |

A central hub assigns a single increasing `u64` sequence beginning at 1 to public joined/message/left events. Recipient order must preserve hub order. Welcome/users/bye/error do not consume sequence numbers. The joining client receives welcome first; its joined event is sent only to existing users. Chat messages are broadcast to all registered users, including the sender. Left goes only to remaining users. Sequence gaps are valid when a recipient was not eligible for an event.

## Reference exchange

1. Alice joins: receives `{"type":"welcome","name":"alice","users":["alice"]}`; joined sequence 1 has no recipients.
2. Bob joins: receives `{"type":"welcome","name":"bob","users":["alice","bob"]}`. Alice receives `{"type":"joined","seq":2,"name":"bob"}`.
3. Alice types `hello bob`: both receive `{"type":"message","seq":3,"from":"alice","text":"hello bob"}`.
4. Bob types `/who`: only Bob receives `{"type":"users","users":["alice","bob"]}`.
5. Bob types `/quit`: Bob receives `{"type":"bye","reason":"quit"}` and EOF; Alice receives `{"type":"left","seq":4,"name":"bob","reason":"quit"}`.

A differently ordered concurrent join produces a different valid event order. The reference assumes each step completes before the next begins.

## Concurrency, bounds, and lifecycle

Use Tokio TCP streams. Give each client a reader and one exclusive writer task. A hub owns the roster and accepts commands over a bounded channel (capacity 256). Use per-client bounded outbound queues with configured capacity.

- The hub performs no socket I/O and never awaits a slow client's writer. Use a nonblocking enqueue policy for fan-out.
- When a recipient's outbound queue is full, disconnect that recipient with reason `slow_client`; do not silently discard chat messages while claiming ordered delivery. Remaining clients keep progressing. Process cleanup through the hub without unbounded recursive fan-out.
- Bound each complete outbound-frame write to 5 seconds. A blocked writer is closed. Capacity counts every admitted socket until cleanup, including one undergoing handshake.
- If all slots are in use, best-effort send `error` code `server_full` with a bounded write and close the newly accepted socket. Handle rejection inline with a maximum 5-second write deadline, so at most one extra rejection socket exists outside the admitted-client cap; do not spawn unbounded rejection tasks.
- Duplicate name: `error` code `name_taken`, then close. Other malformed input: `protocol_error`, then close. Try to send the error within a bounded deadline; delivery is not guaranteed on a broken or full connection.
- After join, require the next full client frame within idle-timeout-ms; a partial frame does not reset the deadline. An idle client is removed.
- Abrupt EOF removes the user with reason `eof`. I/O failure uses `io_error`; idle expiration uses `idle_timeout`.
- Ctrl+C: stop accepting, notify clients with bye reason `server_shutdown` where possible, and close/join tracked tasks within 5 seconds; abort remaining tasks afterward. Exit 0. No detached task may keep the server alive.

Server stdout is empty. Stderr JSONL events `listening` (address), `join` (name), `disconnect` (name/reason), `reject` (code), and `shutdown` (remaining_tasks). Do not log private chat text by default.

## Milestones and acceptance tests

1. **Bounded codec — 4–5 hours.** Test split/coalesced frames, Unicode lengths, invalid JSON, and maximum sizes.
2. **Hub state machine — 5–7 hours.** Unit-test joins, uniqueness, ordering, rosters, and departures without sockets.
3. **Async I/O and client — 7–11 hours.** Wire readers/writers to the hub; run the reference exchange.
4. **Backpressure/shutdown — 8–13 hours.** Add bounds, deadlines, instrumentation, and adversarial tests.

Required evidence: at least 20 simultaneous clients exchange messages; all eligible healthy clients see identical event order; a duplicate normalized name is rejected; disconnected names become reusable; a partial oversized frame cannot exhaust memory; unregistered clients count toward the cap; abrupt disconnect produces one cleanup; and Ctrl+C exits within the 5-second grace period plus small scheduler tolerance.

Test the slow-client policy deterministically by placing a writer behind a test-controlled gate and using a tiny queue. Do not depend solely on filling OS TCP buffers on your machine. Separately demonstrate that one nonreading socket does not block healthy clients. Test a hub channel under load so you can explain inbound as well as outbound backpressure.

## Explanation questions and assessment

1. How do async tasks differ from one operating-system thread per client?
2. Who owns the roster, and why does that simplify consistency?
3. Why is one writer per connection helpful for framing?
4. What happens when hub input fills? What happens when a client output queue fills?
5. Why are sequence gaps valid but reordered delivered events invalid?
6. How do cancellation and EOF reach hub cleanup exactly once?
7. Which guarantees disappear when a connection closes before queued events are delivered?

**Rubric (100):** protocol/hub semantics 25; concurrency/bounds 30; tests/shutdown 25; Rust organization 10; explanations 10. Extension: named rooms with separate ordered event streams and per-room membership tests.

## Targeted reading

- [Tokio channels](https://tokio.rs/tokio/tutorial/channels), [select](https://tokio.rs/tokio/tutorial/select), and [shutdown](https://tokio.rs/tokio/topics/shutdown).
- [Rust Book chapter 16](https://doc.rust-lang.org/book/ch16-00-concurrency.html).

## Submission and engineering standards

This is an assignment specification, not a supplied implementation. Commands and outputs describe the program you must build. Complete required features before attempting extensions.

- Work in this directory as an independent Cargo package; do not add a nested Git repository.
- Use stable Rust, edition 2024. Record `rustc --version` and `cargo --version` in `REPORT.md`. Commit `Cargo.lock` for reproducible application builds.
- Implement `--help`. Reject unknown flags, missing values, out-of-range values, and invalid flag combinations. Bind servers to loopback in examples.
- Process exit codes: `0` for success/help; `2` for CLI validation failures; `1` for runtime, file, protocol, or operational failures. Cargo itself may return different codes; assess the compiled executable when checking this contract.
- Send requested results to stdout and diagnostics to stderr. If a project defines JSON/JSONL, emit valid UTF-8 JSON, never Rust debug formatting. JSON key order is not significant. No ANSI color in machine-readable output.
- Do not panic on external input. Propagate recoverable failures with `Result`; keep error messages actionable. Document any `unwrap`/`expect` remaining outside tests and justify the invariant.
- Set explicit bounds on buffers, accepted input, and outstanding work. Use monotonic time for elapsed intervals; use wall-clock time only where explicitly required.
- Organize reusable logic under `src/lib.rs`; keep `src/main.rs` focused on argument handling, configuration, and process exit behavior. The module names suggested below are a guide, not a grading requirement.

Run these checks from this directory after implementing the assignment:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
cargo build --release
```

Include `REPORT.md` containing your architecture, protocol or format decisions, an evidence table for acceptance tests, commands and observed results, dependency choices, limitations, and answers to the explanation questions. Include `tests/` for integration tests and `examples/` for small, reproducible transcripts or generated output; do not submit private traffic or oversized files. For every test, state what bug it would catch. Integration tests must use temporary directories and ephemeral ports where possible, bound their runtime, and clean up child processes.

**Completion gate:** required tests pass, the reference demonstration works, errors follow the exit contract, and you can explain the data path and one failure path without reading your implementation. A successful demonstration alone does not satisfy the assignment.

