# PTF Testing for P4 — Learning Notes

Notes from building automated tests for the p4lang/tutorials exercises.

## What is PTF?

PTF (Packet Test Framework) is a Python-based testing framework for data plane programs. It lets you send packets into a software switch, then verify what comes out the other side. Think of it as unit testing but for network behavior — you craft a specific input packet, send it, and assert that the output packet has the right headers, lands on the right port, or gets dropped as expected.

## How a PTF Test File is Structured

A PTF test for a P4 program follows a consistent pattern:

### 1. Imports and Dependencies

```python
import ptf
import ptf.testutils as testutils
from ptf.base_tests import BaseTest
from p4runtime_sh.shell import setup, teardown, TableEntry
```

- `ptf` — the core test framework, gives you packet send/receive/verify primitives
- `ptf.testutils` — helper functions like `simple_tcp_packet()` to craft common packet types and `send_packet()` / `verify_packet()` to push packets through the switch and check output
- `BaseTest` — the parent class all tests inherit from. Handles setup and teardown of the test environment
- `p4runtime_sh` — P4Runtime shell utilities for programming table entries into the switch at runtime

### 2. Logger Setup

```python
import logging
logger = logging.getLogger('BasicFwdTest')
logger.addHandler(logging.StreamHandler())
```

Logging matters because when a test fails, you need to see what packet was sent, what was expected, and what actually came out. Without logs, debugging test failures is guesswork.

### 3. Test Class

Each test is a class that inherits from `BaseTest`. The class has:

- `setUp()` — runs before the test. This is where you program table entries into the switch using P4Runtime. For example, adding an IPv4 forwarding rule that maps a destination IP to an output port with specific MAC rewrites.
- The test method itself — crafts an input packet, sends it into a specific port, and verifies the output.
- Assertions — check that the packet arrived on the expected port, with the expected header modifications (MAC rewrite, TTL decrement, etc.), or that it was dropped when it should have been.

### 4. The Tests Themselves

Each test targets one specific behavior of the P4 program:

- **DropTest** — send a packet with no matching table entries installed. The default action should drop it. Verify nothing comes out on any port.
- **FwdTest** — install a single forwarding entry, send a matching packet, verify it exits on the correct port with the right MAC addresses and decremented TTL.
- **MultiEntryTest** — install multiple LPM entries, send packets matching different prefixes, verify each one routes to the correct port.

The idea is to start with the simplest case (does the default action work?) and build up to more complex scenarios. Each test should be independent — it sets up its own table entries and doesn't depend on state from other tests.

## The runptf.sh Script

The test script automates the full cycle:

1. Compile the solution P4 program with `p4c`
2. Start `simple_switch_grpc` with the compiled JSON
3. Run the PTF tests against the running switch
4. Clean up processes and build artifacts

Tests run against the **solution** files, not the starter code with TODOs. This way they serve as regression tests — if a future compiler update or repo change breaks the expected behavior, the tests catch it.

## Key Libraries

| Library | What it does |
|---|---|
| `ptf` | Core test framework — send/receive/verify packets |
| `ptf.testutils` | Helpers for crafting standard packets (TCP, UDP, etc.) and asserting output |
| `scapy` | Packet construction and parsing under the hood — PTF uses it internally |
| `p4runtime_sh` | Python bindings for P4Runtime — used to program table entries into the switch |
| `grpc` | Transport layer for P4Runtime communication with `simple_switch_grpc` |

## Dependencies That Live Outside the Tutorials Repo

One thing I ran into is that some testing utilities aren't included in the tutorials repo itself:

- `p4runtime_shell_utils.py` lives in [jafingerhut/p4-guide/testlib](https://github.com/jafingerhut/p4-guide/tree/master/ptf-tests) — the test script currently hardcodes a path to this
- `veth` interface setup requires an external script to create the virtual Ethernet pairs that connect test ports to the switch

These are open questions I flagged in the PR for maintainer guidance. In a self-contained test setup, these dependencies would either be vendored into the repo or documented as prerequisites with clear setup instructions.

## What I Took Away

Writing PTF tests forced me to think about the P4 program from the outside in. Instead of "does this compile?" it's "does a packet with destination 10.0.1.1 actually come out on port 1 with the right MAC?" That's a fundamentally different kind of verification — you're testing the behavior, not just the syntax.

The pattern is transferable to other exercises. Each P4 tutorial exercise has a solution with defined expected behavior. The same structure — set up table entries, craft packets, verify output — applies to basic_tunnel, ecn, firewall, and the rest. The main thing that changes is which tables you program and what header fields you assert on.
