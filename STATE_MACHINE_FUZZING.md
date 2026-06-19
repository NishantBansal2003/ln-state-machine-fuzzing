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

### LDK

- **Language:** Rust
- **Fuzzing engine:** `honggfuzz`, `libFuzzer` and `AFL`
- **Fuzzing docs:**
  - https://github.com/lightningdevkit/rust-lightning/blob/main/fuzz/README.md
- **Run continuously:** I currently use `libFuzzer` most of the time, so the
  commands below focus on that. For honggfuzz and AFL, refer to the LDK fuzzing
  documentation. fuzz the `gossip_discovery_target` target:
  ```shell
  cd fuzz
  export RUSTFLAGS="--cfg=fuzzing --cfg=secp256k1_fuzz --cfg=hashes_fuzz"
  cargo +nightly fuzz run --fuzz-dir fuzz-fake-hashes --features "libfuzzer_fuzz" gossip_discovery_target
  ```
- **Corpus location:** `$PWD/fuzz/{fuzz-dir(mentioned above)}/corpus/gossip_discovery_target`
- **Coverage report:**
  ```shell
  cargo +nightly fuzz coverage --fuzz-dir fuzz-fake-hashes --features "libfuzzer_fuzz" gossip_discovery_target
  llvm-cov show target/aarch64-apple-darwin/coverage/aarch64-apple-darwin/release/gossip_discovery_target \
  	-instr-profile=fuzz-fake-hashes/coverage/gossip_discovery_target/coverage.profdata \
  	-format=html \
  	-output-dir=coverage-html \
  	-show-line-counts-or-regions \
  	-show-instantiations=false
  ```
- **Reproduce a crash:**
  ```shell
  cargo +nightly fuzz run --fuzz-dir fuzz-fake-hashes --features=libfuzzer_fuzz gossip_discovery_target {fuzz-dir(mentioned above)}/artifacts/gossip_discovery_target/crash-file-name
  ```

### CLN

- **Language:** C
- **Fuzzing engine:** `libFuzzer`
- **Fuzzing docs:**
  - https://github.com/ElementsProject/lightning/blob/master/doc/contribute-to-core-lightning/testing.md
  - https://github.com/ElementsProject/lightning/pull/8885
  - https://llvm.org/docs/LibFuzzer.html
- **Run continuously:** fuzz `fuzz-gossipd` on 8 workers
  ```shell
  # Build CLN for fuzzing
  ./configure CC=clang --enable-fuzzing --enable-address-sanitizer --enable-ub-sanitizer --disable-valgrind
  # Build the fuzz target
  make tests/fuzz/fuzz-gossipd
  # Start fuzzing
  ./tests/fuzz/fuzz-gossipd -jobs=8 -workers=8 tests/fuzz/corpora/fuzz-gossipd
  ```
- **Corpus location:** `tests/fuzz/corpora/fuzz-gossipd`
- **Coverage report:**
  ```shell
  # Build CLN with coverage instrumentation
  ./configure CC=clang --enable-fuzzing --enable-address-sanitizer --enable-ub-sanitizer --disable-valgrind --enable-coverage CC=clang
  # Build the fuzz target
  make tests/fuzz/fuzz-gossipd
  # Generate coverage reports
  LLVM_PROFILE_FILE="default.profraw" ./tests/fuzz/fuzz-gossipd -runs=0 tests/fuzz/corpora/fuzz-gossipd
  llvm-profdata merge -sparse default.profraw -o default.profdata
  llvm-cov show ./tests/fuzz/fuzz-gossipd \
    -instr-profile=default.profdata \
    -format=html \
    -output-dir=coverage-html \
    -show-line-counts-or-regions
  ```
- **Reproduce a crash:** `./tests/fuzz/fuzz-gossipd crash-file-name`

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

## Running it cross-implementation

The nice thing about smite is that it's implementation-agnostic. You model a
state machine once and that single fuzz test runs across all four
implementations, instead of hand-writing a custom test for each engine. That
saves a ton of time.

That said, this is complementary to per-implementation fuzzing, not a
replacement. Don't think of it as "just run smite and skip the
implementation-specific tests." Because smite fuzzes the implementations
agnostically, behaviour can vary from one to the next, so there are plenty of
cases and assertions you can check inside a single implementation's own fuzz
test that simply aren't possible in smite. Keep writing the standalone tests
alongside it.

### Smite

- **Language:** Rust
- **Fuzzing engine:** AFL++ with Nyx (currently)
- **Fuzzing docs:**
  - https://github.com/morehouse/smite
  - https://github.com/AFLplusplus/AFLplusplus
  - https://github.com/dergoegge/fuzzamoto
  - https://github.com/google/syzkaller
- **Run continuously** — only the IR scenario is shown here; for the others
  check the smite docs:

  ```sh
  TARGET=ldk
  SCENARIO=ir

  # Build the Docker image and Nyx sharedir as above
  docker build -t smite-$TARGET-$SCENARIO -f workloads/$TARGET/Dockerfile --build-arg SCENARIO=$SCENARIO .
  ./scripts/setup-nyx.sh /tmp/smite-nyx smite-$TARGET-$SCENARIO ~/AFLplusplus

  # Enable the KVM VMware backdoor (required for Nyx)
  ./scripts/enable-vmware-backdoor.sh

  # Build the custom mutator
  cargo build --release -p smite-ir-mutator

  # Create seed corpus (an empty file works -- the mutator generates fresh programs)
  mkdir -p /tmp/smite-seeds
  printf '\x00' > /tmp/smite-seeds/empty

  # Start fuzzing with the custom mutator
  AFL_CUSTOM_MUTATOR_LIBRARY=target/release/libsmite_ir_mutator.so \
  AFL_CUSTOM_MUTATOR_ONLY=1 \
  AFL_FRAMESHIFT_DISABLE=1 \
  AFL_DISABLE_TRIM=1 \
  ~/AFLplusplus/afl-fuzz -X -i /tmp/smite-seeds -o /tmp/smite-out -- /tmp/smite-nyx
  ```

- **Corpus location:** `/tmp/smite-out/default/queue/*`
- **Coverage report:**

  ```sh
  # Generate coverage report
  ./scripts/coverage-report.sh $TARGET $SCENARIO /tmp/smite-out/default/queue/

  # View the report
  firefox ./$TARGET-$SCENARIO-coverage-report/html/index.html
  ```

- **Reproduce a crash:**

  ```sh
  # Get the crash input
  cp /tmp/smite-out/default/crashes/<crashing-input> ./crash
  mkdir -p $DEBUG_DIR

  # Reproduce in local mode (use the matching image and scenario binary)
  docker run --rm -v $PWD/crash:/input.bin -e SMITE_INPUT=/input.bin -v "$DEBUG_DIR:/data" -e SMITE_DATA_DIR=/data smite-$TARGET-$SCENARIO /$TARGET-scenario
  ```
