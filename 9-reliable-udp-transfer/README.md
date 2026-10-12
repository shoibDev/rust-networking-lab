# Assignment 09 — Reliable File Transfer over UDP

**Estimated effort:** 40–60 hours. **Result:** a custom, bounded UDP file-transfer protocol with selective acknowledgments, retries, integrity verification, safe completion, and reproducible network impairment.

## Prerequisites and learning objectives

Complete projects 03, 05, 06, 07, and 08. Be comfortable with binary codecs, async timers, sequence numbers, bounded maps, incremental hashing, temporary files, and state-machine testing.

Explain reliability versus integrity versus authenticity, retransmission, selective acknowledgments, sliding windows, duplicate suppression, pacing, receiver completion versus sender knowledge, and why a final acknowledgment can also be lost.

This is an educational protocol for loopback or a controlled LAN. Its fixed window and pacing are not general Internet congestion control. It does not replace TCP/QUIC or provide encryption, authentication, resume-after-crash, multiple simultaneous transfers, or persistent deduplication across restarts.

## Required CLI and demonstration

Receiver in terminal A:

```bash
cargo run --quiet -- receive --bind 127.0.0.1:9100 --output received.bin --deadline-ms 120000
```

Create a deterministic 1 MiB fixture in terminal B:

```bash
python3 -c "from pathlib import Path; Path('payload.bin').write_bytes(bytes(range(256))*4096)"
cargo run --quiet -- send --target 127.0.0.1:9100 --input payload.bin --chunk-bytes 1024 --window 8 --rto-ms 200 --max-retries 8 --rate-pps 100 --deadline-ms 120000 --transfer-id 42
```

Compare only after the sender succeeds; the receiver remains available for duplicate FIN messages for 30 seconds before exiting:

```bash
python3 -c "from pathlib import Path; assert Path('payload.bin').read_bytes() == Path('received.bin').read_bytes()"
```

Sender: target/input required; defaults otherwise as shown. Transfer ID is an optional unsigned nonzero 64-bit value; if absent, generate from OS randomness. Numeric IPv4 endpoints only.

Limits: file 0–67,108,864 bytes; chunk 256–1,172 bytes; window 1–64; initial RTO 10–3,200 ms; max retries 1–8; rate 1–100 DATA datagrams/second; deadline 1–3,600,000 ms. The 120-second default will not handle every maximum-size/low-rate transfer; choose a suitable deadline. Receiver accepts one transfer and then exits after completion linger. Support port 0 in tests.

Receiver refuses an existing final output file before listening. Its parent must already exist. This command never receives a filename from the network.

## RUF/1 datagram format

Every datagram has a 28-byte header and at most 1,172 payload bytes; maximum total size is 1,200 bytes. This is a conservative lab sizing choice, not proof of a discovered path MTU.

| Offset | Bytes | Field |
| --- | --- | --- |
| 0 | 4 | ASCII `RNUF` |
| 4 | 1 | Version = 1 |
| 5 | 1 | Type code |
| 6 | 2 | Reserved = zero |
| 8 | 8 | Transfer ID, unsigned big-endian |
| 16 | 4 | Sequence, unsigned big-endian |
| 20 | 2 | Payload length, unsigned big-endian |
| 22 | 2 | Reserved = zero |
| 24 | 4 | CRC-32, unsigned big-endian |
| 28 | L | Payload |

CRC uses **CRC-32/ISO-HDLC** (the common IEEE CRC-32): polynomial 0x04C11DB7, reflected implementation polynomial 0xEDB88320, init/xorout 0xffffffff, reflected input/output. Compute over the entire header+payload with CRC bytes set to zero. Test vector `"123456789"` gives 0xcbf43926. A checksum crate is permitted.

Require exact datagram length 28+L, valid magic/version/reserved fields, and matching CRC. Use a receive buffer large enough to identify oversize datagrams; never accept a truncated receive as valid. Drop invalid/corrupted datagrams silently and count them diagnostically. They do not reset progress timers.

## Types, payloads, and sequence rules

Let N=`ceil(file_bytes/chunk_bytes)`, with N=0 for an empty file.

| Type/code | Sequence | Payload |
| --- | --- | --- |
| START / 1 | 0 | 44-byte manifest |
| START_ACK / 2 | 0 | Identical 44-byte manifest |
| DATA / 3 | 1–N | File chunk |
| ACK / 4 | Exact DATA sequence acknowledged | Empty |
| FIN / 5 | N+1 | Expected SHA-256, 32 raw bytes |
| FIN_ACK / 6 | N+1 | Verified SHA-256, 32 raw bytes |
| ERROR / 7 | 0 | One-byte error code + UTF-8 message, total 1–161 bytes |

Manifest offsets within its payload: file size `u64` at 0; chunk size `u16` at 8; window `u16` at 10; SHA-256 raw bytes at 12–43. Validate all announced bounds before creating a file. Error codes: 1 busy, 2 invalid_manifest, 3 digest_mismatch, 4 file_io, 5 protocol_error. Unknown types and invalid payload lengths are invalid datagrams.

All DATA payloads are exactly chunk_bytes except the last, whose length is the remaining bytes. Empty files send no DATA. The sender calculates the manifest hash before START and streams the input afterward; do not load the whole file. The input must remain unchanged during the transfer. A changed file cannot be reported successfully unless it still matches the manifest digest.

## Sender state machine and retry policy

States: prepare → awaiting START_ACK → sending DATA → awaiting FIN_ACK → success/failed.

- Match both endpoint and transfer ID for all replies. START_ACK must echo the manifest; FIN_ACK must carry the expected sequence/hash. ACKs outside the transmitted range are ignored.
- The sender's base is the lowest not-yet-acknowledged sequence. Send only sequences in `[base, base+window-1]`, clipped to N. Out-of-order ACKs mark individual chunks but do not move base past a gap.
- Keep at most window outstanding chunks plus bounded bookkeeping. Advance base across consecutively acknowledged chunks. Do not resend an already acknowledged chunk.
- Every outstanding DATA and each START/FIN control exchange has a retry timer. Each packet begins at rto-ms; after every timeout double its delay up to 3,200 ms. Permit max-retries retransmissions after the initial transmission, then fail with exit 1.
- Repeated or unrelated ACKs do not reset other packet timers. Start each timer at actual transmission, not while waiting in a pacing queue.
- Pace original and retransmitted DATA at rate-pps, with at least 1/rate seconds between DATA sends; do not emit catch-up bursts. START/FIN use their bounded exponential retry policy.
- Keep retransmission scheduling fair across overdue packets. Do not let one missing sequence permanently starve another.
- Send FIN only after every DATA is acknowledged. Succeed only after a valid FIN_ACK; a DATA ACK alone is not evidence of a verified published file.
- Apply one overall deadline beginning with the initial START send. No progress packet may extend it. A valid matching ERROR, exhausted retry budget, deadline, or I/O failure yields exit 1.
- CRC and SHA-256 detect corruption; they do not authenticate the peer. Transfer IDs also do not authenticate anything.

## Receiver state, disk handling, and completion

States: waiting START → receiving → completed/lingering → exit.

- The first valid START locks the receiver to source IP/port and transfer ID. A duplicate identical START gets the same START_ACK without truncating/recreating the file. A conflicting manifest on the active identity gets protocol_error. Other START identities get best-effort busy; unrelated DATA/ACK/FIN are ignored.
- Waiting for initial START is bounded by deadline-ms from startup. Once active, use a new deadline-ms for transfer completion and a fixed 30-second **progress timeout**. Only accepting a new valid DATA chunk advances progress time; duplicates and unrelated traffic do not. A valid START begins this progress timer.
- Create a unique temporary file in the final output directory with exclusive creation. On failure/timeout/cancellation, close and delete it. Never publish an incomplete final output.
- Track next expected sequence. Accept only valid chunks in `[next_expected, next_expected+window-1]`, clipped to N; buffer at most window chunks. Flush contiguous chunks in order and update an incremental SHA-256.
- ACK a chunk only after successfully storing it in the bounded buffer or temporary file. Duplicate accepted chunks are re-ACKed without a second write. DATA below next_expected is already stored and may be re-ACKed; future chunks beyond the receive window are dropped without ACK.
- Receiving all chunks is insufficient to publish. On a valid FIN after all N chunks, compare byte count, manifest hash, FIN hash, and computed hash. A mismatch returns digest_mismatch, deletes the temporary file, and fails.
- Flush and sync the file, then publish without overwriting an existing final path. Use an atomic no-clobber operation supported by the filesystem; same-filesystem hard-link-then-unlink is one possible implementation for this lab. If the filesystem cannot support your chosen operation, fail explicitly. Do not rely on an existence check followed by an overwriting rename.
- Send FIN_ACK only after successful verification and publication. After publication, retain completed identity/hash for **30 seconds** and re-ACK duplicate FINs; never republish. The retry cap makes even the slowest configured FIN exchange bounded to 28.8 seconds after its first attempt. The linger is a bounded mitigation, not a guarantee against unlimited network delay.
- Receiver failure after publication cannot unpublish a file merely because FIN_ACK was lost. Explain that the sender can still time out despite receiver success.
- During linger, duplicate stored DATA may be ACKed and identical START may receive START_ACK. Other identities remain busy. Exit 0 after linger; Ctrl+C during active transfer cleans the temporary file and exits 1.

No resume after crash is required. Do not claim exactly-once delivery across process restarts.

## Required impairment relay

Add a third mode to make testing portable:

```bash
cargo run --quiet -- impair --bind 127.0.0.1:9101 --target 127.0.0.1:9100 --scenario loss
```

Send to 127.0.0.1:9101 instead of directly to the receiver. The relay pins the first sender endpoint, forwards to its configured receiver using its own socket, and sends returning traffic back to the pinned sender. It never edits transfer IDs or recomputes CRC after corrupting a payload. Restart it between experiments.

Required scenarios:

| Scenario | Deterministic rule |
| --- | --- |
| `clean` | Forward unchanged |
| `loss` | Drop the first DATA for each sequence divisible by 5; drop the first ACK for each sequence divisible by 7; later occurrences pass |
| `reorder` | Hold even DATA until DATA with sequence held+1 arrives, deliver that odd packet first, then held even; flush a held packet after 50 ms; duplicate DATA with sequence divisible by 3 |
| `corrupt` | Flip bit 0 of the first payload byte on the first DATA for sequences divisible by 5; later occurrences pass |
| `blackhole` | Forward handshake/control traffic but discard every DATA |

Bound relay memory: at most one held reorder datagram, no growing delay queues, and at most N sequence bookkeeping entries for this one transfer. If another even DATA arrives while a different sequence is held, flush the held packet before holding the new one; a retry of the already held sequence may pass without replacing it. Apply duplication when forwarding a qualifying DATA, including when flushing held data. Pin the sender only after a CRC-valid, bounds-valid START; invalid/unrelated traffic must not change the pinned peer. `clean` is the default scenario; bind/target are required. Stderr JSONL records impairment actions, type, and sequence; stdout stays empty. Stop with Ctrl+C. This relay is a test utility, not a router.

For empty files the blackhole DATA rule has nothing to drop; use a nonempty file in failure tests.

## JSON result and diagnostics

Sender successful stdout:

```json
{"role":"sender","transfer_id":42,"file_bytes":1048576,"chunk_bytes":1024,"window":8,"chunks":1024,"sha256":"fbbab289f7f94b25736c58be46a994c441fd02552cc6022352e3d86d2fab7c83","datagrams_sent":1026,"retransmissions":0,"elapsed_ms":10240.0,"goodput_mbps":0.8192,"verified":true}
```

The hash corresponds to the specified 1 MiB fixture. Counts assume no retries; timings/goodput are illustrative. `datagrams_sent` counts all successfully sent START/DATA/FIN transmissions. `retransmissions` counts all repeated START/DATA/FIN sends. Elapsed time runs from initial START to accepted FIN_ACK; goodput = file_bytes*8/elapsed_seconds/1,000,000.

Receiver successful stdout, emitted after linger:

```json
{"role":"receiver","transfer_id":42,"file_bytes":1048576,"chunks":1024,"sha256":"fbbab289f7f94b25736c58be46a994c441fd02552cc6022352e3d86d2fab7c83","output":"received.bin","verified":true}
```

Receiver `chunks` counts unique stored DATA chunks, not retries. Empty-file goodput is zero. Success summaries are emitted only on success; failures use stderr and exit 1. Log state transitions, retries (type/sequence/attempt), rejected frames, and verification failure as JSONL to stderr; never dump file payloads.

## Milestones, tests, and evaluation

1. **Codec/hash/state skeleton — 6–8 hours.** Validate every offset and CRC vector. Write pure transition tests before network I/O.
2. **Stop-and-wait — 10–14 hours.** Set window=1; complete handshake, data, final verification, no-clobber publication, retries, and lost FIN_ACK recovery.
3. **Sliding window — 8–12 hours.** Enable window>1, selective ACKs, out-of-order buffering, bounded memory, pacing, and fair timers.
4. **Impairment relay — 8–12 hours.** Build the scenarios above; record successful hash-identical transfers under loss/reorder/corrupt.
5. **Failure matrix/report — 8–14 hours.** Test deadlines, cleanup, and metrics; compare window 1 and 8 using the same controlled RTT/delay harness.

Required tests: empty file; one byte; exact chunk multiple; last partial chunk; maximum accepted file size with a suitable deadline; rejected over-limit manifest; duplicated START does not truncate; duplicate/out-of-order DATA writes once; ACK loss; DATA loss; corrupt CRC ignored and retransmitted; wrong peer/ID ignored; out-of-window ACK ignored; first FIN_ACK lost; all FIN_ACKs lost; final hash mismatch; existing output refused; disk-write/publication failure; blackhole terminates with retry/deadline failure; temporary file removed after failure; and zero remaining transfer tasks after cleanup.

Use fake/injected time or Tokio's paused-time test support for long retry/linger tests rather than waiting 30 seconds in every test. Include at least one real-socket transfer through each successful relay scenario. Show buffer/window occupancy instrumentation proving your bound.

## Explanation questions and assessment

1. Why does UDP need your sequence/ACK logic even though the OS already implements checksums?
2. Explain how a lost ACK causes a duplicate DATA and why that must not duplicate file bytes.
3. Walk through base=2, window=4, and ACKs for 3/4/5 arriving before 2.
4. Why is CRC per packet useful when the whole file also has SHA-256?
5. Why do neither checksum provide peer authentication?
6. What does FIN_ACK prove, and what uncertainty remains if every FIN_ACK is lost?
7. How do window, RTT, and pacing bound throughput? Separate a window/RTT bound from the configured rate cap.
8. What would Internet congestion control, restart-safe resume, or multiple receivers add?

**Rubric (100):** protocol/state correctness 30; retries/windows/bounds 25; integrity/completion/disk safety 15; impairment evidence/tests 20; explanations 10. Extension: adaptive RTT estimation and retransmission timeout, with ambiguity handling for retransmitted samples; or congestion-aware window control evaluated in an isolated emulator.

## Targeted reading

- [RFC 8085](https://www.rfc-editor.org/rfc/rfc8085.html), sections 3.1–3.2: congestion, pacing, and datagram sizing.
- [RFC 6298](https://www.rfc-editor.org/rfc/rfc6298.html): TCP timer design, as background for an optional adaptive-RTO extension. RUF/1's required retry policy is a course design, not an implementation of this RFC.
- [RFC 1350](https://www.rfc-editor.org/rfc/rfc1350.html): compare TFTP's stop-and-wait approach with your design.
- [Tokio time](https://docs.rs/tokio/latest/tokio/time/) and [test-util features](https://docs.rs/tokio/latest/tokio/#feature-flags).

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

