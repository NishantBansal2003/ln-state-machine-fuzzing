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
      <td align="center">❌</td>
      <td align="center">❌</td>
      <td align="center">❌</td>
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
