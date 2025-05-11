# Indexer Improvement Brainstorm

Currently, the `Indexer` trait is too brittle. How can we redesign it be more useful and support all possible indexers?

## Moving Parts

* **Data types to index:** We should index both full transactions and floating txouts. The current canonicalization algorithm only considers full transactions. However, if we wish to include floating txouts in our UTXO set, we need to have these floating txouts indexed.
* **How to categorize updates types:** Updates (applied to `TxGraph`/`IndexedTxGraph`) can be categorized into these sub-categories:
  * Needs filtering v.s. no filtering needed: Updates that do not need filtering (`TxUpdate`), and updates that require filtering (`Block`, mempool transactions, etc.) before being inserted into the `TxGraph`. The way we filter data is different depending on the data type:
    * Transactions - We should only include those that the indexer considers "relevant".
    * Floating txouts - We should only include those that the indexer considers "relevant" or are prevouts of txs that we have already included in the `TxGraph` (for fee calculation).
    * Associated data (anchors, timestamps) - We should only include those who are associated with transactions that are/will-be included in the `TxGraph`.
  * Chronological vs non-chronological. Transactions coming from spk-based chain sources are not guaranteed to be chronological (these typically arrive as `TxUpdate`s). Transactions in blocks are chronological, and relevancy-detection can be optimized as transaction-prevouts are guaranteed to be visited before the transaction itself.

## Indexer Design Considerations

* An indexer may only support certain update types. For example, a silent-payment-indexer needs updates to have transactions with the prev-txouts attached. Whereas a `KeychainTxOutIndex` can support a broader set of update types. In the future, there may be indexers that require updates with more data attached (who knows?).
* We want indexers to be composable. I.e. a silent-payment-indexer and a descriptor-indexer may want to exist in the same wallet and share the same `TxGraph`. Both these indexers will support block updates with prev-txout data attached and each update should be passed through both these indexers to determine relevancy before including transactions (and associated data) into `TxGraph`.

## Ideas

```rust
/// A trait representing a persistable index of transaction data.
///
/// This index maintains internal state that can be updated incrementally and persisted.
pub trait TxIndex<A> {
    /// A set of changes that can be applied to or produced by this index.
    type ChangeSet: Merge;

    /// Reindex the internal state using the provided [`TxGraph`].
    ///
    /// This assumes that all relevant updates have already been passed through
    /// [`TxIndexer::index_and_apply`].
    fn reindex(&mut self, tx_graph: &TxGraph<A>);

    /// Returns whether the given transaction is relevant to this index.
    ///
    /// This is typically used during filtering to determine whether a transaction should be
    /// retained and persisted.
    fn is_relevant(&self, tx: &Transaction) -> bool;

    /// Applies the given changeset to the index.
    fn apply_changeset(&mut self, changeset: Self::ChangeSet);

    /// Computes the [`ChangeSet`](TxIndex::ChangeSet) required to represent the entire current
    /// state.
    fn initial_changeset(&self) -> Self::ChangeSet;
}

/// Indexes an update type `U` that does not require filtering and can be applied directly to
/// `TxGraph`.
pub trait TxIndexer<A, U = TxUpdate<A>>: TxIndex<A>
where
    U: Into<TxUpdate<A>>,
{
    /// Indexes an `update` before being applied to `TxGraph`.
    fn index(&mut self, changeset: &mut Self::ChangeSet, update: &U);
}

/// Indexes an update type `U` that requires filtering before it can be applied to `TxGraph`.
pub trait TxFilteringIndexer<A, U>: TxIndex<A> {
    /// Indexes and transforms an `update` into an `TxUpdate` (with filtering) to be applied to 
    /// `TxGraph`.
    fn index_and_transform(&mut self, changeset: &mut Self::ChangeSet, update: &U) -> TxUpdate<A>;
}
```

