# Assignment 02 — TCP Echo Server and Client

**Estimated effort:** 12–18 hours. **Result:** a blocking TCP client/server pair that echoes arbitrary byte messages using a defined length-prefixed protocol.

## Prerequisites and learning objectives

Complete project 01. Study ownership of resources, slices, `Vec<u8>`, `Read`/`Write` traits, loops, `std::io::ErrorKind`, and `Result` propagation. Read the standard `TcpListener` and `TcpStream` documentation.

Explain a listening socket versus an accepted stream, IP address versus port, TCP connection establishment at a conceptual level, EOF, half-close, partial reads/writes, and application framing. The operating system implements TCP; this assignment does not implement TCP packets.

Start with a **single-client-at-a-time blocking server**. Later concurrency is intentionally reserved for project 06. Do not assume one write produces one read.

## Required CLI and demonstration

| Mode | Flags and defaults |
| --- | --- |
| `server` | `--bind 127.0.0.1:7000`; `--timeout-ms 5000` |
| `client` | `--connect ADDRESS` required; exactly one of `--message TEXT` or `--input PATH`; `--output PATH` optional; `--timeout-ms 5000` |

Timeout range: 1–60,000 ms. Numeric IPv4 socket addresses are required; DNS and IPv6 are extensions. Ports 1–65535 for manual commands; support bind port 0 in tests and print the actual bound address.

Terminal A:

```bash
cargo run --quiet -- server --bind 127.0.0.1:7000 --timeout-ms 5000
```

Terminal B:

```bash
cargo run --quiet -- client --connect 127.0.0.1:7000 --message "hello rust"
```

Exact client result, excluding key order:

```json
{"sent_bytes":10,"received_bytes":10,"echo_matches":true,"output":null}
```

Do not print message contents to stdout. With `--output`, write the exact echoed bytes to that path and return its supplied spelling in `output`. If parent directories do not exist, report exit 1 rather than creating them implicitly. An existing output file may be replaced only after receiving and validating the entire response. `--message` encodes UTF-8; `--input` reads binary bytes. Reject inputs over 65,536 bytes.

## Wire protocol — Echo/1

Each request and response is:

| Offset | Size | Meaning |
| --- | --- | --- |
| 0 | 4 bytes | Unsigned payload length, network/big-endian order |
| 4 | L bytes | Payload, 0 ≤ L ≤ 65,536 |

For `hi`, the bytes are `00 00 00 02 68 69`. The echo response must match the entire frame. A zero-length message is valid.

The server processes any number of request frames on one connection, in order, and responds once per complete frame. The supplied CLI client sends one frame, reads one response, verifies equality, and closes.

- Maintain explicit progress through a four-byte header and then its payload. A partial header/payload followed by EOF is a protocol error. EOF exactly between frames is a normal disconnect.
- Reject an excessive length immediately; do not allocate according to that length. Close this connection and continue accepting clients.
- Handle partial writes with `write_all` or a verified loop. A single `write` is insufficient.
- Set read and write timeouts on both peers. In this beginner version these are **I/O inactivity bounds**, not an end-to-end request deadline; document that a slow trickle can prolong a request.
- The client must verify returned length and bytes, not just print a response. Mismatch yields exit 1.
- A malformed or disconnected client must not terminate the listener. A fatal listener/bind error yields exit 1.

Server emits one JSONL object per event to stderr: `listening` with `address`; `connected` with `peer`; `echo` with `peer` and `bytes`; `closed` with `peer` and `reason`. Reasons are `eof`, `timeout`, `protocol_error`, or `io_error`. Server stdout is empty. Stop the manually run server with Ctrl+C; OS termination is sufficient for this first assignment.

## Suggested milestones

1. **Frame codec — 3–4 hours.** Implement pure encoding and bounded decoding. Test an in-memory reader that supplies only one byte at a time.
2. **Server — 3–4 hours.** Bind, accept, and serve one connection in a loop. Demonstrate two messages on one connection with a test client.
3. **Client — 2–3 hours.** Connect, encode, validate the echo, optionally write a file. Configure timeouts and diagnostics.
4. **Adversarial tests and report — 4–7 hours.** Use ephemeral ports and actual sockets. Explain why fragmentation/coalescing is legal.

Suggested modules: `frame`, `server`, `client`. Use `std::net` and blocking `std::io`. No Tokio or networking framework. A JSON crate and CLI crate are permitted.

## Acceptance tests

| Scenario | Required evidence |
| --- | --- |
| Reference message | Exact 10-byte success summary |
| Empty payload | Length zero accepted; echo matches |
| Non-UTF-8 payload containing 0x00 and 0xff | Output file byte-for-byte identical |
| Header delivered in four separate writes | One complete echo, no premature parse |
| Two frames in one write | Two correct responses on the same connection |
| EOF after two header bytes or midway through payload | Only that connection fails |
| Length 65,537 | Rejected without allocating that payload |
| Idle client with short test timeout | Connection closes; listener remains usable |
| Client cannot connect | Exit 1 and useful endpoint diagnostic |
| Mismatching mock server response | Client exit 1; no successful output file |

For the binary demonstration, create a fixture:

```bash
python3 -c "from pathlib import Path; Path('input.bin').write_bytes(bytes([0,255,1,2,3]))"
cargo run --quiet -- client --connect 127.0.0.1:7000 --input input.bin --output echoed.bin
python3 -c "from pathlib import Path; assert Path('input.bin').read_bytes() == Path('echoed.bin').read_bytes()"
```

Python is an optional fixture utility, not a runtime dependency of your Rust application. Test 65,536 bytes and randomized payloads in Rust as well.

## Explanation questions and assessment

1. Why can six bytes sent together arrive in several reads?
2. What goes wrong if you read directly into a UTF-8 string?
3. Draw the sequence of socket calls from listener startup through one request and clean EOF.
4. Why do you need to distinguish EOF before a header from EOF during it?
5. Who owns the accepted socket, and what happens when that owner drops it?
6. Why does this single-client server stall other clients? Distinguish blocking from packet loss.
7. Contrast an I/O inactivity timeout with a deadline for the whole request.

**Rubric (100):** framing/binary correctness 35; client/server behavior 20; adverse tests 20; Rust/errors 10; explanations 15. Extension: add a bounded total request deadline without changing the wire format.

## Targeted reading

- [TcpStream](https://doc.rust-lang.org/std/net/struct.TcpStream.html) and [Read](https://doc.rust-lang.org/std/io/trait.Read.html).
- [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html), section 2: TCP service model.

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

