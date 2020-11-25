# Ethereum 2.0 Light Client Support

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Constants](#constants)
- [Configuration](#configuration)
  - [Misc](#misc)
  - [Time parameters](#time-parameters)
  - [Domain types](#domain-types)
- [Containers](#containers)
  - [Extended containers](#extended-containers)
    - [`BeaconBlockBody`](#beaconblockbody)
    - [`BeaconState`](#beaconstate)
  - [New containers](#new-containers)
    - [`SyncCommittee`](#synccommittee)
- [Helper functions](#helper-functions)
  - [`Predicates`](#predicates)
    - [`optional_fast_aggregate_verify`](#optional_fast_aggregate_verify)
  - [Beacon state accessors](#beacon-state-accessors)
    - [`get_sync_committee_indices`](#get_sync_committee_indices)
    - [`get_sync_committee`](#get_sync_committee)
  - [Block processing](#block-processing)
    - [Sync committee processing](#sync-committee-processing)
  - [Epoch processing](#epoch-processing)
    - [Final updates](#final-updates)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

This is a standalone beacon chain patch adding light client support via sync committees.

## Constants

| Name | Value |
| - | - | 
| `BASE_REWARDS_PER_EPOCH` | `uint64(5)` |

## Configuration

### Misc

| Name | Value |
| - | - | 
| `SYNC_COMMITTEE_SIZE` | `uint64(2**10)` (= 1024) |
| `SYNC_COMMITTEE_PUBKEY_AGGREGATES_SIZE` | `uint64(2**6)` (= 64) |
| `G2_INFINITY_POINT_SIG` | `BLSSignature(b'\xc0' + b'\x00' * 95)` |

### Time parameters

| Name | Value | Unit | Duration |
| - | - | :-: | :-: |
| `EPOCHS_PER_SYNC_COMMITTEE_PERIOD` | `Epoch(2**8)` (= 256) | epochs | ~27 hours |

### Domain types

| Name | Value |
| - | - |
| `DOMAIN_SYNC_COMMITTEE` | `DomainType('0x07000000')` |

## Containers

### Extended containers

#### `BeaconBlockBody`

```python
class BeaconBlockBody(phase0.BeaconBlockBody):
    # Sync committee aggregate signature
    sync_committee_bits: Bitvector[SYNC_COMMITTEE_SIZE]
    sync_committee_signature: BLSSignature
```

#### `BeaconState`

```python
class BeaconState(phase0.BeaconState):
    # Sync committees
    current_sync_committee: SyncCommittee
    next_sync_committee: SyncCommittee
```

### New containers

#### `SyncCommittee`

```python
class SyncCommittee(Container):
    pubkeys: Vector[BLSPubkey, SYNC_COMMITTEE_SIZE]
    pubkey_aggregates: Vector[BLSPubkey, SYNC_COMMITTEE_SIZE // SYNC_COMMITTEE_PUBKEY_AGGREGATES_SIZE]
```

## Helper functions

### `Predicates`

#### `optional_fast_aggregate_verify`

```python
def optional_fast_aggregate_verify(pubkeys: Sequence[BLSPubkey], message: Bytes32, signature: BLSSignature) -> bool:
    """
    If ``pubkeys`` is an empty list, the given ``signature`` should be a stub ``G2_INFINITY_POINT_SIG``.
    Otherwise, verify it with standard BLS FastAggregateVerify API.
    """
    if len(pubkeys) == 0:
        return signature == G2_INFINITY_POINT_SIG
    else:
        return bls.FastAggregateVerify(pubkeys, message, signature)
```

### Beacon state accessors

#### `get_sync_committee_indices`

```python
def get_sync_committee_indices(state: BeaconState, epoch: Epoch) -> Sequence[ValidatorIndex]:
    """
    Return the sync committee indices for a given state and epoch.
    """
    MAX_RANDOM_BYTE = 2**8 - 1
    base_epoch = Epoch((max(epoch // EPOCHS_PER_SYNC_COMMITTEE_PERIOD, 1) - 1) * EPOCHS_PER_SYNC_COMMITTEE_PERIOD)
    active_validator_indices = get_active_validator_indices(state, base_epoch)
    active_validator_count = uint64(len(active_validator_indices))
    seed = get_seed(state, base_epoch, DOMAIN_SYNC_COMMITTEE)
    i = 0
    sync_committee_indices: List[ValidatorIndex] = []
    while len(sync_committee_indices) < SYNC_COMMITTEE_SIZE:
        shuffled_index = compute_shuffled_index(uint64(i % active_validator_count), active_validator_count, seed)
        candidate_index = active_validator_indices[shuffled_index]
        random_byte = hash(seed + uint_to_bytes(uint64(i // 32)))[i % 32]
        effective_balance = state.validators[candidate_index].effective_balance
        if effective_balance * MAX_RANDOM_BYTE >= MAX_EFFECTIVE_BALANCE * random_byte:
            sync_committee_indices.append(candidate_index)
        i += 1
    return sync_committee_indices
```

#### `get_sync_committee`

```python
def get_sync_committee(state: BeaconState, epoch: Epoch) -> SyncCommittee:
    """
    Return the sync committee for a given state and epoch.
    """
    indices = get_sync_committee_indices(state, epoch)
    validators = [state.validators[index] for index in indices]
    pubkeys = [validator.pubkey for validator in validators]
    aggregates = [
        bls.AggregatePKs(pubkeys[i:i + SYNC_COMMITTEE_PUBKEY_AGGREGATES_SIZE])
        for i in range(0, len(pubkeys), SYNC_COMMITTEE_PUBKEY_AGGREGATES_SIZE)
    ]
    return SyncCommittee(pubkeys, aggregates)
```

### Block processing

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    phase0.process_block(state, block)
    process_sync_committee(state, block.body)
```

#### Sync committee processing

```python
def process_sync_committee(state: BeaconState, body: BeaconBlockBody) -> None:
    # Verify sync committee aggregate signature signing over the previous slot block root
    previous_slot = max(Slot(state.slot), Slot(1)) - Slot(1)
    committee_indices = get_sync_committee_indices(state, get_current_epoch(state))
    participant_indices = [committee_indices[i] for i in range(len(committee_indices)) if body.sync_committee_bits[i]]
    participant_pubkeys = [state.validators[participant_index].pubkey for participant_index in participant_indices]
    domain = get_domain(state, DOMAIN_SYNC_COMMITTEE, compute_epoch_at_slot(previous_slot))
    signing_root = compute_signing_root(get_block_root_at_slot(state, previous_slot), domain)
    assert optional_fast_aggregate_verify(participant_pubkeys, signing_root, body.sync_committee_signature)

    # Reward sync committee participants
    participant_rewards = Gwei(0)
    active_validator_count = uint64(len(get_active_validator_indices(state, get_current_epoch(state))))
    for participant_index in participant_indices:
        base_reward = get_base_reward(state, participant_index)
        reward = Gwei(base_reward * active_validator_count // len(committee_indices) // SLOTS_PER_EPOCH)
        increase_balance(state, participant_index, reward)
        participant_rewards += reward

    # Reward beacon proposer
    increase_balance(state, get_beacon_proposer_index(state), Gwei(participant_rewards // PROPOSER_REWARD_QUOTIENT))
```

### Epoch processing

#### Final updates

```python
def process_final_updates(state: BeaconState) -> None:
    phase0.process_final_updates(state)
    next_epoch = get_current_epoch(state) + Epoch(1)
    if next_epoch % EPOCHS_PER_SYNC_COMMITTEE_PERIOD == 0:
        state.current_sync_committee = state.next_sync_committee
        state.next_sync_committee = get_sync_committee(state, next_epoch + EPOCHS_PER_SYNC_COMMITTEE_PERIOD)
```