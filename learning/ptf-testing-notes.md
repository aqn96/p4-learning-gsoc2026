# PTF Testing for P4 — Learning Notes

Notes from building automated tests for the p4lang/tutorials exercises.

## What is PTF?

PTF (Packet Test Framework) is a Python-based testing framework for data plane programs. It lets you send packets into a software switch, then verify what comes out the other side. Think of it as unit testing but for network behavior — you craft a specific input packet, send it, and assert that the output packet has the right headers, lands on the right port, or gets dropped as expected.

## The Full Testing Flow

When you run `make test`, here's what happens end to end:

```
make test
  → calls runptf.sh
    → creates veth pairs (virtual ethernet interfaces)
    → compiles solution P4 program with p4c
    → starts simple_switch_grpc (BMv2 software switch)
    → runs PTF tests (Python)
      → each test:
        1. P4InfoHelper reads compiled metadata (translates names → numeric IDs)
        2. Connects to switch over gRPC
        3. Claims master controller role (MasterArbitrationUpdate)
        4. Pushes compiled P4 program onto the switch
        5. Writes table entries (match-action rules)
        6. Crafts a packet and sends it into a port
        7. Asserts what comes out (correct port, correct headers, or dropped)
        8. Closes connections
    → kills the switch process
    → deletes veth interfaces
    → done
```

One command, fully self-contained. No external dependencies.

## How a PTF Test File is Structured

### 1. Imports and Dependencies

```python
import os
import sys
import ptf
import ptf.testutils as tu
from ptf.base_tests import BaseTest

# Import p4runtime_lib from the tutorials repo
sys.path.append(
    os.path.join(os.path.dirname(os.path.abspath(__file__)),
                 '../../../utils/'))
import p4runtime_lib.bmv2
import p4runtime_lib.helper
from p4runtime_lib.switch import ShutdownAllSwitchConnections
```

- `ptf` — core test framework, gives you packet send/receive/verify primitives
- `ptf.testutils` — helpers like `simple_tcp_packet()` to craft packets and `send_packet()` / `verify_packets()` to push packets through the switch and check output
- `BaseTest` — parent class all tests inherit from. Handles setup and teardown
- `p4runtime_lib.helper` — `P4InfoHelper` reads the p4info file and translates human-readable names (like `MyIngress.ipv4_lpm`) into the numeric IDs that P4Runtime uses internally
- `p4runtime_lib.bmv2` — `Bmv2SwitchConnection` manages the gRPC connection to the software switch
- `ShutdownAllSwitchConnections` — cleanup function that closes all open gRPC connections
- `sys.path.append` — needed because `p4runtime_lib` lives in `utils/` which is three directories up from the test file. Python only imports from directories it knows about, so you have to explicitly add the path

### 2. Logger Setup

```python
import logging
logger = logging.getLogger(None)
ch = logging.StreamHandler()
ch.setLevel(logging.INFO)
logger.addHandler(ch)
```

When a test fails, you need to see what packet was sent, what was expected, and what actually came out. Without logs, debugging test failures is guesswork.

### 3. Test Base Class — setUp and tearDown

```python
class BasicFwdTest(BaseTest):
    def setUp(self):
        # Step 1: Create P4InfoHelper — reads compiled metadata,
        # translates human-readable table/action names to numeric IDs
        self.p4info_helper = p4runtime_lib.helper.P4InfoHelper(p4info_txt_fname)

        # Step 2: Connect to the switch over gRPC
        self.sw = p4runtime_lib.bmv2.Bmv2SwitchConnection(
            name='s1',
            address=grpc_addr,
            device_id=0)

        # Step 3: Claim master controller role
        # P4Runtime supports multiple controllers, but only the master
        # can write. This isn't about which switch is "master" in the
        # network — it's about which controller has write access.
        self.sw.MasterArbitrationUpdate()

        # Step 4: Push the compiled P4 program onto the switch
        # Two things get sent: the p4info (metadata) and the BMv2 JSON (the program)
        self.sw.SetForwardingPipelineConfig(
            p4info=self.p4info_helper.p4info,
            bmv2_json_file_path=p4prog_binary_fname)

    def tearDown(self):
        ShutdownAllSwitchConnections()
```

Each test class inherits from this base. setUp runs before every test, tearDown runs after. The switch gets a fresh connection and program load for each test.

### 4. Helper Function — Writing Table Entries

```python
def add_ipv4_lpm_entry(self, ipv4_addr_str, prefix_len, dst_mac_str, port):
    # buildTableEntry creates a protobuf message — a structured binary
    # object that gets serialized and sent over gRPC to the switch
    table_entry = self.p4info_helper.buildTableEntry(
        table_name='MyIngress.ipv4_lpm',
        match_fields={'hdr.ipv4.dstAddr': (ipv4_addr_str, prefix_len)},
        action_name='MyIngress.ipv4_forward',
        action_params={'dstAddr': dst_mac_str, 'port': port})

    # WriteTableEntry sends the protobuf to the switch, which installs the rule
    self.sw.WriteTableEntry(table_entry)
```

This is a class method (takes `self`) because it needs `self.p4info_helper` to build the entry and `self.sw` to write it. The match field is a tuple `(address, prefix_len)` because LPM matches need both.

### 5. The Tests Themselves

Each test targets one specific behavior of the P4 program:

- **DropTest** — send a packet with no matching table entries installed. The default action should drop it. Verify nothing comes out on any port.
- **FwdTest** — install a single forwarding entry, send a matching packet, verify it exits on the correct port with the right MAC addresses and decremented TTL.
- **MultiEntryTest** — install multiple LPM entries, send packets matching different prefixes, verify each one routes to the correct port.

The idea is to start with the simplest case (does the default action work?) and build up to more complex scenarios. Each test is independent — it sets up its own table entries and doesn't depend on state from other tests.

## The runptf.sh Script

The script automates the full lifecycle:

1. **Create veth pairs** — virtual ethernet interfaces that connect test ports to the switch. Each pair has two ends (e.g., veth0/veth1). The switch binds to even-numbered interfaces, PTF sends/receives on odd-numbered ones.
2. **Compile** the solution P4 program with `p4c` → outputs JSON and p4info
3. **Start** `simple_switch_grpc` bound to the veth interfaces, with `--no-p4` flag (the program gets loaded via P4Runtime in the test setUp)
4. **Run PTF tests** — each test connects to the switch, loads the program, writes entries, sends packets, verifies output
5. **Kill** the switch process
6. **Delete** the veth interfaces

The Makefile has a `test` target that calls this script, so users just type `make test`.

## Key Libraries

| Library | What it does |
|---|---|
| `ptf` | Core test framework — send/receive/verify packets |
| `ptf.testutils` | Helpers for crafting standard packets (TCP, UDP, etc.) and asserting output |
| `scapy` | Packet construction and parsing under the hood — PTF uses it internally |
| `p4runtime_lib` | Tutorials repo's own P4Runtime library — connects to switches, builds and writes table entries |
| `grpc` | Transport layer for P4Runtime communication with `simple_switch_grpc` |

## Resolved Dependency Issues

My initial approach used `p4runtime_sh` and `p4runtime_shell_utils` from an external repo. After review feedback from the maintainers, I refactored to use `p4runtime_lib` which already exists in the tutorials repo at `utils/p4runtime_lib/`. The veth setup is now handled inside `runptf.sh` itself. No external dependencies needed.

The two libraries do the same thing — talk to a switch over P4Runtime — but with different Python APIs:

```python
# OLD: p4runtime_sh (external package, dictionary-style API)
te = sh.TableEntry('MyIngress.ipv4_lpm')(action='MyIngress.ipv4_forward')
te.match['hdr.ipv4.dstAddr'] = '10.0.1.1/32'
te.insert()

# NEW: p4runtime_lib (tutorials repo, explicit build + write)
te = p4info_helper.buildTableEntry(table_name='MyIngress.ipv4_lpm', ...)
sw.WriteTableEntry(te)
```

Lesson learned: before pulling in external libraries, check what the repo already provides. The tutorials repo had its own P4Runtime library the whole time — I just didn't know where to look until a maintainer pointed it out.

## CI Lessons from PR #730

After wiring this into GitHub Actions, I realized most of the hard part wasn't YAML syntax — it was environment assumptions.

### Questions I kept asking while debugging

1. **Will this run the same way on my machine and on GitHub runners?**  
   Not automatically. Network-heavy tests (veth creation, BMv2 processes) are sensitive to container privileges and host architecture.

2. **Do I trust an image name because it sounds right, or because I verified it exists?**  
   I initially used an image name that looked plausible but did not exist publicly. Better workflow: verify first with `docker pull <image>` before committing workflow config.

3. **Should CI call custom commands, or the same command I use locally?**  
   Same command. The most stable approach was making `make test` the single source of truth and calling it in both places.

4. **How do I make failures useful to reviewers?**  
   Save logs as artifacts. When CI fails, BMv2/P4Runtime logs are often the fastest way to identify if it's a dependency issue, permission issue, or test logic issue.

### Practical CI notes I want to remember

- Use `pull_request` triggers so checks run on draft PR iterations, not just post-merge pushes.
- Keep networking permissions explicit (`--privileged` in the container config when veth manipulation is required).
- Local `act` runs are useful for quick iteration, but on Apple Silicon I should expect architecture friction (`linux/amd64` emulation) and treat GitHub runner results as the final signal.


## What I Took Away

Writing PTF tests forced me to think about the P4 program from the outside in. Instead of "does this compile?" it's "does a packet with destination 10.0.1.1 actually come out on port 1 with the right MAC?" That's a fundamentally different kind of verification — you're testing the behavior, not just the syntax. It also made me think more systematically about the structure of the P4 program itself — what the expected default behavior is, what each table entry is supposed to do, and where the edge cases or gaps might be (e.g., what happens when TTL hits 0, or when a packet arrives with a non-IPv4 etherType).

The pattern is transferable to other exercises. Each P4 tutorial exercise has a solution with defined expected behavior. The same structure — set up table entries, craft packets, verify output — applies to basic_tunnel, ecn, firewall, and the rest. The main thing that changes is which tables you program and what header fields you assert on.
