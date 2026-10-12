# Assignment 03 — UDP Network Monitor

**Estimated effort:** 16–24 hours. **Result:** a UDP probe server and client that measure application-level RTT and missed replies, export CSV samples, and summarize results without misleading claims.

## Prerequisites and learning objectives

Complete project 02. Learn `UdpSocket`, `Instant`, `Duration`, byte-array conversion, numeric sorting, and `Option<f64>`. Explain datagrams, no connection handshake, loss/duplication/reordering, socket source addresses, RTT, and why an application timeout does not prove physical packet loss.

This tool sends custom UDP probes to **your probe server**. It is not ICMP ping and does not scan a network. Start with a stop-and-wait probe loop: one outstanding probe at a time.

## Required CLI

| Mode | Flags |
| --- | --- |
| `server` | `--bind 127.0.0.1:7001`; `--drop-every 0`; `--delay-ms 0` |
| `probe` | `--target ADDRESS` required; `--count 10`; `--interval-ms 1000`; `--timeout-ms 500`; `--csv PATH` optional; `--session-id U64` optional |

Require count 1–10,000, interval/timeout 1–60,000 ms, delay 0–60,000 ms, and drop-every 0–10,000. Use numeric IPv4 endpoints. The client binds an ephemeral loopback port for loopback targets; for other targets bind an appropriate unspecified local IPv4 address. If no session ID is supplied, generate one from an OS randomness source; this identifier is not authentication.

Terminal A:

```bash
cargo run --quiet -- server --bind 127.0.0.1:7001
```

Terminal B:

```bash
cargo run --quiet -- probe --target 127.0.0.1:7001 --count 5 --interval-ms 100 --timeout-ms 200 --session-id 42 --csv samples.csv
```

## Wire format — Probe/1

Every probe is exactly 24 bytes; the server echoes all 24 bytes unchanged.

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | ASCII `RNMP` |
| 4 | 1 | Version, value 1 |
| 5 | 3 | Reserved, all zero |
| 8 | 8 | Session ID, unsigned big-endian |
| 16 | 8 | Sequence number, unsigned big-endian, beginning at 1 |

Use a receive buffer large enough to detect oversized datagrams, such as 65,535 bytes, and reject every datagram not exactly 24 bytes or with invalid magic/version/reserved fields. Echo to the source address returned by `recv_from`.

The client must match source IP/port, session, sequence, and full payload before accepting a reply. Record the send instant locally; the server does not read or compare client clocks.

- Start RTT measurement immediately before the send call. A successful send starts the absolute reply deadline.
- Read until the current deadline, ignoring unrelated, duplicate, and late responses. Resetting a full timeout for every bad datagram is forbidden.
- After success or timeout, wait `interval-ms` before sending the next probe. Thus the interval is a **gap between completed attempts**, not a fixed-rate scheduler.
- Do not retransmit probes. A timeout is a missed reply for that attempt. Continue through all requested attempts.
- A send failure is an operational failure and yields exit 1. Completing a valid probe run, including 100% missed replies, yields exit 0.
- The controlled server delay is blocking in this version. Count only valid probes globally since server startup; `drop-every N` discards valid probe numbers N, 2N, and so on. Invalid datagrams do not increment this counter.

## CSV and JSON output contracts

CSV uses LF, a header, and one row per attempt in sequence order:

```csv
sequence,status,rtt_ms
1,ok,0.420
2,timeout,
3,ok,0.600
```

`status` is `ok` or `timeout`. RTT is nonnegative milliseconds with three decimal places; timeout RTT is empty. Refuse to overwrite an existing CSV and create no implicit parent directories. A CSV write failure yields exit 1.

Stdout contains one summary object:

```json
{"target":"127.0.0.1:7001","session_id":42,"sent":5,"received":5,"timeouts":0,"reply_loss_pct":0.0,"rtt_min_ms":0.42,"rtt_mean_ms":0.53,"rtt_p95_ms":0.71,"rtt_max_ms":0.71,"mean_abs_rtt_delta_ms":0.10}
```

Timing numbers above are **illustrative**, not expected localhost measurements. Field definitions:

- `sent` = successful sends; `received` = matched timely replies; `timeouts = sent - received`.
- `reply_loss_pct = 100 * timeouts / sent`. Call this *missed-reply percentage*, not a direct measurement of one-way packet loss.
- Compute RTT statistics on successful replies only. P95 uses nearest rank: sort n samples and choose one-based rank `ceil(0.95*n)`.
- `mean_abs_rtt_delta_ms` is the mean of `abs(rtt[i]-rtt[i-1])` **only for successful consecutive sequence numbers**. Exclude pairs separated by a timeout. This is a custom variation metric, not standardized RTP jitter.
- With no successes, all five RTT/variation fields are null. With one success, min/mean/p95/max are that sample and variation is null. With no eligible consecutive pair, variation is null.
- JSON numeric comparisons use tolerances; emit finite numbers and never NaN/infinity. Serialize the unrounded internal measurements, not CSV-rounded values.

Server stdout is empty. Stderr JSONL: `listening` with `address`; `reply` or `drop` with `session_id` and `sequence`; `invalid_datagram` with `reason`. Manual Ctrl+C termination is sufficient.

## Milestones and acceptance tests

1. **Codec and statistics — 4–5 hours.** Test byte offsets/endian conversion and pure summary functions using synthetic values.
2. **Echo server and client — 4–6 hours.** Demonstrate one matched reply and one deadline timeout.
3. **Validation and exports — 4–5 hours.** Implement source/session filtering, bounded deadlines, CSV, and JSON.
4. **Controlled experiments — 4–8 hours.** Collect baseline, dropping-server, and delayed-server reports.

Required tests:

| Case | Expected result |
| --- | --- |
| 5 probes, no impairment | 5 matched replies on a healthy local machine; dynamic RTT |
| 6 probes, fresh server with `--drop-every 3` | 4 received, 2 timeouts, approximately 33.333333% missed replies |
| Server delay 300 ms, client timeout 100 ms | All 3 attempts time out; late replies never count for later sequences |
| Wrong source/session/sequence | Ignore; original deadline remains fixed |
| 23-byte, 25-byte, and bad-version messages | Server does not echo |
| Synthetic RTTs 1, 2, 3, 4, 10 on consecutive successes | Mean 4; p95/max 10; variation 2.25 |
| Successes only at sequences 1 and 3 | Variation null |
| Zero successful replies | All RTT fields null; valid completed run exits 0 |

Use a fresh dropping server so its global counter starts at zero. Local timing checks must allow scheduling tolerance; do not grade RTT with exact wall-clock assertions.

## Explanation questions and assessment

1. Which directions could fail when a probe times out?
2. Why can RTT be measured without synchronized clocks, while one-way latency generally cannot?
3. Why does UDP `connect`, if you use it, not create a TCP-style handshake?
4. What prevents a late reply from being mistaken for the next probe?
5. Explain nearest-rank P95 for five samples and why it differs from the mean.
6. What are the limits of the sequential server and stop-and-wait measurement design?
7. Why should a future high-rate version add pacing and congestion awareness?

**Rubric (100):** protocol/deadlines 30; statistics 20; controlled evidence/tests 25; Rust/CLI 10; explanations 15. Extension: fixed-rate concurrent probes with a bounded outstanding map; compare the measurement bias introduced by overlap.

## Targeted reading

- [UdpSocket](https://doc.rust-lang.org/std/net/struct.UdpSocket.html).
- [RFC 8085](https://www.rfc-editor.org/rfc/rfc8085.html), sections 3.1 and 3.2: responsible UDP use and message sizing.

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

