# Assignment 07 — Offline Packet Analyzer

**Estimated effort:** 24–36 hours. **Result:** an offline classic-PCAP parser that decodes Ethernet, IPv4, TCP, and UDP into reproducible JSON, without requiring administrator access to run.

## Prerequisites and learning objectives

Complete projects 01–05; project 06 is useful. Understand byte offsets, bit fields, endianness, checked length arithmetic, nested protocol layers, file streaming, and bounded parsers.

Explain encapsulation, capture length versus original length, Ethernet versus IP addresses, IPv4 header length/fragmentation, TCP flags, UDP length, and what a packet trace can and cannot prove.

The required tool reads files only. Live capture, TCP reassembly, TLS decryption, IPv6, VLANs, pcapng, Wi-Fi link types, Linux cooked captures, and application decoding are extensions. Do not silently treat every capture as Ethernet. Do not use a packet-decoding crate or PCAP parser for the required implementation.

## CLI and run contract

```bash
cargo run --quiet -- --input fixtures/three-packets.pcap --packets out/packets.jsonl
```

Both flags are required. Input is a regular file. Create missing output parent directories. Refuse to overwrite an existing packets file. Stream the input and output; never load the entire capture into memory. Successful stdout is one summary JSON object; individual packet results go only to the specified JSONL file.

The fixture mentioned above is a file **you create and commit** at milestone 1, not a file supplied with this specification.

## Capture-format scope and limits

Support classic PCAP version 2.4 with Ethernet link type exactly 1, including both byte orders and both timestamp resolutions:

| Initial four bytes | Byte order | Timestamp fraction |
| --- | --- | --- |
| `d4 c3 b2 a1` | Little-endian | Microseconds |
| `a1 b2 c3 d4` | Big-endian | Microseconds |
| `4d 3c b2 a1` | Little-endian | Nanoseconds |
| `a1 b2 3c 4d` | Big-endian | Nanoseconds |

Parse the 24-byte global header and each 16-byte record header using the selected order. Accept snaplen 1–1,048,576 bytes. Record captured length must not exceed snaplen, original length, or 1,048,576 bytes. Timestamp fractional values must be below one million or one billion as appropriate. Accept at most one million records; report a resource-limit failure before record 1,000,001. Empty valid captures are allowed.

Reject unsupported magic/version/link type explicitly, including pcapng. Ignore legacy timezone/accuracy fields after reading them; do not use them to shift timestamps. This course's link-type restriction excludes extra high-bit metadata flags.

A partial global/record header or missing declared captured bytes is a **corrupt file**, yielding exit 1 with byte offset and no success summary. Close the output; it may contain already-processed rows, and the failure diagnostic must label it partial. It is not the same as a complete record whose original packet was longer than the captured snapshot.

## Layer decoding and classification

Every complete capture record produces exactly one JSONL row. Classify it as `decoded`, `unsupported`, `truncated`, or `malformed`.

- Require 14 captured bytes for an Ethernet header. Decode destination/source MAC and EtherType. Format MACs as six lowercase hex octets separated by colons.
- EtherType 0x0800 proceeds to IPv4. Other EtherTypes, including IPv6 and VLAN tagging, are `unsupported`; do not guess their payload layout.
- IPv4: validate version 4, IHL ≥ 5, total length ≥ header length, and length relationships before accessing options/transport. Ignore Ethernet padding past IPv4 total length.
- If the needed bytes are absent and captured length < original length, classify `truncated`. If they are absent despite a fully captured record, classify `malformed`. Invalid declared fields are malformed regardless of snapshot length.
- Require the full declared IPv4 datagram to classify it decoded. Decode source/destination, protocol number, TTL, total length, and header length. Header checksum validation is an extension; do not claim it was verified.
- Any nonzero fragment offset or MF=1 means a fragmented datagram. Decode IPv4 metadata, set `transport` to `"FRAGMENT"`, ports to null, and do not parse TCP/UDP even for the initial fragment. No reassembly is required.
- For unfragmented TCP (protocol 6), require 20 bytes and a data offset ≥ 5 whose header length fits the IP payload. Decode ports, sequence, acknowledgment, and the eight standard flag bits. Emit flags in order FIN, SYN, RST, PSH, ACK, URG, ECE, CWR when set.
- For unfragmented UDP (protocol 17), require 8 bytes; declared UDP length must be at least 8 and no larger than the IP payload. Decode ports and length.
- Other valid IPv4 protocols use `transport:"OTHER"`, null ports, and preserve the numeric protocol. They remain decoded IPv4 records.
- Derive payload sizes from protocol lengths, not original frame length. Do not dump application payload bytes.

When an individual packet is malformed, emit its error row, continue processing other complete records, and make the overall exit code 1 after emitting the final summary. Unsupported/truncated packets alone do not make the run fail.

## Output schemas

Every row has `index` (one-based), `timestamp_ns` (integer), `captured_len`, `original_len`, and `status`. Convert timestamps to nanoseconds exactly with `u64`.

Decoded rows also have `ethernet` and `ipv4`; `tcp`/`udp` appear only for those transports. Example synthetic TCP SYN:

```json
{"index":1,"timestamp_ns":1000000000,"captured_len":54,"original_len":54,"status":"decoded","ethernet":{"source":"02:00:00:00:00:01","destination":"02:00:00:00:00:02","ethertype":2048},"ipv4":{"source":"192.0.2.1","destination":"192.0.2.2","protocol":6,"ttl":64,"header_bytes":20,"total_bytes":40,"transport":"TCP","source_port":50000,"destination_port":80},"tcp":{"sequence":1000,"acknowledgment":0,"flags":["SYN"],"header_bytes":20,"payload_bytes":0}}
```

UDP object: `length_bytes`, `payload_bytes`. OTHER/FRAGMENT have neither TCP nor UDP object. Nondecoded rows add a short stable `reason` string; decoded metadata for those rows is optional and not graded.

Summary:

```json
{"packets_total":3,"decoded":3,"unsupported":0,"truncated":0,"malformed":0,"tcp":2,"udp":1,"other_ipv4":0,"fragmented_ipv4":0,"captured_bytes":178,"original_bytes":178,"first_timestamp_ns":1000000000,"last_timestamp_ns":3000000000}
```

Counters are record counts. `decoded = tcp + udp + other_ipv4 + fragmented_ipv4`; total equals decoded+unsupported+truncated+malformed. Byte sums include every record. First/last timestamps are those of the first/last records in **file order**, not min/max; use null for both in an empty capture.

## Deterministic fixture and milestones

1. **Capture reader and fixtures — 5–7 hours.** Build a small fixture generator in Rust tests or `examples/make_fixture.rs`. Commit a valid little-endian microsecond Ethernet capture with:
   - At 1 s: 54-byte TCP SYN 192.0.2.1:50000 → 192.0.2.2:80, sequence 1000, TTL 64, MACs matching the example.
   - At 2 s: 54-byte reverse SYN+ACK, sequence 2000, acknowledgment 1001.
   - At 3 s: 70-byte Ethernet/IPv4/UDP packet 192.0.2.1:53000 → 192.0.2.53:53 with UDP length 36 and 28 opaque payload bytes.
   Use valid IPv4 checksums so independent tools recognize the fixture; UDP zero checksum is allowed for this IPv4 fixture. Payload decoding is not required.
2. **Protocol parsers — 7–10 hours.** Add pure Ethernet/IP/TCP/UDP decoders and bounds tests.
3. **Rows, counters, and CLI — 4–6 hours.** Match both example outputs and stream results.
4. **Independent comparison/report — 8–13 hours.** Inspect the fixture in Wireshark; compare fields, not pretty-printer formatting.

Acceptance tests: all four magic encodings; exact three-packet counters; IPv4 options and TCP options; Ethernet padding; UDP length below 8; TCP data offset below 5; malformed IPv4 lengths; initial/later fragments counted as fragments; unsupported VLAN/IPv6 EtherType; truncated snapshot counted separately; corrupt record container fails; empty valid capture; unsupported pcapng/link type; record length exceeding cap; and nonmonotonic timestamps preserved in file order.

Optional real-capture evidence may be obtained with Wireshark, but save **classic PCAP with Ethernet link type**. Loopback captures often use other link formats; they are not automatically valid inputs to this restricted parser.

## Explanation questions and assessment

1. Why is PCAP byte order independent of network byte order inside packets?
2. What does captured length smaller than original length tell you?
3. Why are IP and TCP header lengths variable?
4. Why can a fragment prevent safe transport-header decoding?
5. Why does a SYN usually carry a sequence number even without application payload?
6. Can observing one SYN prove a completed connection? What extra evidence is needed?
7. Why does encrypted payload not hide all endpoint/timing information?

**Rubric (100):** capture-format parsing 20; layered decoding/bounds 30; fixtures/comparison/tests 25; CLI/schema 10; explanations 15. Extension: IPv6 with explicit extension-header handling and new fixtures, or TCP flow summaries without claiming full stream reassembly.

## Targeted reading

- [libpcap savefile format](https://github.com/the-tcpdump-group/libpcap/blob/master/pcap-savefile.manfile.in) and [libpcap source](https://github.com/the-tcpdump-group/libpcap): capture container.
- [RFC 791](https://www.rfc-editor.org/rfc/rfc791.html), section 3.1: IPv4 format.
- [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html), section 3.1: TCP header.
- [RFC 768](https://www.rfc-editor.org/rfc/rfc768.html): UDP header.

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

