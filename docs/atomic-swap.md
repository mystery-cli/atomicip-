# Atomic Swap Flow

This document describes the trustless patent sale mechanism in AtomicIP.

## Overview

An **atomic swap** allows a seller to exchange an IP decryption key for payment in a single transaction — if the key is invalid, the payment fails automatically. No escrow, no intermediary, no counterparty risk.

---

## Swap Lifecycle

```
┌─────────┐       ┌─────────┐       ┌──────────┐       ┌───────────┐
│ Pending │  -->  │Accepted │  -->  │Completed │       │ Cancelled │
└─────────┘       └─────────┘       └──────────┘       └───────────┘
     │                 │                                      ▲
     │                 └──────────────────────────────────────┘
     └────────────────────────────────────────────────────────┘
```

| State | Description |
|---|---|
| **Pending** | Seller has initiated the swap; buyer has not yet accepted |
| **Accepted** | Buyer has sent payment; waiting for seller to reveal key |
| **Completed** | Seller revealed valid key; payment released; IP transferred |
| **Cancelled** | Swap aborted by seller (if Pending) or buyer (if Accepted + expired) |

---

## Sequence Diagram

```
Seller                  AtomicSwap Contract              IpRegistry              Buyer
  │                            │                            │                      │
  │ 1. initiate_swap()         │                            │                      │
  ├───────────────────────────>│                            │                      │
  │                            │ verify IP ownership        │                      │
  │                            ├───────────────────────────>│                      │
  │                            │<───────────────────────────┤                      │
  │                            │ create SwapRecord          │                      │
  │                            │ status = Pending           │                      │
  │<───────────────────────────┤                            │                      │
  │                            │                            │                      │
  │                            │         2. accept_swap()   │                      │
  │                            │<───────────────────────────┼──────────────────────┤
  │                            │ transfer payment to contract                      │
  │                            │ status = Accepted          │                      │
  │                            ├────────────────────────────┼──────────────────────>│
  │                            │                            │                      │
  │ 3. reveal_key()            │                            │                      │
  ├───────────────────────────>│                            │                      │
  │                            │ verify_commitment()        │                      │
  │                            ├───────────────────────────>│                      │
  │                            │<───────────────────────────┤                      │
  │                            │ if valid:                  │                      │
  │                            │   transfer payment to seller                      │
  │                            │   transfer IP to buyer     │                      │
  │                            │   status = Completed       │                      │
  │<───────────────────────────┤                            │                      │
  │                            │                            │                      │
  │                            │ if invalid:                │                      │
  │                            │   refund buyer             │                      │
  │                            │   status = Cancelled       │                      │
  │                            ├────────────────────────────┼──────────────────────>│
```

---

## Step-by-Step Flow

### 1. Seller Initiates Swap

```rust
let swap_id = atomic_swap.initiate_swap(
    token,        // Payment token address (e.g., XLM)
    ip_id,        // The IP to sell
    seller,       // Seller's address (requires auth)
    price,        // Price in stroops (1 XLM = 10^7 stroops)
    buyer,        // Buyer's address
);
```

**Checks:**
- Seller must own the IP (`IpRegistry.get_ip(ip_id).owner == seller`)
- IP must not be revoked
- No other active swap exists for this `ip_id`
- Price must be > 0

**Result:**
- Swap created with `status = Pending`
- Expiry set to ~7 days from now

---

### 2. Buyer Accepts Swap

```rust
atomic_swap.accept_swap(swap_id);
```

**Checks:**
- Swap must be in `Pending` state
- Buyer must authorize the transaction
- Buyer must have sufficient token balance

**Result:**
- Payment transferred from buyer to contract
- Swap status updated to `Accepted`
- `accept_timestamp` recorded

---

### 3. Seller Reveals Key

```rust
atomic_swap.reveal_key(swap_id, secret, blinding_factor);
```

**Checks:**
- Swap must be in `Accepted` state
- Only seller can call this
- `verify_commitment(ip_id, secret, blinding_factor)` must return `true`

**Result if key is valid:**
- Payment released to seller
- IP ownership transferred to buyer
- Swap status updated to `Completed`

**Result if key is invalid:**
- Payment refunded to buyer
- Swap status updated to `Cancelled`

---

### 4. Cancellation Paths

#### Seller Cancels (Pending Only)

```rust
atomic_swap.cancel_swap(swap_id);
```

Only allowed if swap is still `Pending` (buyer has not yet accepted).

#### Buyer Cancels (Accepted + Expired)

```rust
atomic_swap.cancel_swap(swap_id);
```

Only allowed if:
- Swap is in `Accepted` state
- Current time > `expiry` timestamp
- Seller has not called `reveal_key`

This protects buyers from sellers who accept payment but never reveal the key.

---

## Security Properties

| Property | Enforcement |
|---|---|
| **Atomicity** | Payment and key exchange happen in the same transaction — no partial completion |
| **Trustlessness** | Smart contract verifies the key; no human arbitrator needed |
| **No Escrow Risk** | Payment held by contract, not a third party |
| **Expiry Protection** | Buyers can reclaim funds if seller abandons the swap |
| **Invalid Key Refund** | If `verify_commitment` fails, buyer is automatically refunded |

---

## Example: Full Swap Execution

```rust
// 1. Seller initiates
let swap_id = swap_contract.initiate_swap(
    xlm_token_address,
    ip_id,
    seller_address,
    100_000_000, // 10 XLM
    buyer_address,
);

// 2. Buyer accepts (sends 10 XLM to contract)
swap_contract.accept_swap(swap_id);

// 3. Seller reveals key
swap_contract.reveal_key(swap_id, secret, blinding_factor);

// If key is valid:
//   - Seller receives 10 XLM
//   - Buyer receives IP ownership
//   - Swap status = Completed
```

---

## Common Failure Scenarios

| Scenario | Outcome |
|---|---|
| Seller reveals invalid key | Buyer refunded; swap cancelled |
| Seller never reveals key | Buyer cancels after expiry; refunded |
| Buyer never accepts | Seller cancels; no payment involved |
| IP is revoked before swap completes | `initiate_swap` panics; swap cannot be created |

---

## Gas Optimization

- Use `initiate_swap` once per IP sale (not per negotiation attempt)
- Batch multiple IP sales if selling to the same buyer
- Cancel pending swaps promptly to free storage

---

## Related Documentation

- [Commitment Scheme](commitment-scheme.md) — How to construct valid secrets
- [Security Considerations](security.md) — Best practices for key management
- [Threat Model](threat-model.md) — Attack vectors and mitigations

---

## Batch Operations (#469)

Batch functions allow a seller or buyer to initiate, accept, or complete multiple swaps in a single transaction, reducing fees and round-trips.

### batch_initiate_swap

Seller initiates multiple patent sales at once. All swaps share the same buyer and payment token.

```rust
let swap_ids: Vec<u64> = swap_contract.batch_initiate_swap(
    token,       // Payment token (same for all swaps)
    ip_ids,      // Vec of IP IDs to sell
    seller,      // Seller address (requires auth)
    prices,      // Vec of prices — prices[i] corresponds to ip_ids[i]
    buyer,       // Buyer address
    0,           // required_approvals (0 = none)
    None,        // referrer
);
```

**Constraints:**
- `ip_ids.len() == prices.len()`
- Seller must own every IP in `ip_ids`
- No active swap may exist for any of the IPs
- All prices must be > 0

**Result:** Returns a `Vec<u64>` of the newly created swap IDs, one per IP.

---

### batch_accept_swaps

Buyer accepts multiple Pending swaps in one call. Payment for each swap is transferred to the contract.

```rust
swap_contract.batch_accept_swaps(
    swap_ids,  // Vec of swap IDs to accept
    buyer,     // Buyer address (requires auth)
);
```

**Constraints:**
- Every swap must be in `Pending` state
- `buyer` must match the `buyer` field on each swap
- Required approvals (if any) must already be collected

**Result:** All swaps move to `Accepted`. A single `BatchAccepted` event is emitted.

---

### batch_reveal_keys

Seller reveals decryption keys for multiple Accepted swaps in one call. Each key is verified; payment is released per swap.

```rust
swap_contract.batch_reveal_keys(
    swap_ids,         // Vec of swap IDs
    secrets,          // Vec of secrets — secrets[i] for swap_ids[i]
    blinding_factors, // Vec of blinding factors
    seller,           // Seller address (requires auth)
);
```

**Constraints:**
- `swap_ids`, `secrets`, and `blinding_factors` must all have the same length
- Every swap must be in `Accepted` state
- Seller must be the initiator of every swap
- Every `verify_commitment(ip_id, secret, blinding_factor)` must return `true`

**Result:** All swaps move to `Completed`. Protocol fees are deducted per swap. A single `BatchKeysRevealed` event is emitted.

---

### Batch Flow Example

```rust
// 1. Seller lists three IPs for sale in one transaction
let swap_ids = swap_contract.batch_initiate_swap(
    xlm_token,
    vec![ip_id_1, ip_id_2, ip_id_3],
    seller,
    vec![100_000_000, 200_000_000, 50_000_000],
    buyer,
    0,
    None,
);

// 2. Buyer accepts all three (sends total payment in one call)
swap_contract.batch_accept_swaps(swap_ids.clone(), buyer);

// 3. Seller reveals all three keys (completes all swaps in one call)
swap_contract.batch_reveal_keys(
    swap_ids,
    vec![secret_1, secret_2, secret_3],
    vec![blinding_1, blinding_2, blinding_3],
    seller,
);
```

### Events

| Event | Symbol | Emitted by |
|---|---|---|
| `BatchAcceptedEvent` | `btch_acp` | `batch_accept_swaps` |
| `BatchKeysRevealedEvent` | `btch_key` | `batch_reveal_keys` |

Individual `SwapInitiatedEvent` events are still emitted per swap inside `batch_initiate_swap`.
