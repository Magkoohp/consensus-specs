# Electra Light Client -- Fork Logic

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Helper functions](#helper-functions)
  - [`normalize_merkle_branch`](#normalize_merkle_branch)
- [Upgrading light client data](#upgrading-light-client-data)
- [Upgrading the store](#upgrading-the-store)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

This document describes how to upgrade existing light client objects based on the [Deneb specification](../../deneb/light-client/sync-protocol.md) to Electra. This is necessary when processing pre-Electra data with a post-Electra `LightClientStore`. Note that the data being exchanged over the network protocols uses the original format.

## Helper functions

### `normalize_merkle_branch`

```python
def normalize_merkle_branch(branch: Sequence[Bytes32],
                            gindex: GeneralizedIndex) -> Sequence[Bytes32]:
    depth = floorlog2(gindex)
    num_extra = depth - len(branch)
    return [Bytes32()] * num_extra + [*branch]
```

## Upgrading light client data

A Electra `LightClientStore` can still process earlier light client data. In order to do so, that pre-Electra data needs to be locally upgraded to Electra before processing.

```python
def upgrade_lc_header_to_electra(pre: deneb.LightClientHeader) -> LightClientHeader:
    return LightClientHeader(
        beacon=pre.beacon,
        execution=ExecutionPayloadHeader(
            parent_hash=pre.execution.parent_hash,
            fee_recipient=pre.execution.fee_recipient,
            state_root=pre.execution.state_root,
            receipts_root=pre.execution.receipts_root,
            logs_bloom=pre.execution.logs_bloom,
            prev_randao=pre.execution.prev_randao,
            block_number=pre.execution.block_number,
            gas_limit=pre.execution.gas_limit,
            gas_used=pre.execution.gas_used,
            timestamp=pre.execution.timestamp,
            extra_data=pre.execution.extra_data,
            base_fee_per_gas=pre.execution.base_fee_per_gas,
            block_hash=pre.execution.block_hash,
            transactions_root=pre.execution.transactions_root,
            withdrawals_root=pre.execution.withdrawals_root,
            blob_gas_used=pre.execution.blob_gas_used,
            excess_blob_gas=pre.execution.blob_gas_used,
            deposit_requests_root=Root(),  # [New in Electra:EIP6110]
            withdrawal_requests_root=Root(),  # [New in Electra:EIP7002:EIP7251]
            consolidation_requests_root=Root(),  # [New in Electra:EIP7251]
        ),
        execution_branch=pre.execution_branch,
    )
```

```python
def upgrade_lc_bootstrap_to_electra(pre: deneb.LightClientBootstrap) -> LightClientBootstrap:
    return LightClientBootstrap(
        header=upgrade_lc_header_to_electra(pre.header),
        current_sync_committee=pre.current_sync_committee,
        current_sync_committee_branch=normalize_merkle_branch(
            pre.current_sync_committee_branch, CURRENT_SYNC_COMMITTEE_GINDEX_ELECTRA),
    )
```

```python
def upgrade_lc_update_to_electra(pre: deneb.LightClientUpdate) -> LightClientUpdate:
    return LightClientUpdate(
        attested_header=upgrade_lc_header_to_electra(pre.attested_header),
        next_sync_committee=pre.next_sync_committee,
        next_sync_committee_branch=normalize_merkle_branch(
            pre.next_sync_committee_branch, NEXT_SYNC_COMMITTEE_GINDEX_ELECTRA),
        finalized_header=upgrade_lc_header_to_electra(pre.finalized_header),
        finality_branch=normalize_merkle_branch(
            pre.finality_branch, FINALIZED_ROOT_GINDEX_ELECTRA),
        sync_aggregate=pre.sync_aggregate,
        signature_slot=pre.signature_slot,
    )
```

```python
def upgrade_lc_finality_update_to_electra(pre: deneb.LightClientFinalityUpdate) -> LightClientFinalityUpdate:
    return LightClientFinalityUpdate(
        attested_header=upgrade_lc_header_to_electra(pre.attested_header),
        finalized_header=upgrade_lc_header_to_electra(pre.finalized_header),
        finality_branch=normalize_merkle_branch(
            pre.finality_branch, FINALIZED_ROOT_GINDEX_ELECTRA),
        sync_aggregate=pre.sync_aggregate,
        signature_slot=pre.signature_slot,
    )
```

```python
def upgrade_lc_optimistic_update_to_electra(pre: deneb.LightClientOptimisticUpdate) -> LightClientOptimisticUpdate:
    return LightClientOptimisticUpdate(
        attested_header=upgrade_lc_header_to_electra(pre.attested_header),
        sync_aggregate=pre.sync_aggregate,
        signature_slot=pre.signature_slot,
    )
```

## Upgrading the store

Existing `LightClientStore` objects based on Deneb MUST be upgraded to Electra before Electra based light client data can be processed. The `LightClientStore` upgrade MAY be performed before `ELECTRA_FORK_EPOCH`.

```python
def upgrade_lc_store_to_electra(pre: deneb.LightClientStore) -> LightClientStore:
    if pre.best_valid_update is None:
        best_valid_update = None
    else:
        best_valid_update = upgrade_lc_update_to_electra(pre.best_valid_update)
    return LightClientStore(
        finalized_header=upgrade_lc_header_to_electra(pre.finalized_header),
        current_sync_committee=pre.current_sync_committee,
        next_sync_committee=pre.next_sync_committee,
        best_valid_update=best_valid_update,
        optimistic_header=upgrade_lc_header_to_electra(pre.optimistic_header),
        previous_max_active_participants=pre.previous_max_active_participants,
        current_max_active_participants=pre.current_max_active_participants,
    )
```