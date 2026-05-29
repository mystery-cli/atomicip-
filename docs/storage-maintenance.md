# Storage Maintenance

This document describes the four storage maintenance features added to the IP Registry contract.

## 1. Cleanup Expired/Revoked Commitments

**Function:** `cleanup_revoked_commitment(ip_id)`

Removes a revoked IP record and its commitment-owner index entry from persistent storage, freeing ledger space. Only the record owner may call this.

- The IP must already be revoked before cleanup is allowed.
- The owner's ID list is updated to remove the cleaned-up entry.
- Emits a `cleanup` event on success.

```rust
// Revoke first, then clean up
registry.revoke_ip(&ip_id);
registry.cleanup_revoked_commitment(&ip_id);
```

## 2. Periodic Snapshots for Disaster Recovery

**Functions:** `create_snapshot(caller)`, `get_snapshot(snapshot_id)`

Creates a lightweight snapshot of the registry state for disaster recovery. Each snapshot records:

- `snapshot_id` — monotonically increasing ID
- `timestamp` — ledger timestamp at creation
- `total_count` — number of IPs committed so far
- `checksum` — sha256 of the current `NextId` counter (state fingerprint)

Admin-only. Snapshot IDs start at 1 and increment with each call.

```rust
let snap_id = registry.create_snapshot(&admin);
let snap = registry.get_snapshot(&snap_id).unwrap();
assert_eq!(snap.total_count, expected_count);
```

## 3. Cryptographic Checksum Integrity Verification

**Functions:** `compute_integrity_checksum(caller)`, `verify_integrity_checksum()`

Computes a sha256 checksum over all **active** (non-revoked) commitment hashes in ID order and stores it under `CommitmentChecksumV2`. Admin-only for computation.

`verify_integrity_checksum()` recomputes the checksum and compares it to the stored value. Returns `true` if they match or no checksum has been stored yet.

```rust
// After any state change, recompute and verify
let checksum = registry.compute_integrity_checksum(&admin);
assert!(registry.verify_integrity_checksum());
```

Revoked commitments are excluded from the checksum, so revoking an IP changes the checksum.

## 4. Batch Expire Commitments

**Function:** `batch_revoke_commitments(owner, ip_ids)`

Revokes multiple IP commitments in a single transaction. The caller must own every IP in the list. All IPs are revoked atomically — if any check fails the entire transaction panics.

Returns the number of IPs revoked.

```rust
let ids = Vec::from_array(&env, [id1, id2, id3]);
let count = registry.batch_revoke_commitments(&owner, &ids);
assert_eq!(count, 3);
```

Each revoked IP emits a `revoked` event and an immutable audit entry.
