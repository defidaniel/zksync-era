# DAL (Data access Layer)

This crate provides read and write access to the main database (which is Postgres), that acts as a primary source of
truth.

Current schema is managed by `sqlx`. Schema changes are stored in the [`migrations`](migrations) directory.

## Schema overview

_This overview skips prover-related and Ethereum sender-related tables, which are specific to the main node._

### Miniblocks and L1 batches

- `miniblocks`. Stores miniblock headers.

- `miniblocks_consensus`. Stores miniblock data related to the consensus algorithm used by the decentralized sequencer.
  Tied one-to-one to miniblocks (the consensus side of the relation is optional).

- `l1_batches`. Stores L1 batch headers.

- `commitments`. Stores a part of L1 batch commitment data (event queue and bootloader memory commitments). In the
  future, other commitment-related columns will be moved here from `l1_batches`.

### Transactions

- `transactions`. Stores all transactions received by the node, both L2 and L1 ones. Transactions in this table are not
  necessarily included into a miniblock; i.e., the table is used as a persistent mempool as well.

### VM storage

See [`zksync_state`] crate for more context.

- `storage_logs`. Stores all the VM storage write logs for all transactions, as well as non-transaction writes generated
  by the bootloader. This is the source of truth for the VM storage; all other VM storage implementations (see the
  [`zksync_state`] crate) are based on it (e.g., by adding persistent or in-memory caching). Used by multiple components
  including Metadata calculator, Commitment generator, API server (both for reading one-off values like account balance,
  and as a part of the VM sandbox) etc.

- `initial_writes`. Stores initial writes information for each L1 batch, i.e., the enumeration index assigned for each
  key. Used when generating L1 batch metadata in Metadata calculator and Commitment generator components, and in the VM
  sandbox in API server for fee estimation.

- `protective_reads`. Stores protective read information for each L1 batch, i.e., keys influencing VM execution for the
  batch that were not modified. Used when generating L1 batch metadata in Commitment generator.

- `factory_deps`. Stores bytecodes of all deployed L2 contracts.

- `storage`. **Obsolete, going to be removed; must not be used in new code.**

### Other VM artifacts

- `events`. Stores all events (aka logs) emitted by smart contracts during VM execution.

- `l2_to_l1_logs`. Stores L2-to-L1 logs emitted by smart contracts during VM execution.

- `call_traces`. Stores call traces for transactions emitted during VM execution. (Unlike with L1 node implementations,
  in Era call traces are currently proactively generated for all transactions.)

- `tokens`. Stores all ERC-20 tokens registered in the L1–L2 bridge.

- `transaction_traces`. **Obsolete, going to be removed; must not be used in new code.**

### Snapshot generation and recovery

See [`snapshots_creator`] and [`snapshots_applier`] crates for the overview of application-level nodes snapshots.

- `snapshots`. Stores metadata for all snapshots generated by `snapshots_creator`, such as the L1 batch of the snapshot.

- `snapshot_recovery`. Stores metadata for the snapshot used during node recovery, if any. Currently, this table is
  expected to have no more than one row.

## Logical invariants

In addition to foreign key constraints and other constraints manifested directly in the DB schema, the following
invariants are expected to be upheld:

- If a header is present in the `miniblocks` table, it is expected that the DB contains all artifacts associated with
  the miniblock execution, such as `events`, `l2_to_l1_logs`, `call_traces`, `tokens` etc. (See State keeper I/O logic
  for the exact definition of these artifacts.)
- Likewise, if a header is present in the `l1_batches` table, all artifacts associated with the L1 batch execution are
  also expected in the DB, e.g. `initial_writes` and `protective_reads`. (See State keeper I/O logic for the exact
  definition of these artifacts.)
- Miniblocks and L1 batches present in the DB form a continuous range of numbers. If a DB is recovered from a node
  snapshot, the first miniblock / L1 batch is **the next one** after the snapshot miniblock / L1 batch mentioned in the
  `snapshot_recovery` table. Otherwise, miniblocks / L1 batches must start from number 0 (aka genesis).

## Contributing to DAL

Some tips and tricks to make contributing to DAL easier:

- If you want to add a new DB query, search the DAL code or the [`.sqlx`](.sqlx) directory for the identical /
  equivalent queries. Reuse is almost always better than duplication.
- It usually makes sense to instrument your queries using [`instrument`](src/instrument.rs) tooling. See the
  `instrument` module docs for details.
- It's best to cover added queries with unit tests to ensure they work and don't break in the future. `sqlx` has
  compile-time schema checking, but it's not a panacea.
- If there are doubts as to the query performance, run a query with [`EXPLAIN`] / `EXPLAIN ANALYZE` prefixes against a
  production-size database.

### Backward compatibility

All DB schema changes are expected to be backward-compatible. That is, _old_ code must be able to function with the
_new_ schema. As an example, dropping / renaming columns is not allowed. Instead, a 2-phase migration should be used:

1. The column should be marked as obsolete, with its mentions replaced in all queries. If the column should be renamed,
   a new column should be created and data (if any) should be copied from the old column (see also:
   [_Programmatic migrations_](#programmatic-migrations)).
2. After a significant delay (order of months), the old column may be removed in a separate migration.

### Programmatic migrations

We cannot afford non-trivial amount of downtime caused by a data migration. That is, if a migration may cause such
downtime (e.g., it copies non-trivial amount of data), it must be organized as a programmatic migration and run in the
node background (perhaps, splitting work into chunks with a delay between them so that the migration doesn't hog all DB
resources).

[`zksync_state`]: ../state
[`snapshots_creator`]: ../../bin/snapshots_creator
[`snapshots_applier`]: ../snapshots_applier
[`explain`]: https://www.postgresql.org/docs/14/sql-explain.html
