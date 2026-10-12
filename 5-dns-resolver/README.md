# Assignment 05 — DNS Stub Resolver

**Estimated effort:** 24–36 hours. **Result:** a command-line DNS client that constructs queries, safely decodes responses, and retries over TCP when the UDP reply is truncated.

## Prerequisites and learning objectives

Complete project 03; project 04 is recommended. Understand bit flags, big-endian integer encoding, bounded slice access, offset-based parsing, enums for record types, TCP length prefixes, and absolute deadlines.

Explain stub versus recursive versus authoritative DNS service, record types, the question and answer sections, transaction IDs, compression pointers, TTL, CNAME, and UDP-to-TCP fallback. Although the folder is called a resolver, the required implementation is a **stub client**: it asks one configured recursive server. It does not walk the root/TLD hierarchy.

## CLI and execution

```bash
cargo run --quiet -- --name example.com --type A --server 1.1.1.1:53 --timeout-ms 2000
cargo run --quiet -- --name example.com --type AAAA --server 1.1.1.1:53 --timeout-ms 2000
```

`--name` and `--server` are required. `--type` is A or AAAA, default A. Timeout defaults to 2,000 ms; range 1–60,000 ms. Server must be a numeric IPv4 endpoint. Support `--id U16` for deterministic fixture tests; otherwise generate the 16-bit ID with OS randomness. Never claim that the ID provides cryptographic security.

Names use ASCII hostname labels: letters, digits, interior hyphens, no leading/trailing hyphens; each label is 1–63 bytes. Accept one trailing dot and normalize to lowercase without the trailing dot for output. Reject empty labels, Unicode, underscores, and a wire-encoded name longer than 255 bytes. These are this course's hostname restrictions, not all possible DNS owner names. IDNA is an extension.

Public results change. The reproducible grading target is a local mock server your integration tests implement on `127.0.0.1:15353`.

## Query and parser requirements

Build the query yourself without a DNS library: one question, class IN, type A=1 or AAAA=28, query opcode 0, recursion desired set, and other query flags zero. No EDNS OPT record is required.

Decode the standard header and all counted question/resource-record sections. Output answers; validate and skip supported or unknown authority/additional records using their declared lengths so trailing sections cannot bypass bounds checks.

- Accept only replies from the configured endpoint whose ID, QR, opcode, question count=1, normalized question name, type, and class match the query. Validate lengths before reading. Wrong ID/question/source replies are ignored until the same deadline expires.
- Reject malformed matching replies with exit 1, including invalid record boundaries, reserved label prefixes, out-of-bounds compression pointers, pointer cycles, and excessive expanded name lengths. No unchecked indexing.
- Track name-decoding position separately from the number of bytes consumed at the original offset. A compression pointer ends the encoded name at the original location but may continue decoding elsewhere.
- Limit pointer traversals to 128 and expanded names to 255 wire bytes. Bound response parsing work; cap the total counted records at 4,096.
- Decode A (exactly 4 RDATA bytes), AAAA (exactly 16), and CNAME (bounded name decoding whose bytes consumed at the original RDATA offset must equal RDLENGTH). Unknown types are retained as numeric type plus lowercase hexadecimal RDATA.
- Validate every RR class/TTL/RDLENGTH while parsing; do not assume all answers are IPv4 or have the query type. TTL is an unsigned 32-bit number.
- A matching UDP reply with TC=1 triggers TCP retry to the same IP/port, using the identical query. Require at least a valid matching header/question before this decision; the truncated remainder need not be fully parseable.
- DNS over TCP uses a two-byte big-endian message length. Read the full length-prefixed message, including across partial reads. Reject a TCP response that is malformed, mismatched, or still truncated.
- Use one absolute deadline covering UDP and any TCP fallback. Socket retries must use the remaining time. No automatic retry to another resolver.
- UDP receive cap: 65,535 bytes. TCP encoded length inherently limits the message to 65,535 bytes. A too-short message is an error if it can be identified as a matching response; otherwise ignore it until deadline.

CNAME records may appear in the answer list. **Do not perform new queries to chase aliases** in the required version; display the received records and explain the limitation. Caching, DNSSEC validation, DNS-over-HTTPS/TLS, and iterative resolution are extensions.

## Output contract

Emit one JSON object on a successfully parsed matching response, followed by LF:

```json
{
  "query":{"name":"example.test","type":"A","class":"IN"},
  "server":"127.0.0.1:15353",
  "id":4660,
  "transport":"udp",
  "rcode":0,
  "rcode_name":"NOERROR",
  "authoritative":false,
  "recursion_available":true,
  "truncated":false,
  "answers":[{"name":"example.test","type":"A","class":1,"ttl":60,"data":"192.0.2.10"}]
}
```

`transport` is the final transport, `udp` or `tcp`. Answer order follows the wire. Name strings normalize lowercase without a final dot; the root is `"."`. Known type strings are A/AAAA/CNAME. For unknown types, `type` is the numeric code and `data` is hex without a prefix. A/AAAA data uses standard Rust address display; CNAME data is a normalized name.

`rcode` is the four-bit base response code. Names for 0–5 are NOERROR, FORMERR, SERVFAIL, NXDOMAIN, NOTIMP, REFUSED; others use `UNKNOWN`. NOERROR, including an empty answer list, exits 0. Other valid DNS rcodes still emit the structured result but exit 1. Timeouts, parse failures, or connection errors emit no result and give stderr diagnostics with exit 1.

An empty NOERROR answer list is not NXDOMAIN. It can be reported as an empty list without inventing a nonexistent-host conclusion.

## Milestones and deterministic fixtures

1. **Query encoder — 4–5 hours.** Unit-test a query with fixed ID 0x1234. For `example.test A`, its full bytes must equal:
   `12 34 01 00 00 01 00 00 00 00 00 00 07 65 78 61 6d 70 6c 65 04 74 65 73 74 00 00 01 00 01`.
2. **Safe name/RR parser — 8–12 hours.** Add compressed/uncompressed names, A/AAAA/CNAME/unknown records, and malformed fixtures.
3. **UDP and TCP — 5–7 hours.** Implement source validation, one deadline, truncation fallback, and length-prefixed TCP response reads.
4. **Mock server/report — 7–12 hours.** Integration tests run local UDP and TCP listeners on the same numeric port.

Create and commit fixture bytes or hex files. Required fixture set:

| Fixture | Expected behavior |
| --- | --- |
| A reply: ID 0x1234, flags 0x8180, QDCOUNT=1, ANCOUNT=1 | Reference JSON above |
| Answer owner pointer `c0 0c`, type/class 1/1, TTL 60, RDLENGTH 4, data `c0 00 02 0a` | Safe compressed-name decode |
| AAAA answer `2001:db8::10` | Exact decoded address |
| CNAME plus terminal A in the received answers | Display both; no additional query |
| NXDOMAIN flags with rcode 3 | Structured NXDOMAIN, empty answers, exit 1 |
| NOERROR with no answers | Empty answers, exit 0 |
| Unknown RR type with three data bytes | Numeric type and six hex characters |
| Wrong ID/question then valid response | Ignore wrong reply; accept valid one |
| Compression self-pointer or truncated RDATA | Exit 1; no panic/hang |
| UDP TC=1 then fragmented TCP reply | Final transport tcp, correct answer |
| All replies absent | Exit 1 at deadline; no unbounded retry |

For the exact A fixture, copy the query's question bytes after a response header, then append:
`c0 0c 00 01 00 01 00 00 00 3c 00 04 c0 00 02 0a`.
Do not hardcode public example.com IPs as your acceptance criteria.

## Explanation questions and assessment

1. Why is this a stub client rather than a complete recursive resolver?
2. Explain the two cursor positions needed for compressed-name decoding.
3. Why do ID, question, and endpoint checks all matter?
4. How does NXDOMAIN differ from NOERROR with zero answers?
5. Why is TCP fallback still necessary for a UDP-based query?
6. What would correct CNAME chasing and TTL-aware caching require?
7. What does your tool know about DNSSEC authenticity? What does it not verify?

**Rubric (100):** codec and safe parsing 35; transport/deadlines 20; deterministic tests 25; Rust/errors 10; explanations 10. Extension: bounded CNAME chasing with loop detection and a clearly defined deadline/cache policy.

## Targeted reading

- [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035.html), sections 4.1 and 4.2: message format, names, and transports.
- [RFC 3596](https://www.rfc-editor.org/rfc/rfc3596.html), section 2: AAAA data.
- [RFC 7766](https://www.rfc-editor.org/rfc/rfc7766.html), TCP requirements.
- [RFC 9267](https://www.rfc-editor.org/rfc/rfc9267.html), sections 2–3: unsafe DNS parsing patterns.

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

