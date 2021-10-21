# BOLT #XX: Sub-channel protocol

# Table of Contents

  * [Motivations](#motivations)
  * [Overview](#overview)
  * [Split transaction](#split-transaction)
      * [Split transaction outputs](#split-transaction-outputs)
  * [Sub-channel commitment transaction](#sub-channel-commitment-transaction)
  * [Sub-channels protocol](#sub-channels-protocol)
      * [The `subchannel_request` message](#the-subchannel_request-message)
      * [The `subchannel_accept` message](#the-subchannel_accept-message)
      * [The `subchannel_finalize` message](#the-subchannel_finalize-message)
      * [The `subchannel_reject` message](#the-subchannel_reject-message)
  * [Handle concurrent sub-channel requests](#handle-concurrent-sub-channel-requests)


## Motivations

To support functionalities between two peers of a Lightning Network channel in addition to the common payment scheme, they need a way to create additional channels along with their main payment channel.

While the main motivation for this specification is enabling the collocation of [Discreet Log Contract](https://github.com/discreetlogcontracts/dlcspecs/blob/master/Introduction.md#the-bet-a-short-introduction-to-discreet-log-contracts) channels, the scheme described is generic and any application that can be built using channels could use it as a basis.

## Overview

To avoid interactions between the sub-channels, this specification introduces a "split" transaction that takes as input the [funding output](03-transactions.md#funding-transaction-output) and contains as many outputs as the number of desired sub-channels (at the minimum two, one for the main payment channel and another one for another sub-channel).
In order to keep the sub-channel establishment protocol application agnostic, a [commitment transaction](03-transactions.md#commitment-transaction) is created for each sub-channel during the setup phase, which can then be revoked and replaced through an application specific protocol.
Note that the split transaction is also revocable to enable updates to the set of sub-channels, or return the channel to a regular one.

For simplicity, the protocol requires the first sub-channel to be set as the main payment channel inheriting the HTLCs from the original channel upon creation.

The rest of this specification details the protocol in which the peers need to take part to establish and update sub-channels, as well as how to [deal with race between two concurrent sub-channel requests](#Handle-concurrent-sub-channel-requests).

## Split transaction

The split transaction is used to fan out the fund output to multiple ones that can be used as inputs to sub-channels.

In order to enable updates to the set of sub-channels, a split transaction can be revoked, meaning that each party holds a different version, and all of the outputs are timelocked or spendable directly with knowledge of the correct secrets, which consist of a revocable secret updated every time the split transaction is updated, and a `split_key` whose public point should be exchanged prior to the sub-channel establishment.

_Note: where or when the split_pubkey is exchanged is not yet defined. It could maybe be added as part of the open channel TLV stream?_

* version: 2
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from `funding_created` message
   * `txin[0]` sequence: upper 8 bits are 0x80, lower 24 bits are upper 24 bits of the obscured split number
   * `txin[0]` script bytes: 0
   * `txin[0]` witness: `0 <signature_for_pubkey1> <signature_for_pubkey2>`

The 48-bit split number is obscured by `XOR` with the lower 48 bits of:

    SHA256(payment_basepoint from open_channel || payment_basepoint from accept_channel)

This obscures the number of updates to the split transaction made on the channel in the
case of unilateral close, yet still provides a useful index for both
nodes (who know the `payment_basepoint`s) to quickly find a revoked
split transaction.

### Split transaction outputs

Each output of a split transaction is delayed to allow an opportunity for the counter party to submit a penalty transaction.
It can be claimed without delay by the counter-party with the revocation secret and a signature using its `split_key`.
Note that due to the addition of the split transaction, recovery of fund in the case of a unilateral close takes twice as much time than for a regular channel.
In addition, since unilateral closing of a channel might take longer due to the split transaction, on-chain fulfillment of HTLCs should be initiated earlier.
As both split and commitment transactions can be broadcast together, initiating channel failure a few blocks earlier should suffice.

#### `subchannel` Output

This output provides funds for a sub-channel and is timelocked using `OP_CHECKSEQUENCEVERIFY` in case the split transaction it belongs to gets revoked.
It can be claimed, without delay, by the other party if they know the revocation private key.
However, since the output pays to both parties, we need to add a condition to the revocation path, which is done using the `split_key` of the party remote party.
The output is a version-0 P2WSH, with a witness script:

    OP_IF
        # Penalty transaction
        <remote_split_pubkey>
        OP_CHECKSIGVERIFY
        <revocationpubkey>
        OP_CHECKSIG
    OP_ELSE
        `to_self_delay`
        OP_CHECKSEQUENCEVERIFY
        OP_DROP
        2
        <pubkey1>
        <pubkey2>
        2
        OP_CHECKMULTISIG
    OP_ENDIF

* Where `pubkey1` is the lexicographically lesser of the two `funding_pubkey` in compressed format, and where `pubkey2` is the lexicographically greater of the two.

The output is spent by an input with `nSequence` field set to `to_self_delay` (which can only be valid after that duration has passed) and witness:

    <> <privkey1sig> <privkey2sig> <>

If a revoked split transaction is published, the other party can spend all of its outputs immediately with the following witness:

    <revocation_sig> <own_split_sig> 1

### Anchor outputs

If `options_anchor` was negotiated for the channel, anchor outputs should be added to the split transaction similarly than desribed in [BOLT #3](03-transactions.md#to_local_anchor-and-to_remote_anchor-Output-(option_anchors)).

## Sub-channel commitment transaction

Commitment transactions for sub-channels are similar to the [one for regular channels](03-transactions.md#Commitment-transaction-outputs).

## Sub-channel protocol

Sub-channel creation and update can be initiated by any of the two party of a channel.
The initiator proposes the creation or update of sub-channels by sending a `subchannel_request` message.
If the counter-party accepts, the channel is transformed to include a split transaction and a number of sub-channels.
After the sub-channel setup is complete, all [BOLT 2](02-peer-protocol.md) messages with the `channel_id` of the channel are considered to relate to the first sub-channel.
Updates to other sub-channels are performed using application specific protocols, or at least using a different `channel_id` (this specification does not concern with how this should be achieved).

### The `subchannel_request` message

This message constitutes a proposal by the sending node to create or update a set of sub-channels.
It contains information about the number of sub-channels, their respective balance as well as signatures for the commitment transactions of each sub-channel.
Setting the value of `num_subchannels` to 0 indicates a desire to close all but the main payment sub-channel and resume ordinary LN operating mode.

1. type: xx (`subchannel_request`)
2. data:
   * [`channel_id`: `channel_id`]
   * [`signature`: `split_signature`]
   * [`u8`: `num_subchannels`]
   *   [`u64`: `sender_satoshis`]  ‾|
   *   [`u64`: `receiver_satoshis`] | x `num_subchannels`
   *   [`signature`: `signature`]  _|

#### Requirements

A sending node:
  - MUST not set `num_subchannels` to 1.
  - MUST set `num_subchannels` to a value greater than 1 if the channel is a single channel at the time of sending.
  - MUST set the values `sender_satoshis` and `receiver_satoshis` of the first sub-channel to large enough values to provide for all HTLC outputs and satisfy each peer's channel reserve.
  - MUST not apply any change to the main channel commitment transaction or send any `commitment_signed` message prior to having received a `subchannnel_accept` or `subchannel_reject` message from the receiver.
  - MUST set the values `sender_satoshis` and `receiver_satoshis` such that:
    - The sum of all `sender_satoshis` is equal to the previous balance of the sending party (plus or minus the fee difference with the previous channel state if the sender funded the channel) plus the sum of all outbound HTLC values.
    - The sum of all `receiver_satoshis` is equal to the previous balance of the receiving party (plus or minus the fee difference with the previous channel state if the receiver funded the channel) plus the sum of all inbound HTLC values.
    - The sum of each `sender_satoshis`/`receiver_satoshis` pair is greater than `dust_limit_satoshi`.
  - if `num_subchannels` is greater than 1:
    - MUST set `split_signature` to the Bitcoin signature of the split transaction of the receiving party.
    - MUST set each commitment transaction `signature` to the Bitcoin signature of the commitment transaction of the receiving party for the corresponding sub-channel.
  - if `num_subchannels` is equal to 0:
    - MUST set `split_signature` to the Bitcoin signature of the commitment transaction of the receiving party.

A receiving node:
  - if `num_subchannels` is set to 1:
    - MUST fail the channel.
  - if any of `split_signature` or commitment transactions `signature` is invalid OR non-compliant with the LOW-S standard rule:
    - MUST fail the channel.
  - if the values `sender_satoshis` and `receiver_satoshis` of the first sub-channel are not enough to provide for the HTLC outputs of the main channel commitment transactions and satisfy each peer's channel reserve:
    - SHOULD fail the channel.
  - if the sum of all `sender_satoshis`is not equal to the previous balance of the sending party (plus or minus the fee difference with the previous channel state if the sender funded the channel):
    - SHOULD fail the channel.
  - if the funding node balance is not sufficient to cover the fees for the split and commitment transactions at the negotiated feerate:
    - SHOULD fail the channel.
  - if the sum of all `reveiver_satoshis`is not equal to the previous balance of the sending party (plus or minus the fee difference with the previous channel state if the receiver funded the channel):
    - SHOULD fail the channel.
  
#### Rationale

There always need to be a main payment channel part of the sub-channels, and there is no point in creating sub-channels if it is the only one.

In order for both parties to be able to close the main payment channel, the first output needs to provide enough fund to cover for all on-going HTLCs.

To keep things simple the main payment channel is not updated as part of the sub-channels creation.

Creating sub-channels should not alter the balance of the parties apart from adjusting for increase or decrease in fees.

### The `subchannel_accept` message

If the accept party agrees to opening/updating sub-channels, it should reply with a `subchannel_accept` message.
The message includes the revocation secret for the previous commitment/split transaction as the sender can use the split/commitment transaction signature provided in the received `subchannel_request` message to forcibly close the channel.
The sender provide `num_subchannels` + 1 `point`, one for the split/commitment transaction, and one for each sub-channel so as to be able to revoke their commitment transactions independently.

1. type: xx (`subchannel_accept`)
2. data: 
   * [`channel_id`: `channel_id`]
   * [`32*byte`:`per_split_secret`]
   * [`signature`: `split_signature`]
   * [`point`: `next_per_split_point`]
   * [`u8`: `num_subchannels`]
   *   [`signature`: `signature`]             ‾|
   *   [`point`: `next_per_commitment_point`] _| x num_subchannels

 #### Requirements

 A sending node:

  - MUST set `per_split_secret` to the secret used to generate keys for the previous split/commitment transaction.
  - MUST set `next_per_split_point` to the values for its next split/commitment transaction.
  - MUST set `num_subchannels` to the same value as in the received `subchannel_request`.
  - if `num_subchannels` is greater than 1:
    - MUST set `next_per_commitment_points` to the values for the next commitment transactions of each sub-channel.
    - MUST set `split_signature` to the Bitcoin signature of the split transaction of the sending party.
    - MUST set each commitment transaction `signature` to the Bitcoin signature of the commitment transaction of the receiving party for the corresponding sub-channel.
  - if `num_subchannels` is equal to 0:
    - MUST set `split_signature` to the Bitcoin signature of the commitment transaction of the receiving party.

A receiving node:

  - if it has not previously sent a `subchannel_request` with the same value for `num_subchannels`:
    - SHOULD fail the channel.
  - if `per_split_secret` is not a valid secret key or does not generate the previous `per_split_point`:
    - MUST fail the channel.
  - if the `per_split_secret` was not generated by the protocol in [BOLT #3](03-transactions.md#per-commitment-secret-requirements):
    - MAY fail the channel.
  - if any of `split_signature` or commitment transactions `signature` is invalid OR non-compliant with the LOW-S standard rule:
    - MUST fail the channel.

#### Rationale

Sending a `subchannel_accept` message without having received a `subchannel_request` message is a violation of the protocol and indicates a malfunction.

### The `subchannel_finalize` message

Upon receiving a `subchannel_accept` message, a node must reply with a `subchannel_finalize` message to revoke its previous commitment/split transaction.

1. type: xx (`subchannel_finalize`)
2. data: 
   * [`channel_id`: `channel_id`]
   * [`32*byte`:`per_split_secret`]
   * [`point`:`next_per_split_point`]
   * [`u8`: `num_subchannels`]
   * [`num_subchannels * point`:`next_per_commitment_point`]

 #### Requirements

 A sending node:

  - MUST set `per_split_secret` to the secret used to generate keys for the previous split/commitment transactions.
  - MUST set each `next_per_split_point` and `next_per_commitment_point` to the values for its next split/commitment transactions.

A receiving node:

  - if it has not previously sent a `subchannel_accept` message
    - MUST fail the channel.
  - if `per_split_secret` is not a valid secret key or does not generate the previous `per_split_point`:
    - MUST fail the channel.
  - if the `per_split_secret` was not generated by the protocol in [BOLT #3](03-transactions.md#per-commitment-secret-requirements):
    - MAY fail the channel.

#### Rationale

Sending a `subchannel_finalize` message without having received a `subchannel_accept` message is a violation of the protocol and indicates a malfunction.

### The `subchannel_reject` message

Upon receiving a `subchannel_request` message, a node may reject the request using the `subchannel_reject` message.

1. type: xx (`subchannel_finalize`)
2. data: 
   * [`channel_id`: `channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`reason`]

The `reason` field is a non encrypted field for the benefit of the `subchannel_request` sender to optionally provide an explanation for the rejection.

#### Requirements

A sending node:

  - MAY set the `reason` field to detail the reason for rejection.

A receiving node:
  - if it has not previously sent a `subchannel_request` message:
    - SHOULD fail the channel

#### Rationale

As the reason is directly transmitted to the recipient there is no reason to encrypt the `reason` field.

## Handle concurrent sub-channel requests

There is a possibility that the two nodes of a channel send a `subchannel_request` message at around the same time.
To solve this, the node with the highest `node_id` MUST reply with a `subchannel_reject` message if it receives a `subchannel_request` message after having sent a `subchannel_request` message of its own and before receiving a `subchannel_accept` or `subchannel_reject` message.
If the node with the highest `node_id` then receives a `subchannel_reject` message, it SHOULD wait for a small amount of time before sending another `subchannel_request` message to leave time to the other peer to send its own request.
