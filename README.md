# ln-state-machine-fuzzing

The table below demonstrates the current state of state machine fuzzing across Lightning
Network implementations:

<table>
  <thead>
    <tr>
      <th align="left">Target / Implementation</th>
      <th align="center"><a href="https://github.com/lightningnetwork/lnd">LND</a></th>
      <th align="center"><a href="https://github.com/ElementsProject/lightning">CLN</a></th>
      <th align="center"><a href="https://github.com/ACINQ/eclair">Eclair</a></th>
      <th align="center"><a href="https://github.com/lightningdevkit/rust-lightning">LDK</a></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Legacy Channel Establishment</td>
      <td align="center">✅</td>
      <td align="center">✅</td>
      <td align="center">✅</td>
      <td align="center">✅</td>
    </tr>
    <tr>
      <td>Dual-Funded Channel Establishment</td>
      <td align="center">🚫</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
    </tr>
    <tr>
      <td>Normal Operation / HTLCs & Commitments</td>
      <td align="center">⏳</td>
      <td align="center">✅</td>
      <td align="center">❌</td>
      <td align="center">✅</td>
    </tr>
    <tr>
      <td>Channel Mutual Close</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
    </tr>
    <tr>
      <td>Channel / Node Discovery</td>
      <td align="center">⏳</td>
      <td align="center">⏳</td>
      <td align="center">❌</td>
      <td align="center">✅</td>
    </tr>
    <tr>
      <td>Peer Storage / Backup</td>
      <td align="center">🚫</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
    </tr>
    <tr>
      <td>Channel Splicing</td>
      <td align="center">🚫</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">✅</td>
    </tr>
    <tr>
      <td>Watchtower</td>
      <td align="center">❌</td>
      <td align="center">🔌</td>
      <td align="center">🚫</td>
      <td align="center">🔌</td>
    </tr>
    <tr>
      <td>BOLT12 / Onion Messages</td>
      <td align="center">🚫</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
    </tr>
    <tr>
      <td>Full Stack</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">✅</td>
    </tr>
  </tbody>
</table>

**Legend:** ✅ Implemented · ❌ Not Implemented · ⏳ In Progress · 🚫 Not Supported · 🔌 Externally Managed

## Fuzz Test Coverage

The following fuzz tests were added to improve state machine fuzzing coverage across different implementations:

### Implementation-Specific Fuzz Tests

- **LND**
  - htlcswitch: add htlc state machine fuzz tests: [PR #10773](https://github.com/lightningnetwork/lnd/pull/10773)
  - discovery+lnmock: add gossip state machine fuzz tests: [PR #10605](https://github.com/lightningnetwork/lnd/pull/10605)

- **LDK**
  - fuzz: add fuzz target for P2PGossipSync gossip message handling: [PR #4532](https://github.com/lightningdevkit/rust-lightning/pull/4532)

- **CLN**
  - fuzz: add test for the gossipd message processing: [PR #9115](https://github.com/ElementsProject/lightning/pull/9115)

- **Eclair**
  - Eclair lacked fuzzing engine support, so I added Jazzer-based fuzz testing support in [PR #3276](https://github.com/ACINQ/eclair/pull/3276). I also added basic fuzz tests to bring coverage more in line with other Lightning implementations, including:
    - Add fuzz tests for onion, route blinding and lightning message codecs: [PR #3282](https://github.com/ACINQ/eclair/pull/3282)
    - Add fuzz tests for Bolt11 and Bolt12 deserialization: [PR #3292](https://github.com/ACINQ/eclair/pull/3292)

### Cross-Implementation Fuzz Tests via Smite

Some state machines are being tested via [smite](https://github.com/morehouse/smite), which enables writing a single state machine fuzz test that can be executed against all Lightning Network implementations. This approach significantly reduces the manual work required to test the same scenarios across different implementations.

The table below tracks the state machines targeted via smite and their progress. PR links are added as each state machine's tests are implemented.

| State Machine                          | Status         | PRs / Links                                                                                                     |
| -------------------------------------- | -------------- | --------------------------------------------------------------------------------------------------------------- |
| Legacy Channel Establishment           | 🚧 In progress | [PRs](https://github.com/morehouse/smite/pulls?q=is%3Apr+author%3ANishantBansal2003+created%3A%3C%3D2026-07-10) |
| Normal Operation / HTLCs & Commitments | 📋 Planned     | [Implementation plan](https://github.com/morehouse/smite/issues/111)                                            |
| Dual-Funded Channel Establishment      | 📋 Planned     | —                                                                                                               |
| Channel Mutual Close                   | 📋 Planned     | —                                                                                                               |
| Channel Reestablish                    | 📋 Planned     | —                                                                                                               |
| Peer Storage / Backup                  | 📋 Planned     | —                                                                                                               |
| Channel Splicing                       | 📋 Planned     | —                                                                                                               |

Status legend: 📋 Planned · 🚧 In progress · ✅ Done

_**Note:** Additional state machines will be added as the smite project evolves and high-priority scenarios are implemented._

See my [smite PRs](https://github.com/morehouse/smite/pulls?q=is%3Apr+author%3ANishantBansal2003+is%3Aclosed) for the full implementation details.

## Issues Discovered

Through this fuzzing effort, several bugs and issues have been discovered across different Lightning Network implementations:

- **LND**
  - [bug]: Force close triggered on restart during incomplete commit dance: [Issue #10618](https://github.com/lightningnetwork/lnd/issues/10618)
  - discovery: fix race on remoteUpdateHorizon in GossipSyncer: [PR #10530](https://github.com/lightningnetwork/lnd/pull/10530)
  - discovery: enforce strict validation of peer gossip messages: [PR #10581](https://github.com/lightningnetwork/lnd/pull/10581)
  - Zero-timestamp gossip DoS: [Blog](https://nishantbansal2003.github.io/posts/LND-Zero-Timestamp-Gossip-DoS/)

- **CLN**
  - openingd crashes on an assertion when a peer sends `open_channel` with `funding_satoshis > WALLY_SATOSHI_MAX` (3 variants, likely sharing a single fix, tracked in one thread): [Issue #9225](https://github.com/ElementsProject/lightning/issues/9225)

- **Eclair**
  - parseinvoice accepts BOLT 11 invoice with non-zero bech32 padding bits: [Issue #3281](https://github.com/ACINQ/eclair/issues/3281)

- **Bolts**
  - Incorrect HTLC deductions in Appendix C/F test vectors: [Issue #1335](https://github.com/lightning/bolts/issues/1335)

_**Note:** Additional security-sensitive issues were discovered during this work but have not been publicly disclosed. These vulnerabilities are being handled through responsible disclosure processes with the respective implementation teams, and will be detailed on [nishantbansal2003.github.io](https://nishantbansal2003.github.io) once disclosure is appropriate._
