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
      <td align="center">❌</td>
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
