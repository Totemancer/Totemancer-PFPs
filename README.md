# Totemancer PFP/Avatar Collection

Chain: TON

## Summary

- Supply: 10,000 Aethra PFPs (Avatars / Profile Pictures), pre-minted and listed for sale. Numbers #1 to #10 are reserved for an on-chain auction at a later date.
- Numbers: buyers can choose any available number from #11 to #10000 during marketplace purchase. Auction is used from #1 to #10.
- Claims: Juiceboard participants can claim free or discounted random PFPs for a limited time (30 days) via a Relayer.
- Fairness: claim numbers are assigned by a deterministic and auditable algorithm using a public transaction hash.
- Upgrade: traits and rarity are not revealed by a global drop. Holders trigger a manual Upgrade in the Mini App, which sends a small on-chain transaction. The transaction hash seeds a public algorithm that deterministically assigns traits and rarity. Neither the user nor the team knows the outcome before the upgrade tx is finalized.

## Supply and numbering

- Canonical range: #1..#10000.
- Marketplace selectable during primary sale: #11..#10000.
- Reserved for auction: #1..#10.

## Sale

- All 10,000 PFPs are pre-minted and listed.
- Standard price before increase: 30 TON.
- Discounted price between 0 TON and 25 TON.
- Planned starting price 50 TON.
- Numbers from #1 to #10 are reserved for an auction.

Anyone can directly purchase any available number from #11..#10000 from on-chain marketplaces such as Getgems without a discount. Numbers #1..#10 will be sold via auction.

## Claim program for Juiceboard

Eligibility
- Have at least 1 JP collected before the claim snapshot time.

Claim window
- 30 days. Exact start and end timestamps will be posted in this repository. After the window closes, only marketplace purchases remain.

Relayer flow
- Totemancer Relayer buys from the marketplace and transfers to eligible users.
- Free claim example:
  - Relayer buys at the standard price (example 30 TON) and transfers the avatar to you.
- Discounted claim example:
  - Discounted effective price example: 25 TON.
  - If you have a 10 TON credit for the second avatar, you submit 15 TON (+ ~0.1 TON network fees) so your total outlay becomes 25 TON.
  - Relayer buys at the standard price (example 30 TON) and transfers to you.

Important
- The Relayer only purchases from the official collection.
- If a purchase fails because the target number was just sold, the Relayer retries under deterministic rules.

## Deterministic number assignment for claims

Claims use a public, reproducible mapping. Claimants do not pick numbers.

Inputs
- tx_hash: Transaction hash of the claimant's qualifying payment. This value is public on TON and immutable. Use the 64-character hex string as shown by explorers, case insensitive.

Domain
- Valid numbers for assignment: 11..10000 inclusive.
- Reserved numbers 1..10 are excluded.

Mapping
1. Parse tx_hash as a big-endian hex integer.
2. Compute base = (int(tx_hash, 16) % 9990) + 11.
   - 9990 equals the count of numbers in 11..10000.
3. If base is already taken, assign the next available number by incrementing and wrapping from 10000 to 11.
4. Continue until an available number is found.

Determinism
- The same tx_hash always maps to the same starting point.
- Given a known sale state at the evaluation block height, anyone can reproduce the assignment.

Reference implementation in Go
```go
package main

import (
    "encoding/hex"
    "fmt"
    "math/big"
)

func AssignNumber(txHashHex string, isTaken func(int) bool) (int, error) {
    const domainMin = 11
    const domainMax = 10000
    const domainSize = domainMax - domainMin + 1 // 9990

    hb, err := hex.DecodeString(txHashHex)
    if err != nil {
        return 0, err
    }
    h := new(big.Int).SetBytes(hb)
    mod := new(big.Int).Mod(h, big.NewInt(domainSize)).Int64()
    n := int(mod) + domainMin // 11..10000

    for i := 0; i < domainSize; i++ {
        if !isTaken(n) {
            return n, nil
        }
        if n < domainMax {
            n++
        } else {
            n = domainMin
        }
    }
    return 0, fmt.Errorf("no available numbers in domain")
}
```

Example: basic usage when nothing is reserved and the ID has availability

```go
n, _ := AssignNumber("TX_HASH", func(int) bool { return false })
fmt.Println("Number: #", n)
```

Notes
- Verify locally using the code above. Call AssignNumber(txHashHex, func(int) bool { return false }) and compare the result.

## Traits and Upgrade (user initiated)

Overview
- There is no global reveal. Traits and rarity are assigned only when the holder triggers Upgrade in the Mini App.
- The Mini App prompts a small on-chain tx (example 0.01 TON). Fees are non-refundable.
- The tx hash from this Upgrade is used as the seed input to a public, deterministic algorithm that assigns traits and rarity.

Inputs
- upgrade_tx_hash: transaction hash of the holder's Upgrade transaction sent from their wallet. Use the 64-character hex string as shown by explorers, case insensitive.

Procedure
1. At least 7 days before enabling Upgrade, we publish the exact code that defines:
   - trait catalogs and weights
   - PRNG choice and seeding scheme
   - mapping from token number to trait rolls given upgrade_tx_hash
2. After the public announcement, Upgrade becomes available in the Mini App.
3. When a holder taps Upgrade and the tx is finalized on chain, the published algorithm computes traits and rarity from upgrade_tx_hash.
4. The resulting traits are written to metadata and become visible on marketplaces and in app.

Determinism and fairness
- The algorithm and weights are published in advance for audit.
- The per-token seed is the holder's own upgrade_tx_hash, which is public and not under team control.
- Neither the holder nor the team can know the exact outcome before the tx is finalized, since the final tx hash is not known until inclusion.

Verification
- Anyone can recompute traits for a given token by running the published code with the observed upgrade_tx_hash and token number.

## Auctions for #1..#10

- Numbers #1..#10 are reserved for an on-chain auction.
- Auctions will use the official Getgems Auction contracts.
- All auctions will be fully on chain and publicly verifiable.

## Marketplace vs claims

- Marketplace purchase: you pick any available number in #11..#10000.
- Claim assignment: your number is assigned by the deterministic algorithm.
- If the algorithm outputs a number that was just bought, the assignment uses the next available number with wrap.

## Verification guide

You can independently verify a claim assignment.

Steps
1. Locate the tx_hash of your claim payment on a TON explorer.
2. Read the sale state at the block height for your claim.
3. Run the reference code using your tx_hash and an is_taken implementation bound to that sale state.
4. Confirm that the resulting number matches what you received.

You can also verify a trait assignment after Upgrade by running the published trait algorithm with your upgrade_tx_hash and token number.

We will ship
- CLI: pfp-verify

## Contracts and addresses

- Collection address: TBA
- Relayer address: TBA

All exact addresses and hashes will be pinned here before the sale goes live.

## Security notes

- Deterministic randomness is for transparency, not secrecy. Everyone can recompute the same result.
- Claim number assignment uses a public tx hash from the claim payment. Trait assignment uses a public tx hash from the holder's Upgrade.
- The Relayer never holds user private keys and only holds NFTs transiently during transfer.
- Upgrade fees are non-refundable because they are network fees.

## FAQ

Q: Can I choose a number during a claim  
A: No. Claims are assigned by the deterministic algorithm.

Q: Can I choose a number during a marketplace purchase  
A: Yes. Any available number from #11..#10000.

Q: When do my traits appear  
A: Only after you trigger Upgrade in the Mini App and the upgrade transaction is finalized.

Q: Can I postpone Upgrade  
A: Yes. You can hold an unrevealed token and upgrade later. Traits are not assigned until you upgrade.

Q: What happens if my assigned number is bought at the same time  
A: The assignment increments to the next available number with wrap. This rule is deterministic.

Q: Why use transaction hashes for randomness  
A: They are public, immutable, and not under team control. They provide fair seeds for deterministic assignment.

Q: Is the trait assignment fair  
A: Yes. The code is published in advance and the per-token seed is your own upgrade_tx_hash. Anyone can reproduce the exact mapping.

## Changelog

- 2025-11-03: initial draft
