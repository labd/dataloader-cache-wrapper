---
"@labdigital/dataloader-cache-wrapper": minor
---

Write cache misses with a single `setMany` instead of a `set` per key.

This is not a round-trip win: the individual writes were already pipelined by the
store's client, and the same commands and bytes go over the socket either way.
What changes is that the batch is one call into the store, so the store can write
it as a unit -- the redis adapter wraps it in a single MULTI -- and tracing shows
one batch instead of one command per key, which for a batch of 36 products was 36
spans of noise per request.

The writes are also awaited now, so a failing store surfaces as an `error` event
on it rather than as a floating rejection.

`setMany` requires keyv 5.3.4 or newer -- earlier versions hand unserialized
entries to the store adapter, which writes values `get` cannot read back. The
peer range moves from `>=4.0.0 <6.0.0` to `>=5.3.4 <6.0.0`.
