# Assignment 01 — IP and Subnet Explorer

**Estimated effort:** 8–12 hours. **Result:** a deterministic command-line tool that explains an IPv4 subnet, checks membership, and splits it into smaller subnets.

## Prerequisites and learning objectives

Before starting, understand Rust functions, `u8`/`u32`/`u64`, arrays, structs, enums, `Option`, `Result`, `match`, borrowed strings, and unit tests. Read Rust Book chapters 3–6 and 9, plus the testing material in chapter 11.

By completion, you must be able to explain an IPv4 address as 32 bits, CIDR prefix length, network and host bits, bitwise masks, network byte order, and why an address is not itself a route. You should also distinguish the default gateway from DHCP: a gateway routes traffic; DHCP may assign configuration.

This is an **offline IPv4 calculation tool**. It does not discover interfaces, configure routes, send packets, or implement IPv6. Do the subnet arithmetic yourself; using `std::net::Ipv4Addr` for address parsing and display is allowed.

## Required command-line interface

| Command | Required flags | Optional flags |
| --- | --- | --- |
| `inspect` | `--cidr IPv4/PREFIX` | None |
| `contains` | `--cidr IPv4/PREFIX --ip IPv4` | None |
| `split` | `--cidr IPv4/PREFIX --new-prefix N` | None |

Examples, from this directory:

```bash
cargo run --quiet -- inspect --cidr 192.168.10.37/24
cargo run --quiet -- contains --cidr 192.168.10.37/24 --ip 192.168.10.200
cargo run --quiet -- split --cidr 192.168.10.37/24 --new-prefix 26
```

All successful commands output **one JSON object**, followed by LF. Parse dotted-decimal IPv4 through `Ipv4Addr`. Accept prefix 0–32; reject whitespace inside the CIDR, missing or additional slash characters, IPv6 input, and invalid octets. Host bits in the supplied address are allowed and normalized.

## Calculation and output specification

`inspect` must produce these fields; this example is exact except for JSON key order:

```json
{
  "input": "192.168.10.37/24",
  "network": "192.168.10.0",
  "prefix": 24,
  "netmask": "255.255.255.0",
  "wildcard": "0.0.0.255",
  "last_address": "192.168.10.255",
  "broadcast": "192.168.10.255",
  "total_addresses": 256,
  "usable_hosts": 254,
  "first_usable": "192.168.10.1",
  "last_usable": "192.168.10.254",
  "host_policy": "conventional"
}
```

- Network = address AND mask. Wildcard = the 32-bit inverse of mask. Last address = network OR wildcard.
- Use `u64` for address counts: `2^(32-prefix)`. A /0 has 4,294,967,296 addresses; it must not overflow.
- /0 through /30: apply the **conventional arithmetic policy**, reserve the first and last addresses, and label `host_policy` as `"conventional"`. This is a calculation rule, not a claim that every address in the block is globally assignable.
- /31: explicitly adopt the point-to-point policy; both addresses are usable, `broadcast` is null, and `host_policy` is `"point_to_point"`.
- /32: one host route, one usable address, `broadcast` is null, and `host_policy` is `"host_route"`.
- First/last usable are strings for every valid prefix under these policies. `last_address` is always a string.
- Handle /0 without shifting a 32-bit value by 32. Use checked arithmetic or an explicit /0 branch.

`contains` returns normalized CIDR, supplied IP, and a boolean:

```json
{"cidr":"192.168.10.0/24","ip":"192.168.10.200","contains":true}
```

Membership includes every address in the numeric block, including conventional network/broadcast addresses. It does not test reachability or host usability.

`split` returns:

```json
{"parent":"192.168.10.0/24","new_prefix":26,"count":4,"subnets":["192.168.10.0/26","192.168.10.64/26","192.168.10.128/26","192.168.10.192/26"]}
```

Require old prefix ≤ new prefix ≤ 32. Equal prefix returns one normalized block. Emit subnets in increasing address order. Reject a split yielding more than 4,096 blocks with exit 2 before allocating the result.

## Suggested design and milestones

1. **Address representation — 2 hours.** Parse input into a struct containing `Ipv4Addr` and prefix. Implement address-to-`u32` and back using explicit big-endian bytes. Verify round trips for zero, loopback, and maximum address.
2. **Inspect — 3 hours.** Write pure mask/range functions. Implement /0, /31, and /32 before adding the CLI. Completion: the reference JSON matches.
3. **Membership and splitting — 2–3 hours.** Keep these operations pure; test parent normalization and overflow boundaries.
4. **CLI, reporting, and defense — 1–4 hours.** Add serialization, errors, integration tests, and explanations.

Suggested modules: `address`, `subnet`, and `cli`. Standard library plus a JSON serialization crate is sufficient; argument-parsing crates are optional. Do not use a subnet-calculation crate for the required arithmetic.

## Acceptance tests

| Case | Required result |
| --- | --- |
| `192.168.10.37/24` | Reference inspect output above |
| `10.0.0.7/30` | Network .4, last/broadcast .7, usable .5–.6, count 4 |
| `10.0.0.4/31` | Two usable endpoints .4–.5; null broadcast |
| `127.0.0.1/32` | Network/first/last all 127.0.0.1, count 1 |
| `0.0.0.0/0` | Mask 0.0.0.0; count 4,294,967,296 |
| /24 contains its .0 and .255 | Both true; membership is not usability |
| /24 contains the next /24's first address | False |
| /24 split into /26 | Four exact ordered blocks above |
| Split /24 into /23, or /0 into /32 | Exit 2, useful diagnostic |
| `300.1.1.1/24`, /33, missing slash | Exit 2; no panic |

Add a property-style loop over all prefixes and a representative address set: every normalized network belongs to its block; round trips preserve the input; split children are disjoint and together cover the parent. You do not need a property-testing dependency.

## Explanation questions and assessment

Answer with a worked binary example, not only definitions:

1. Derive the /26 mask and locate 192.168.10.130 within its normalized subnet.
2. Why is `u8` enough for an octet but `u32` inadequate for the /0 address count?
3. What changes between membership, usable-host arithmetic, and actual reachability?
4. What is special about /31? Under what link assumption is your output appropriate?
5. Why does converting via big-endian bytes make the arithmetic predictable?
6. How does `Result` distinguish malformed input from a valid `false` membership result?

**Rubric (100 points):** arithmetic and boundary correctness 40; CLI/schema 15; tests 20; Rust design/errors 10; explanations/report 15. Optional extension: IPv6 prefix membership using `u128`; do not subtract a broadcast address from IPv6 ranges.

## Targeted reading

- [Rust Ipv4Addr](https://doc.rust-lang.org/std/net/struct.Ipv4Addr.html).
- [RFC 3021](https://www.rfc-editor.org/rfc/rfc3021.html), sections 2.1–2.2: the /31 point-to-point exception.
- [Rust Book](https://doc.rust-lang.org/book/), chapters 3–6, 9, and 11.

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

