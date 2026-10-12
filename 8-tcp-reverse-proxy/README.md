# Assignment 08 — TCP Reverse Proxy

**Estimated effort:** 24–36 hours. **Result:** a Tokio layer-4 reverse proxy that balances connections across fixed upstreams, forwards bytes in both directions, respects half-close, and bounds memory.

## Prerequisites and learning objectives

Complete projects 02, 04, and 06; project 07 is useful. Understand async socket ownership, partial writes, semaphore permits, task tracking, timeouts, and how a byte stream differs from an HTTP message.

Explain why the proxy has two TCP connections, what backpressure means, why dropping both directions on the first EOF is wrong, and how connection-level load balancing differs from request-level routing.

This is a **TCP reverse proxy for configured backends**. It does not parse HTTP, select backends by URL/Host, terminate TLS, pool connections, or act as an arbitrary-destination forward proxy.

## CLI

```bash
cargo run --quiet -- --bind 127.0.0.1:9000 --upstream 127.0.0.1:7000 --upstream 127.0.0.1:7003 --max-connections 128 --connect-timeout-ms 2000 --idle-timeout-ms 10000 --shutdown-grace-ms 5000
```

At least one `--upstream` is required; accept at most 32. All addresses are numeric IPv4 endpoints. Flag defaults are those shown except `--bind`, default 127.0.0.1:9000. Max connections: 1–1,024. Timeouts/grace: 1–60,000 ms. Support bind port 0 for tests. Duplicate upstreams are CLI errors; reject port zero for upstreams.

Select one upstream for each **admitted** connection using round robin in configured order, starting with upstream index 0. Advance the index once per admission, even if connect fails. Do not automatically retry another upstream. This policy makes failure/replay behavior clear and testable.

## Byte forwarding and lifecycle contract

1. Accept client and acquire one capacity slot before creating relay work. If no slot is available, close the new socket promptly and log overload. Do not queue unlimited waiting clients.
2. Connect to the selected upstream within connect-timeout-ms. During connection setup, do not eagerly buffer arbitrary client input.
3. After connect, relay client → upstream and upstream → client concurrently. Use fixed buffers of at most 16 KiB per direction.
4. Handle partial writes; never discard unwritten bytes. A slow destination must eventually stop reads from that direction's source. Memory may not grow in proportion to total bytes transferred.
5. Client read EOF: finish forwarding already-read client bytes, shut down the upstream **write half**, and keep forwarding the upstream response.
6. Upstream read EOF: finish forwarding upstream bytes, shut down the client write half, and keep the other direction alive if it still has useful work.
7. Normal completion occurs after both relay directions finish. A fatal read/write error closes both directions and records the error; do not continue with a corrupted connection.
8. Idle timeout starts once connected and resets only when at least one byte is **successfully written to either destination**. A blocked write must not defeat the timeout. Time waiting for connect is covered separately.
9. Ctrl+C: stop accepting and allow existing relays to finish for shutdown-grace-ms. After the grace period, cancel remaining relays, close sockets, join tasks, and exit 0.
10. Release capacity on every completion/failure/cancellation path. Keep tracked task bookkeeping bounded and reap completed tasks during normal operation.

Write your own relay loops for the required milestone so you understand partial writes, metrics, and progress tracking. Tokio utilities are useful references; replacing the loops later is optional if you preserve the contract and explain the change.

## Demonstration using project 02

From this directory, start backend A in terminal A:

```bash
cargo run --quiet --manifest-path ../02-tcp-echo-server/Cargo.toml -- server --bind 127.0.0.1:7000 --timeout-ms 5000
```

Backend B in terminal B:

```bash
cargo run --quiet --manifest-path ../02-tcp-echo-server/Cargo.toml -- server --bind 127.0.0.1:7003 --timeout-ms 5000
```

Start the proxy in terminal C using the command above. In terminal D, run two **sequential** client requests:

```bash
cargo run --quiet --manifest-path ../02-tcp-echo-server/Cargo.toml -- client --connect 127.0.0.1:9000 --message "hello rust"
cargo run --quiet --manifest-path ../02-tcp-echo-server/Cargo.toml -- client --connect 127.0.0.1:9000 --message "hello rust"
```

Both clients produce `{"sent_bytes":10,"received_bytes":10,"echo_matches":true,"output":null}`. The first is assigned port 7000, the second 7003. Each direction forwards **14 bytes**, because the application payload has a four-byte framing header. The proxy does not remove or reinterpret it.

Project 02 backends are single-client servers, so do not use them alone to evaluate high-concurrency proxy performance. Use an async mock backend in integration tests.

## Logging and shutdown output

The proxy writes no stdout while running. Stderr JSONL events:

- `listening`: `address`, `upstreams`, `max_connections`.
- `assigned`: `connection_id`, `peer`, `upstream`.
- `overload`: `peer`.
- `connection_complete`: fields below.

Example completion record; the duration is illustrative:

```json
{"event":"connection_complete","connection_id":1,"peer":"127.0.0.1:50123","upstream":"127.0.0.1:7000","client_to_upstream_bytes":14,"upstream_to_client_bytes":14,"duration_ms":2.5,"reason":"eof"}
```

Connection IDs increase from 1 for admitted connections. Byte counters include bytes actually written, including partial progress before failure; count no rejected/unwritten bytes. Duration covers admission to completion, including upstream connect.

Reasons: `eof`, `connect_error`, `connect_timeout`, `idle_timeout`, `io_error`, or `shutdown`. Do not send HTTP error text into an opaque protocol when connect fails; close and log.

On graceful Ctrl+C, emit one final JSON summary to stdout:

```json
{"accepted":2,"completed":2,"failed":0,"overload_rejected":0,"client_to_upstream_bytes":28,"upstream_to_client_bytes":28,"remaining_tasks":0}
```

`accepted` means admitted to a slot. `completed` counts normal EOF completion; `failed` includes all other completion reasons, including forced shutdown. After cleanup, accepted=completed+failed. Byte totals sum admitted connections, including failed ones. Fatal listener errors yield exit 1 and stderr diagnostics.

## Milestones and acceptance tests

1. **Single upstream relay — 5–7 hours.** Forward binary streams with bounded buffers and accurate byte counts.
2. **Half-close and deadlines — 5–7 hours.** Use a backend that waits for request EOF, then returns a response; it must work through the proxy.
3. **Balancing, limits, shutdown — 6–9 hours.** Add round robin, admission control, and task cleanup.
4. **Adverse tests and measurements — 8–13 hours.** Produce a concise performance/limitations report.

Required tests:

| Test | Acceptance |
| --- | --- |
| Sequential requests to two healthy upstreams | Assignments alternate deterministically |
| Arbitrary binary stream larger than the buffer | Exact bytes, accurate counters |
| Client half-closes its write side | Full delayed upstream response arrives |
| Upstream half-closes first | Remaining direction is not immediately discarded |
| Backend reads slowly | Source eventually experiences backpressure; buffers stay bounded |
| Idle connection with small test timeout | Closes for idle_timeout |
| No reachable backend | Connect failure is isolated; proxy remains available |
| Max-connections 2 with two held relays | Third connection rejected; slot returns after cleanup |
| Concurrent load, e.g. 64 clients against async backend | All healthy transfers complete without unbounded task growth |
| Ctrl+C during a stuck relay | Exits after grace, remaining_tasks=0 |

Evaluate backpressure with a test backend that deliberately stops reading and a sender substantially larger than OS socket buffers. Keep a test-controlled deadline so the experiment terminates. Report the theoretical application buffer bound (two relay buffers per active connection), then distinguish it from OS socket buffers, task allocation, and allocator overhead. Do not claim a strict RSS bound based only on the two buffers.

## Explanation questions and assessment

1. Why does this proxy see application bytes but not act as the original TCP connection?
2. Why does the first EOF not necessarily mean the response is complete?
3. Trace how a slow upstream causes the client's send operation to stall.
4. What happens when an upstream fails? Why have you avoided automatic replay?
5. Why can balanced connection counts still produce unbalanced work?
6. What extra protocol logic is needed for HTTP request-level load balancing?
7. What are the tradeoffs between draining and cancelling during shutdown?

**Rubric (100):** bidirectional/half-close correctness 30; backpressure/lifecycle 25; tests/metrics 25; Rust/CLI 10; explanations 10. Extension: explicit backend health checks with a revised selection policy; never replay partially forwarded data.

## Targeted reading

- [Tokio copy_bidirectional](https://docs.rs/tokio/latest/tokio/io/fn.copy_bidirectional.html), especially EOF and shutdown semantics.
- [Tokio shutdown](https://tokio.rs/tokio/topics/shutdown).
- [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html), connection closing and half-close discussion.

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

