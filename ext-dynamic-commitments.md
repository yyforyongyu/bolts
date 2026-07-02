# Extension BOLT: Dynamic Commitments

This protocol defines a method for changing selected channel parameters after a
channel is open.

## Overview

A dynamic commitment update is negotiated while the channel is quiescent. The
peer that sends `dyn_propose` is the proposer. The peer that receives
`dyn_propose` is the responder. The proposer MUST be the quiescence initiator for
this quiescence session.

The following parameters may be changed:

* `dust_limit_satoshis`
* `max_htlc_value_in_flight_msat`
* `channel_reserve_satoshis`
* `htlc_minimum_msat`
* `to_self_delay`
* `max_accepted_htlcs`
* `channel_flags`

The proposer selects new values for parameters it advertised when the channel was
opened. `channel_flags` is channel-wide announcement intent. Either peer may
propose a new value, and both peers apply the accepted value.

This protocol changes channel parameters, not channel type. A node MUST NOT
reject a proposal solely because the channel uses taproot commitments; for such
channels, `dyn_commit_sig` uses the taproot partial-signature form defined below.

When validating `dyn_propose`, the responder MUST validate the channel that would
result from replacing the proposer's current advertised values with the proposed
values. In particular, the responder:

* MUST reject a proposal where either peer's `channel_reserve_satoshis` would be
  less than either peer's `dust_limit_satoshis`.
* MUST reject a proposal where `dust_limit_satoshis` is smaller than 354
  satoshis, matching the `open_channel` requirement in
  [BOLT #2](02-peer-protocol.md#the-open_channel-message).
* MUST reject a proposal where `to_self_delay` is unreasonably large.
* MUST reject a proposal where `max_accepted_htlcs` violates the limit for the
  channel's current channel type.
* MUST reject a proposal for an announced channel if the proposer's resulting
  `max_htlc_value_in_flight_msat` is less than the proposer's resulting
  `htlc_minimum_msat`.
* MUST reject a proposal that includes `channel_reserve_satoshis` if the channel
  does not have explicit local and remote `channel_reserve_satoshis` parameters,
  such as channels opened with `open_channel2` and `accept_channel2`.
* MAY reject a proposal for any parameter value it considers unsafe or
  undesirable.

The normal successful flow is:

```
    +----------+                         +-----------+
    |          |-- stfu ---------------->|           |
    |          |<---------------- stfu --|           |
    |          |                         |           |
    |          |-- dyn_propose --------->|           |
    |          |<------------ dyn_ack ----|           |
    | Proposer |                         | Responder |
    |          |-- dyn_commit_sig ------>|           |
    |          |<------- revoke_and_ack --|           |
    |          |<-- commitment_signed ----|           |
    |          |-- revoke_and_ack ------>|           |
    +----------+                         +-----------+
```

The `dyn_commit_sig` message bundles the accepted proposal, the responder's
acceptance signature, and the proposer's commitment signature for the responder's
next commitment. For taproot channels, the commitment signature is carried using
the same MuSig2 partial-signature-with-nonce form as `commitment_signed`. This
avoids processing a commitment signature before the dynamic update it commits.

## Feature Bits

A node that supports this protocol:

* MUST advertise `option_dynamic_commitments`.
* MUST also advertise `option_quiesce`.
* MUST NOT send `dyn_propose`, `dyn_ack`, `dyn_reject`, or `dyn_commit_sig`
  unless `option_dynamic_commitments` has been negotiated.

## Messages

### The `dyn_propose` Message

This message proposes a set of new channel parameters.

1. type: 111 (`dyn_propose`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`dyn_propose_tlvs`:`tlvs`]

1. `tlv_stream`: `dyn_propose_tlvs`
2. types:
   1. type: 0 (`dust_limit_satoshis`)
   2. data:
      * [`u64`:`dust_limit_satoshis`]
   1. type: 1 (`max_htlc_value_in_flight_msat`)
   2. data:
      * [`u64`:`max_htlc_value_in_flight_msat`]
   1. type: 2 (`htlc_minimum_msat`)
   2. data:
      * [`u64`:`htlc_minimum_msat`]
   1. type: 3 (`channel_reserve_satoshis`)
   2. data:
      * [`u64`:`channel_reserve_satoshis`]
   1. type: 4 (`to_self_delay`)
   2. data:
      * [`u16`:`to_self_delay`]
   1. type: 5 (`max_accepted_htlcs`)
   2. data:
      * [`u16`:`max_accepted_htlcs`]
   1. type: 6 (`channel_flags`)
   2. data:
      * [`byte`:`channel_flags`]

The sender:

* MUST have negotiated `option_dynamic_commitments`.
* MUST have sent and received `channel_ready` for the channel.
* MUST be the quiescence initiator for this quiescence session.
* MUST NOT send `dyn_propose` until the channel is quiescent.
* MUST NOT send `dyn_propose` if either commitment state contains any HTLC,
  whether trimmed or untrimmed.
* MUST NOT send `dyn_propose` if any HTLC add, HTLC removal, fee update, splice,
  splice RBF, or shutdown is pending for the channel.
* MUST include at least one known TLV.
* MUST NOT include a parameter value that violates the requirements for the
  corresponding field in `open_channel` or `accept_channel`.
* MUST NOT include `channel_reserve_satoshis` if the channel does not have
  explicit local and remote `channel_reserve_satoshis` parameters.
* MUST treat the proposal as outstanding after sending `dyn_propose` until the
  proposal is rejected, abandoned on disconnection, or committed.
* MUST NOT send another `dyn_propose` while a proposal is outstanding.
* MUST NOT cancel a proposal in-band after sending `dyn_propose`.
* MUST set undefined bits in `channel_flags` to 0.

The receiver:

* if `channel_id` does not match an existing channel with the sender:
  * MUST send `error` and close the connection.
* if the proposal violates any dynamic commitment precondition:
  * MUST send `error` and close the connection.
* if the proposal fails to parse, including because it contains an unknown even
  TLV type:
  * MUST send `error` and close the connection.
* if the proposal contains an unknown odd TLV type:
  * MUST send `dyn_reject` with the `unknown_or_unsupported_tlv` bit set.
* if it accepts every proposed parameter:
  * MUST send `dyn_ack`.
* otherwise:
  * MUST send `dyn_reject`.

Quiescence prevents new channel updates from being sent, but does not by itself
prove that the current commitment state is clean enough for a params-only dynamic
commitment update. Dynamic commitments require no HTLCs because `dyn_commit_sig`
does not carry HTLC signatures, and changing `dust_limit_satoshis` can make a
trimmed HTLC become untrimmed. Dynamic commitments also require no pending update,
splice, splice RBF, or shutdown state because `dyn_commit_sig` commits only the
accepted dynamic parameter update to the next commitment.

Future optional parameters SHOULD use odd TLV types so older nodes can parse the
proposal and reject unsupported fields. Future required parameters MAY use even
TLV types.

### The `dyn_ack` Message

This message accepts a `dyn_propose`.

1. type: 113 (`dyn_ack`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`signature`]

The sender:

* MUST NOT send `dyn_ack` unless it has received a valid `dyn_propose` for the
  same channel.
* MUST NOT send `dyn_ack` if it has already sent `dyn_ack` or `dyn_reject` for
  this negotiation.
* MUST set `signature` to the signature defined in
  [Appendix A](#appendix-a-dyn_ack-signature).
* MUST remember the accepted proposal and signature until the update is
  committed, the acceptance is abandoned as specified in
  [Reestablish](#reestablish), or the channel fails.

The receiver:

* if `channel_id` does not match an existing channel with the sender:
  * MUST send `error` and close the connection.
* if it has no outstanding `dyn_propose` for the channel:
  * MUST send `error` and close the connection.
* MUST verify `signature` as defined in
  [Appendix A](#appendix-a-dyn_ack-signature).
* MUST persist the accepted proposal and signature before sending
  `dyn_commit_sig`.

### The `dyn_reject` Message

This message rejects a `dyn_propose`.

1. type: 115 (`dyn_reject`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`...*byte`:`parameter_rejections`]

`parameter_rejections` is a fixed-enumeration bitfield using the same bit
ordering as feature vectors. It is not indexed by TLV type. A zero-length or
zero-valued bitfield rejects the negotiation without identifying specific
parameters.

| Bit | Meaning |
| --- | ------- |
| 0 | `unknown_or_unsupported_tlv` |
| 1 | `dust_limit_satoshis` |
| 2 | `max_htlc_value_in_flight_msat` |
| 3 | `htlc_minimum_msat` |
| 4 | `channel_reserve_satoshis` |
| 5 | `to_self_delay` |
| 6 | `max_accepted_htlcs` |
| 7 | `channel_flags` |

For example, rejecting both `dust_limit_satoshis` and
`max_htlc_value_in_flight_msat` is encoded as `0b00000110`.

Bit 0 is permanently reserved for `unknown_or_unsupported_tlv`. Future parameter
rejection bits MUST be appended starting at bit 8, regardless of the proposed
parameter's TLV type.

The sender:

* MUST NOT send `dyn_reject` unless it has received a valid `dyn_propose` for
  the same channel.
* MUST NOT send `dyn_reject` if it has already sent `dyn_ack` or `dyn_reject`
  for this negotiation.
* SHOULD set bits for each parameter it rejects.
* MUST forget the rejected proposal after sending `dyn_reject`.

The receiver:

* if `channel_id` does not match an existing channel with the sender:
  * MUST send `error` and close the connection.
* MUST forget the rejected proposal.
* MUST NOT attempt another dynamic commitment update until a new quiescence
  session has been established.

### The `dyn_commit_sig` Message

This message commits the accepted proposal to the responder's next commitment.

1. type: 117 (`dyn_commit_sig`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`commit_signature`]
   * [`signature`:`dyn_ack_signature`]
   * [`bigsize`:`accepted_tlvs_len`]
   * [`accepted_tlvs_len*byte`:`accepted_tlvs`]
   * [`dyn_commit_sig_tlvs`:`tlvs`]

1. `tlv_stream`: `dyn_commit_sig_tlvs`
2. types:
   1. type: 2 (`partial_signature_with_nonce`)
   2. data:
      * [`32*byte`:`partial_signature`]
      * [`66*byte`:`public_nonce`]

The `accepted_tlvs` field is the exact accepted `dyn_propose_tlvs` byte stream,
with TLV records in strictly increasing order. The `dyn_ack_signature` commits to
these bytes as specified in [Appendix A](#appendix-a-dyn_ack-signature). The
`dyn_commit_sig_tlvs` stream is not part of the accepted proposal; it carries
signature data needed to commit the proposal.

For non-taproot channels, the `commit_signature` field is equivalent to the
`signature` field in `commitment_signed` for the responder's next commitment
after applying `accepted_tlvs`.

For taproot channels, `commit_signature` is set to 64 zero bytes and the
`partial_signature_with_nonce` TLV is equivalent to the
`partial_signature_with_nonce` TLV in `commitment_signed` for the responder's
next commitment after applying `accepted_tlvs`.

Since dynamic commitments require no HTLCs in either commitment state, there are
no HTLC signatures.

Bundling the accepted TLVs, the responder's acceptance signature, and the
proposer's commitment signature avoids a race where a commitment signature is
processed before the dynamic update it signs. It also gives peers one
self-authenticating object to retransmit after reconnect and avoids an extra
round trip before the commitment dance.

The sender:

* MUST NOT send `dyn_commit_sig` unless it has received a valid `dyn_ack` for
  the same proposal.
* MUST set `dyn_ack_signature` to the signature received in `dyn_ack`.
* MUST set `accepted_tlvs` to the exact accepted proposal TLV stream.
* if the channel does not use taproot commitments:
  * MUST set `commit_signature` to a valid low-S signature for the receiver's
    next commitment after applying `accepted_tlvs`.
  * MUST NOT include `partial_signature_with_nonce`.
* if the channel uses taproot commitments:
  * MUST set `commit_signature` to 64 zero bytes.
  * MUST include `partial_signature_with_nonce`.
  * MUST generate a fresh MuSig2 nonce and use it with the receiver's latest
    `next_local_nonce` from `channel_ready`, `channel_reestablish`, or
    `revoke_and_ack` to produce the partial signature, as specified for
    `commitment_signed` in the taproot commitment protocol.
* MUST NOT send `dyn_commit_sig` if a splice transaction is pending for the
  channel.

The receiver:

* if `channel_id` does not match an existing channel with the sender:
  * MUST send `error` and close the connection.
* MUST verify that `dyn_ack_signature` is valid under its node identity public
  key for `accepted_tlvs` and the expected next commitment number.
* if the channel does not use taproot commitments:
  * MUST verify that `commit_signature` is a valid low-S signature for its next
    commitment after applying `accepted_tlvs`.
  * MUST fail the channel if `partial_signature_with_nonce` is present.
* if the channel uses taproot commitments:
  * MUST fail the channel if `commit_signature` is not 64 zero bytes.
  * MUST fail the channel if `partial_signature_with_nonce` is absent or its
    `public_nonce` cannot be parsed as two compressed secp256k1 points.
  * MUST verify `partial_signature_with_nonce` for its next commitment after
    applying `accepted_tlvs`, using the latest `next_local_nonce` it sent in
    `channel_ready`, `channel_reestablish`, or `revoke_and_ack`, as specified for
    `commitment_signed` in the taproot commitment protocol.
* if any signature is invalid:
  * MUST send `error` and fail the channel.
* MUST treat `dyn_commit_sig` as a `commitment_signed` equivalent for
  persistence and retransmission.
* MUST persist the received `dyn_commit_sig`, the accepted TLVs, any
  `dyn_commit_sig_tlvs`, and the resulting commitment state before sending
  `revoke_and_ack`.
* MUST respond with `revoke_and_ack` if `dyn_commit_sig` is valid.

For commitment-number accounting and retransmission, `dyn_commit_sig` is a
commitment signature message for the receiver's next commitment. A node that
sends `dyn_commit_sig` MUST treat it as a sent `commitment_signed` for the
receiver's commitment chain. A node that receives `dyn_commit_sig` MUST treat it
as a received `commitment_signed` for that chain. The BOLT #2 persistence,
`channel_reestablish`, `next_commitment_number`, `next_revocation_number`, and
retransmission rules that apply to `commitment_signed` apply to `dyn_commit_sig`,
except that `dyn_commit_sig` is retransmitted instead of `commitment_signed` and
includes the accepted dynamic commitment TLVs and any `dyn_commit_sig_tlvs`
needed to verify the commitment signature.

## Execution

When the responder receives a valid `dyn_commit_sig`, the proposed parameter
changes are pending on the responder's commitment chain. The responder then sends
`revoke_and_ack` for its prior commitment and sends `commitment_signed` for the
proposer's next commitment with the same dynamic update applied. The responder
does not wait for the proposer to receive the `revoke_and_ack` before sending
that `commitment_signed`.

The responder's post-`dyn_commit_sig` `commitment_signed` commits the proposer's
next commitment with the accepted TLVs applied. This dynamic parameter change is
the update required by the `commitment_signed` rule that forbids signing without
updates.

For channels that have negotiated `option_dynamic_commitments`, a pending dynamic
commitment update counts as an update for the purpose of the BOLT #2
`commitment_signed` requirements.

After the proposer receives the responder's `revoke_and_ack`, the responder's
commitment chain uses the new parameters from the commitment number signed by
`dyn_commit_sig`.

After the proposer receives a valid `commitment_signed`, it sends
`revoke_and_ack` for its prior commitment. The proposer's commitment chain uses
the new parameters from the commitment number signed by that `commitment_signed`.

Both peers:

* MUST apply the accepted values to the parameters changed by `dyn_propose`.
* MUST enforce the new parameters for subsequent channel operation after the
  corresponding commitment chain has locked in the update.
* MUST remember, for each commitment chain, the last commitment height where the
  old script- or output-rendering parameters applied.
* MUST use historical script- or output-rendering parameters when reconstructing
  old commitment states.
* MUST NOT send `update_add_htlc`, `update_fulfill_htlc`, `update_fail_htlc`,
  `update_fail_malformed_htlc`, `update_fee`, splice, splice RBF, or `shutdown`
  messages during the dynamic commitment execution dance.

### Epoch Boundaries

For the parameter set defined by this protocol, the parameters that affect
commitment transaction rendering map to commitment chains as follows:

| Updated parameter | Commitment chain affected |
| --- | --- |
| Proposer's `dust_limit_satoshis` | Proposer's commitment chain |
| Proposer's `to_self_delay` | Responder's commitment chain |

The other parameters in this protocol do not affect commitment transaction
scripts or output trimming, but they still affect subsequent channel validation
and operation.

For the responder's commitment chain, the old parameters apply through the
commitment height immediately before the height signed by `dyn_commit_sig`; the
new parameters apply starting at the height signed by `dyn_commit_sig`.

For the proposer's commitment chain, the old parameters apply through the
commitment height immediately before the height signed by the responder's
`commitment_signed`; the new parameters apply starting at the height signed by
that `commitment_signed`.

Peers MUST use these per-chain boundaries when reconstructing historical
commitment transactions and scripts.

### Quiescence Termination

The dynamic commitments protocol terminates quiescence at the following points:

* for a successful dynamic commitment update:
  * the proposer considers the channel no longer quiescent after it sends the
    final `revoke_and_ack`;
  * the responder considers the channel no longer quiescent after it receives the
    final `revoke_and_ack`.
* for a rejected dynamic commitment update:
  * the responder considers the channel no longer quiescent after it sends
    `dyn_reject`;
  * the proposer considers the channel no longer quiescent after it receives
    `dyn_reject`.
* for a disconnected dynamic commitment update:
  * disconnection ends quiescence as specified in
    [BOLT #2](02-peer-protocol.md#channel-quiescence), and
    [Reestablish](#reestablish) governs dynamic commitment recovery.
* for a timed-out dynamic commitment update:
  * timeout causes disconnection, so quiescence ends through the disconnect path.

If the responder sends normal updates immediately after sending `dyn_reject`, the
proposer MUST NOT process those updates until it has processed `dyn_reject`.

If the proposer sends normal updates immediately after sending the final
`revoke_and_ack`, the responder MUST NOT process those updates until it has
processed the final `revoke_and_ack`.

### Dynamic Commitment Timeout

Once the channel is quiescent for a dynamic commitment update, if a node is
waiting for a peer message required by this protocol (`dyn_propose`, `dyn_ack`,
`dyn_reject`, `dyn_commit_sig`, `revoke_and_ack`, or `commitment_signed`) and
does not receive it within 60 seconds, it SHOULD disconnect. This timeout is
independent of the HTLC-related quiescence timeout in
[BOLT #2](02-peer-protocol.md#channel-quiescence). Disconnection converts the
stalled negotiation or execution into the recovery path in
[Reestablish](#reestablish).

## Reestablish

A node:

* MUST persist a locally originated `dyn_propose` before sending it.
* MUST persist a `dyn_ack` acceptance before sending `dyn_ack`.
* MUST persist a received `dyn_ack` before sending `dyn_commit_sig`.
* MUST persist `dyn_commit_sig`, the accepted TLVs, any `dyn_commit_sig_tlvs`,
  and the commitment number it signs before sending `dyn_commit_sig`.
* MUST persist a sent `dyn_commit_sig` until the corresponding commitment dance
  has completed.

Upon disconnection, any negotiation that has not reached a persisted-sent or
received-and-persisted `dyn_commit_sig` is aborted, except for a responder's
persisted `dyn_ack` acceptance as specified below. A node:

* MUST forget any `dyn_propose` or `dyn_reject` state for that aborted
  negotiation.
* MUST NOT retransmit `dyn_propose`, `dyn_ack`, or `dyn_reject` after
  reconnection.
* MUST start a new quiescence session before attempting the dynamic commitment
  update again.

If a node has received `dyn_ack` but has not persisted a sent `dyn_commit_sig`
before disconnection, it:

* MUST forget the received `dyn_ack`.
* MUST NOT send `dyn_commit_sig` for that proposal after reconnection.

If a node has sent `dyn_ack` but has not received `dyn_commit_sig` before
disconnection, it:

* MUST retain the accepted proposal and `dyn_ack` signature after reconnection.
* MUST NOT send any non-retransmission update for that channel after both
  `channel_reestablish` messages have been exchanged until one of the following
  occurs:
  * it receives a valid `dyn_commit_sig` for the retained acceptance;
  * it receives another update or a new `stfu` for the channel from the proposer,
    which abandons the retained acceptance; or
  * 60 seconds elapse, which abandons the retained acceptance.

If a node has persisted a sent `dyn_commit_sig` and the peer's
`channel_reestablish` reports that it still expects the commitment number signed
by `dyn_commit_sig`, the node:

* MUST retransmit `dyn_commit_sig`.
* MUST NOT retransmit `dyn_propose` for that update.
* MAY retransmit `dyn_commit_sig` even though the channel is no longer
  quiescent.
* MUST NOT send any other update for that channel before the retransmitted
  `dyn_commit_sig`.

If a node receives a valid `dyn_commit_sig` after reconnection:

* if another update for the channel has already been sent or received after
  reconnection, a new `stfu` has been sent or received after reconnection, or the
  retained acceptance has already been abandoned after the 60-second wait:
  * MUST send `error` and fail the channel.
* otherwise:
  * MUST process `dyn_commit_sig` according to this protocol, even if it forgot
    the pre-`dyn_commit_sig` negotiation state during disconnection.

Once `dyn_commit_sig` has been processed, ordinary `commitment_signed` and
`revoke_and_ack` retransmission rules apply, except that the dynamic update is
included in the signed commitment state.

For `channel_reestablish`, a sent or received `dyn_commit_sig` is counted as the
corresponding sent or received `commitment_signed` for the responder's commitment
chain. Peers MUST include the dynamic update when evaluating
`next_commitment_number`, `next_revocation_number`, and whether
`dyn_commit_sig`, `commitment_signed`, or `revoke_and_ack` must be retransmitted.
The `channel_reestablish` messages themselves do not count as another update for
the rule above.

## Channel Announcement State

The `channel_flags` TLV changes the channel's current announcement intent. It
uses the same known bits as the `channel_flags` field in `open_channel`.
Undefined bits MUST be ignored by the receiver.

For channels where `option_dynamic_commitments` was negotiated, requirements in
[BOLT #7](07-routing-gossip.md#the-announcement_signatures-message) that refer to
the `announce_channel` bit in `open_channel` are evaluated using the channel's
current announcement intent after any completed dynamic commitment update.

If a proposal would change a channel from unannounced to announced, the receiver:

* MUST reject the proposal if the funding transaction does not have a confirmed
  `short_channel_id`.
* MUST reject the proposal if the channel type includes `option_scid_alias`.
* MUST reject the proposal if either peer has sent `shutdown`.

If a channel changes from unannounced to announced, then after the dynamic update
has completed, both peers:

* MUST complete the `announcement_signatures` flow for the channel if it has not
  already been completed and the timing requirements in
  [BOLT #7](07-routing-gossip.md#the-announcement_signatures-message) are met.
* SHOULD send a `channel_announcement` once the announcement signatures are
  available.
* SHOULD send current `channel_update` messages for both directions they
  control.

If a channel changes from announced to unannounced, then after the dynamic update
has completed, each peer:

* MUST NOT treat the change as removing previously gossiped announcements from
  the network.
* SHOULD send a `channel_update` disabling its direction before it stops
  advertising usable public policy for the channel.
* MUST NOT send an enabled public `channel_update` for the channel while its
  current announcement intent is unannounced.

If a peer's `htlc_minimum_msat` or `max_htlc_value_in_flight_msat` changes for an
announced channel, the other peer is responsible for the `channel_update` in the
direction toward the peer whose value changed. For example, if the proposer's
values change, the responder is responsible for the responder-to-proposer
direction.

If the changed `max_htlc_value_in_flight_msat` is less than the responsible
peer's currently advertised `htlc_maximum_msat`, the responsible peer MUST send a
new `channel_update` with an `htlc_maximum_msat` no greater than the changed
`max_htlc_value_in_flight_msat` and no less than the peer's resulting
`htlc_minimum_msat` once the dynamic update has completed. Otherwise, the
responsible peer SHOULD send a new `channel_update` reflecting the changed
`htlc_minimum_msat` and, if advertised, an `htlc_maximum_msat` no greater than
the changed `max_htlc_value_in_flight_msat` and no less than the peer's resulting
`htlc_minimum_msat`.

Upon reconnection, announcement-signature retransmission requirements use the
current announcement intent.

## Appendix A: `dyn_ack` Signature

The `dyn_ack` signature is a 64-byte secp256k1 ECDSA signature using the
`signature` encoding defined in [BOLT #1](01-messaging.md#fundamental-types). The
signature MUST be valid under the responder's node identity public key and MUST
use a low-S value.

The signature is over the double-SHA256 digest of:

```
"lightning-dynamic-commitments-dyn-ack-v1" ||
chain_hash ||
channel_id ||
u64(next_commitment_number) ||
accepted_tlvs
```

The `next_commitment_number` is the next commitment number the responder expects
to receive from the proposer. The `accepted_tlvs` field is the exact encoded TLV
stream from `dyn_propose`, with TLV records in strictly increasing order as
required by [BOLT #1](01-messaging.md). When echoed in `dyn_commit_sig`, these
same bytes are carried in the length-delimited `accepted_tlvs` field.

The verifier MUST reject the signature if the proposal TLVs, channel, chain,
expected commitment number, or signature encoding differ from the values required
above.

The responder's node identity key is used so the acceptance can be verified from
the peer identity already authenticated for the channel. Echoing this signature
in `dyn_commit_sig` lets the responder verify the accepted parameters and
commitment number after reconnect even if pre-`dyn_commit_sig` negotiation state
was forgotten.
