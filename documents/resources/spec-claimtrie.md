# Spec: Claimtrie


The claimtrie is a data structure that LBRY uses to store claims and prove the correctness of name resolution.


## Overview

A `name` is a bytestring of 0-255 bytes. It must be a valid UTF8 string. Names are the keys in the claimtrie.

A `claim` is a chunk of data stored in the claimtrie. It records an item that was published to the network or a publisher's identity.

Every claim has a unique `claimID`, an `amount` (how many credits were set aside to back the claim), and a `value`. The value may contain metadata about a piece of content,  a publisher's public key, or other information. See the [Metadata Spec](/resources/spec-metadata) for more information about the value.

The `claimtrie` is a [Merkle Tree](https://en.wikipedia.org/wiki/Merkle_tree) that maps names to claims. Claims are stored as leaf nodes in the tree. Names are stored as the path from the root node to the leaf node.

The hash of the root node  (the `root hash`) is stored in the header of each block in the blockchain. Nodes in the LBRY network use the root hash to efficiently and securely validate the state of the claimtrie.

Multiple claims can exist for the same name. They are all stored in the leaf node for that name, sorted in decreasing order by the total amount of credits backing each claim.


### Operations and Opcodes

There are four claim operations: *create*, *support*, *update*, and *abandon*.

The *create* operation is used to make a new claim for a name, or to submit a competing claim on an existing name. A *support* is a claim that adds to the credit total of an existing claim. A support does not have it’s own claim ID or data. Instead, it has the claim ID of the claim to which its amount will be added. An *update* changes the data or the amount stored in an existing claim or support. Updates do not change the claim ID, so an updated claim retains any supports attached to it. An *abandon* withdraws a claim or support, freeing the associated credits to be used for other purposes.

To enable the above operations, 3 new opcodes were added to the blockchain scripting language: `OP_CLAIM_NAME`,
`OP_SUPPORT_CLAIM`, and `OP_UPDATE_CLAIM` (in Bitcoin they are respectively `OP_NOP6`, `OP_NOP7`, and `OP_NOP8`). Each op
code will push a zero on to the execution stack, and will trigger the claimtrie to perform calculations necessary
for each bid type. Below are the three supported transactions scripts using these opcodes.

```
OP_CLAIM_NAME <name> <value> OP_2DROP OP_DROP <pubKey>
OP_UPDATE_CLAIM <name> <claimId> <value> OP_2DROP OP_2DROP <pubKey>
OP_SUPPORT_CLAIM <name> <claimId> OP_2DROP OP_DROP <pubKey>
```

`<pubKey>` can be any valid Bitcoin payout script, so a claimtrie script is also a pay-to-pubkey script to  a user-controlled address. Note that the zeros pushed onto the stack by the claimtrie opcodes and vectors are all dropped by `OP_2DROP` and `OP_DROP`. This means that claimtrie transactions exist as prefixes to Bitcoin payout scripts and can be spent just like standard transactions.

For example, a claim transaction setting the name “Fruit” to “Apple” and using a pay-to-pubkey script will have the following payout script:

```
OP_CLAIM_NAME Fruit Apple OP_2DROP OP_DROP OP_DUP OP_HASH160 <addressOne>
OP_EQUALVERIFY OP_CHECKSIG
```

Like any standard Bitcoin transaction output script, it will be associated with a transaction hash
and output index. The transaction hash and output index are concatenated and hashed to create the claimID for this claim. For the example above, let's say the above transaction hash is `7560111513bea7ec38e2ce58a58c1880726b1515497515fd3f470d827669ed43` and the output index is `1`. Then the claimID would be `529357c3422c6046d3fec76be2358004ba22e323`.

A support for this bid will have the following payout script:

```
OP_SUPPORT_CLAIM Fruit 529357c3422c6046d3fec76be2358004ba22e323 OP_2DROP OP_DROP
OP_DUP OP_HASH160 <addressTwo> OP_EQUALVERIFY OP_CHECKSIG
```

And now let's say we want to update the original claim to change the value to “Banana”. An update transaction has a special requirement that it must spend the existing claim that it wishes to update in its redeem script. Otherwise, it will be considered invalid and will not make it into the claimtrie. Thus it will have the following redeem script:

```
<signature> <pubKeyForAddressOne>
```

This is identical to the standard way of redeeming a pay-to-pubkey script in Bitcoin.

The payout script for the update transaction is:

```
OP_UPDATE_CLAIM Fruit 529357c3422c6046d3fec76be2358004ba22e323 Banana OP_2DROP
OP_2DROP OP_DUP OP_HASH160 <addressThree> OP_EQUALVERIFY OP_CHECKSIG
```


### Claim States


A claim can have the following states at a given block:

#### Accepted

An accepted claim or support is simply one that has been entered into the blockchain. This happens when the transaction containing the claim is included in a block.

#### Abandoned

An abandoned claim or support is one that was withdrawn by its creator. It is no longer in contention to control a name. Spending the transaction that contains the claim will also cause the claim to become abandoned.


#### Active

A claim is active when it is in contention for controlling a name (or a support for such a claim). An active claim must be accepted and not abandoned. The time it takes an accepted claim to become active is called the activation delay, and it depends on the claim type, the height of the current block, and the height at which the last takeover occurred for the claim’s name.

If the claim is an update or support to the current controlling claim, or if it is the first claim for a name (T = 0), the claim becomes active as soon as it is accepted. Otherwise it becomes active at height A, where $A = C + D$, and $D = min(4032, floor(\frac{H-T}{32}))$

A = activation height
D = activation delay
C = claim height (height when the claim was accepted)
H = current height
T = takeover height (the most recent height at which the controlling claim for the name changed)

In plain English, the delay before a claim becomes active is equal to the claim’s height minus height of the last takeover, divided by 32. The delay is capped at 4032 blocks, which is 7 days of blocks at 2.5 minutes per block (our target block time). The max delay is reached 224 (7x32) days after the last takeover. The goal of this delay function is to give long-standing claimants time to respond to takeover attempts, while still keeping takeover times reasonable and allowing recent or contentious claims to be taken over quickly.

#### Controlling

The controlling claim is the claim that is returned when a name is resolved. The controlling claim must be active and cannot be a support. Only one claim can be controlling for a given name at a given block. To determine which claim is controlling for a given name in a given block, the following algorithm is used:

1. For each active claim for the name, add up the amount of the claim and the amount of all the active supports for that claim. 

1. Determine if a takeover is happening.

    a. If the claim with the greatest total is the controlling claim from the previous block, then nothing changes. That claim is still controlling at this block.
    
    b. Otherwise, a takeover is occurring. Set the takeover height for this name to the current height, recalculate which claims and supports are now active, and then perform step 1 again.

1. At this point, the claim with the greatest total is the controlling claim at this block.

The purpose of 2b is to handle the case when multiple competing claims are made on the same name in different blocks, and one of those claims becomes active but another still-inactive claim has the greatest amount. Step 2b will cause this claim to also activate and become the controlling claim.

Here is a step-by-step example to illustrate the different scenarios. All claims are for the same name.

**Block 13:** Claim A for 10LBC  is accepted. It is the first claim, so it immediately becomes active and controlling.

State: A(10) is controlling

**Block 1001:** Claim B for 20LBC is accepted. It’s activation height is
$1001 + min(4032, floor(\frac{1001-13}{32})) = 1001 + 30 = 1031$

State: A(10) is controlling, B(20) is accepted.

**Block 1010:** Support X for 14LBC for claim A is accepted. Since it is a support for the controlling  claim, it activates immediately.

State: A(10+14) is controlling, B(20) is accepted.

**Block 1020:** Claim C for 50LBC is accepted. The activation height is
$1020 + min(4032, floor(\frac{1020-13}{32})) = 1020 + 31 = 1051$

State: A(10+14) is controlling, B(20) is accepted, C(50) is accepted.

**Block 1031:** Claim B activates. It has 20LBC, while claim A has 24LBC (10 original + 14 from support X). There is no takeover, and claim A remains controlling.

State: A(10+14) is controlling, B(20) is active, C(50) is accepted.

**Block 1040:** Claim D for 300LBC is accepted. The activation height is
$1040 + min(4032, floor(\frac{1040-13}{32})) = 1040 + 32 = 1072$

State: A(10+14) is controlling, B(20) is active, C(50) is accepted, D(300) is accepted.

**Block 1051:** Claim C activates. It has 50LBC, while claim A has 24LBC, so a takeover is initiated. The takeover height for this name is set to 1051, and therefore the activation delay for all the claims becomes $min(4032, floor(\frac{1051-1051}{32})) = 0$. All the claims become active. The totals for each claim are recalculated, and claim D becomes controlling because it has the highest total.

State: A(10+14) is active, B(20) is active, C(50) is active, D(300) is controlling.

