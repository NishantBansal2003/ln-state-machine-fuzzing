# State Machine Fuzzing Guide

These are my notes on how I write state machine fuzz tests for Lightning
implementations, plus the commands I use to actually run them and pull a
coverage report afterwards.

The document is split into two parts: the [general approach](#how-i-approach-it),
and then the [per-implementation commands](#running-it-per-implementation)
for LND, LDK, CLN, Eclair, as well as the [cross-implementation](#running-it-cross-implementation)
setup with Smite.

## How I approach it

The point of this kind of fuzzing is to push a subsystem through its states by
feeding it wire messages and seeing what falls over: crashes, panics, OOMs,
blown assertions, or invariants that should never break but do.

1. **Start by listing the states and how you move between them.** Think about this
   from the receiving side, not the sending side. The peer controls the bytes on
   the wire, so that's the surface that actually matters. A few messages can only
   be fully processed if the node sends something back first (a handshake reply, or
   a query that has to be answered before the next message is accepted), so pull
   those send steps in too, but only because you need them to keep the receive path
   moving. For gossip, say, you'd end up with three states, one per inbound message you
   care about: `channel_announcement`, `channel_update`, and `node_announcement`.

2. **Then find where the message actually lands.** Follow it through the code. The
   transport layer eventually hands the decoded message off to some subsystem,
   gossip manager, channel state machine, the HTLC switch, whatever. That handoff
   point is what you target. In the harness you wire-decode the fuzzer bytes into a
   message and feed it straight to that subsystem, skipping the network entirely.

3. **Try to keep the target narrow.** If processing is synchronous and
   deterministic you get way better throughput and crashes are trivial to
   reproduce. That said, going wider and pulling in more of the lower-level stack
   does surface bugs that only show up when layers interact, and honestly a few
   implementations only break in the wider setup. The cost is speed. If the wider
   version still fuzzes fast enough, use it; if it tanks your exec/sec, drop back
   down to the smallest piece that still hits the logic you care about. It depends
   on the state being fuzzed.

4. **Add some checks at the end.** What you assert depends on what you're after. If
   you want logical bugs, check the invariants that should always hold, and check
   whether an error came back and whether it was the error you expected for that
   message (that's how you catch accept/reject regressions). If you just want to
   cast a wide net, lean on the crash oracle: panics, OOMs, sanitizer aborts.
   Tighter assertions find more logic bugs, looser ones explore more.

5. **Let it run, then check coverage.** This stuff really only pays off if you let
   it run for a good while, a couple of days continuously, then generate coverage
   from the corpus it built up and see whether you actually reached every state. If
   some state never got hit, either it needs more time, or it's unreachable because
   your harness is hitting a constraint it never satisfies. If it's the harness,
   fix the test and run it again. Keep going until coverage looks right. Once it
   does, you've got a state machine fuzz test that's actually worth pointing at
   real bugs.

## Running it per-implementation

Here's how to run some fuzz tests, pull a coverage report and reproduce a crash
from the target. Please note that there's a lot more to each engine than this,
more flags, more commands. I've only pulled out the handful I reach for most.
For everything else I've linked each implementation's own fuzzing doc, so go dig
into those for the engine-specific details.

### LND

- **Language:** Go
- **Fuzzing engine:** Go native fuzzing engine
- **Fuzzing docs:**
  - https://go.dev/doc/security/fuzz
- **Run continuously:** fuzz `FuzzGossipStateMachine` on 8 workers:
  ```shell
  go test ./discovery \
    -fuzz='^FuzzGossipStateMachine$' \
    -run='^$' \
    -test.fuzzcachedir='./testdata/fuzz' \
    -parallel 8
  ```
- **Corpus location:** `$PWD/discovery/testdata/fuzz/FuzzGossipStateMachine/*`
- **Coverage report:** build a profile from the collected corpus, then render
  it to HTML:
  ```shell
  go test ./discovery \
    -run='FuzzGossipStateMachine' \
    -coverprofile=coverage.out \
    -covermode=count \
    -timeout 6000s
  go tool cover -html=coverage.out -o coverage.html
  ```
  The `-timeout` matters once the corpus gets large: the coverage run replays
  every input, so a big corpus can blow past the default timeout. Bump it up (or
  drop the flag entirely while the corpus is still small).
- **Reproduce a crash:** LND uses `log.go` for subsystem-specific logging. Enable
  stdout logging in that file to inspect failure logs from the relevant LND subsystem:
  `go test ./discovery -run=FuzzGossipStateMachine/<fail-filename> -v`

### Eclair

- **Language:** Scala
- **Fuzzing engine:** Jazzer based on `libFuzzer`
- **Fuzzing docs:**
  - https://github.com/CodeIntelligenceTesting/jazzer
  - https://github.com/ACINQ/eclair/blob/master/eclair-fuzz/README.md
- **Run continuously:**
  ```shell
  # Rebuild eclair-core before fuzzing new changes.
  ./mvnw clean install -pl eclair-core -am -DskipTests
  JAZZER_FUZZ=1 ./mvnw test -f eclair-fuzz/pom.xml -Dtest=<TestClass>#<fuzzMethod>
  ```
- **Corpus location:** `$PWD$/eclair-fuzz/.cifuzz-corpus/<Package>.<TestClass>/<fuzzMethod>`
- **Coverage report:** To generate an HTML coverage report, download and unzip a [JaCoCo](https://github.com/jacoco/jacoco/releases) release, then run:
  ```shell
  ./eclair-fuzz/generate-coverage-report.sh /path/to/jacoco/lib
  ```
- **Reproduce a crash:** Replay crash inputs only (all fuzz targets)
  `./mvnw test -f eclair-fuzz/pom.xml`
