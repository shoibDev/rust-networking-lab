# Assignment 04 — HTTP/1.1 Server from TCP

**Estimated effort:** 20–30 hours. **Result:** a small, bounded HTTP/1.1 server supporting three routes, with a byte-correct parser and an explicit protocol subset.

## Prerequisites and learning objectives

Complete project 02; project 03 is recommended. Understand borrowed byte slices, buffered reads, finding delimiters, ASCII versus UTF-8, structs/enums for parse results, and deadlines.

Explain HTTP request/response structure, header/body separation, byte-counted Content-Length, methods and status codes, and why HTTP framing is independent of TCP reads. This is a learning implementation over blocking `TcpListener`/`TcpStream`; do not use Axum, Hyper, Actix, or an HTTP parser crate for the required parser.

## Required CLI and scope

```bash
cargo run --quiet -- --bind 127.0.0.1:8080 --timeout-ms 5000
```

Defaults are the values shown. Support port 0 and log the actual address for tests. Timeout is a **total deadline per request phase**: one deadline for headers, then a new deadline for the body, then one for writing the response. Valid range: 1–60,000 ms.

The server handles one connection at a time, accepts one request on it, sends one response, and closes. It must survive a malformed client. No persistent connections, chunked request decoding, TLS, HTTP/2, filesystem serving, or pipelined request processing is required. Always send `Connection: close`.

## HTTP subset and parser rules

- Request line: `METHOD SP TARGET SP HTTP/1.1 CRLF`. Parse strictly; require an ASCII token method and an origin-form target beginning with `/`. Other recognizable HTTP versions yield 505.
- Target maximum: 8,192 bytes. Reject fragments (`#`), control characters, and spaces. Match the path before the first `?`; queries are ignored. Do not percent-decode paths.
- Headers end at `CRLF CRLF`. Require CRLF; bare LF is invalid. Limit the entire header section, including request line and terminating delimiter, to 16,384 bytes.
- Require exactly one nonempty `Host` header. For this subset, reject whitespace/control characters in its trimmed value. Header names are case-insensitive valid ASCII tokens. Trim surrounding space/tab from values, reject illegal controls, and reject folded continuation lines.
- No Content-Length means zero request-body bytes. One Content-Length is a nonempty decimal unsigned integer ≤ 1,048,576. Reject signs, overflow, comma lists, or duplicate Content-Length fields, including identical duplicates; this is a deliberately strict subset.
- Any Transfer-Encoding together with Content-Length yields 400. Transfer-Encoding without Content-Length yields 501 and connection close; do not interpret the bytes as another request.
- Any `Expect` header yields 417. Interim 100 responses are out of scope.
- For `POST /echo`, missing Content-Length yields 411; explicit Content-Length 0 is valid.
- Accumulate headers across reads. Bytes already read past the delimiter belong to the body; preserve them. Read exactly the declared body length within its deadline. Extra buffered bytes are ignored because the connection closes after one request.
- Premature EOF is a failed request. Send 400 if the write side remains usable; otherwise close and log the failure. On an expired read deadline, attempt 408 with a separate bounded response-write deadline.
- Check declared body size before allocation. Keep working buffers bounded. A header section exceeding its cap yields 431; excessive target yields 414; excessive body yields 413.

## Routes and response format

| Path | Methods | Status | Content-Type | Exact response body |
| --- | --- | --- | --- | --- |
| `/` | GET, HEAD | 200 | `text/plain; charset=utf-8` | UTF-8 `rust-networking-lab\n`, 20 bytes |
| `/health` | GET, HEAD | 200 | `application/json` | `{"status":"ok"}\n`, 16 bytes |
| `/echo` | POST | 200 | `application/octet-stream` | Exact request bytes |
| Unknown path | GET, HEAD, POST | 404 | `text/plain; charset=utf-8` | `404 Not Found\n` |

For an existing path with a disallowed method, return 405 with `Allow: GET, HEAD` or `Allow: POST` as applicable. For an unknown path with any other method, return 405 with `Allow: GET, HEAD, POST`. Parse and validate framing before routing; do not issue a success response for a malformed body.

Every response has a status line, Content-Type, Content-Length, Connection, and a blank line using CRLF. Header order is not graded. Content-Length counts **body bytes**, not characters. For HEAD, omit the response body while keeping the same Content-Length the corresponding response body would have had, including error responses where HEAD was parsed.

Error bodies are `STATUS SP REASON LF`; reasons are:

| Code | Reason |
| --- | --- |
| 400 | Bad Request |
| 405 | Method Not Allowed |
| 408 | Request Timeout |
| 411 | Length Required |
| 413 | Content Too Large |
| 414 | URI Too Long |
| 417 | Expectation Failed |
| 431 | Request Header Fields Too Large |
| 501 | Not Implemented |
| 505 | HTTP Version Not Supported |

An exact wire response for GET /health, shown with escaped line endings:

```text
HTTP/1.1 200 OK\r\n
Content-Type: application/json\r\n
Content-Length: 16\r\n
Connection: close\r\n
\r\n
{"status":"ok"}\n
```

The displayed `\r\n`/`\n` are notation; transmit actual CR/LF bytes.

## Run and inspect

Run the server in terminal A. In terminal B:

```bash
curl --http1.1 -i http://127.0.0.1:8080/health
curl --http1.1 -I http://127.0.0.1:8080/
curl --http1.1 --data-binary "hello rust" http://127.0.0.1:8080/echo
curl --http1.1 -i http://127.0.0.1:8080/missing
```

The echo body is exactly `hello rust`, with no added newline. Use a raw Rust socket test for malformed headers; curl intentionally produces mostly valid requests.

Server stdout is empty. Stderr JSONL: `listening` with `address`; `request` with `method`, `path`, `status`, `request_body_bytes`, `response_body_bytes`; and `request_error` with `reason`. Body-byte counts are actual transmitted/received bytes, so HEAD response bytes are zero. If method/path are unavailable, log only `request_error`. Ctrl+C may terminate this blocking server.

## Milestones and acceptance criteria

1. **Parser — 5–7 hours.** Write a pure bounded header parser and feed it split/coalesced buffers. Test request-line and header validation before opening a socket.
2. **Bodies and response writer — 5–7 hours.** Preserve already-buffered bytes; support binary echo and byte-accurate lengths.
3. **Listener and deadlines — 4–6 hours.** Add phase deadlines and isolate per-connection failures.
4. **Protocol tests and report — 6–10 hours.** Record raw requests and expected responses.

Required checks: GET / and /health; HEAD has no body; POST echoes non-UTF-8 bytes; /missing yields 404; PUT / yields 405 and Allow; missing/duplicate Host yields 400; body length beyond cap yields 413; absent length on /echo yields 411; malformed/duplicate length yields 400; TE+CL yields 400; unsupported TE yields 501; oversized headers yield 431; and a slow header trickle exceeds the total header deadline rather than resetting it.

Also send two requests together: only the first is processed, response says close, and no second response appears. Verify fragmented header/body reads and a body arriving in the same read as the header delimiter.

## Explanation questions and assessment

1. Why can counting Rust string characters generate an incorrect Content-Length?
2. Explain the distinction between request framing and route semantics.
3. Why do conflicting body-length indicators matter to intermediaries?
4. What does HEAD mean, and why can its Content-Length be nonzero?
5. Why does an absolute deadline resist a slow trickle better than resetting a timeout?
6. Which requirements would you add before calling this a general HTTP server?

**Rubric (100):** parser/framing 35; routes/responses 20; tests/deadlines 20; Rust design 10; report/explanations 15. Extension: persistent connections, with a new test matrix for leftover buffered data and response ordering.

## Targeted reading

- [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html), sections 2–6: HTTP/1.1 syntax and framing.
- [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html), method/status semantics, especially HEAD.
- [Rust Read](https://doc.rust-lang.org/std/io/trait.Read.html) and [Write](https://doc.rust-lang.org/std/io/trait.Write.html).

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

