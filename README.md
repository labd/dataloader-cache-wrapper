# node-dataloader-cache-wrapper

[![npm](https://img.shields.io/npm/v/@labdigital/dataloader-cache-wrapper)](https://www.npmjs.com/package/@labdigital/dataloader-cache-wrapper)
[![CI](https://github.com/labd/dataloader-cache-wrapper/actions/workflows/release.yml/badge.svg)](https://github.com/labd/dataloader-cache-wrapper/actions/workflows/release.yml)

A wrapper around [DataLoader](https://github.com/graphql/dataloader) that adds
a shared cache, backed by a [Keyv](https://github.com/jaredwray/keyv) store,
in front of your batch load function. Keys are first looked up in the store.
Only the misses go to your batch function, and their results are written back
to the store with a TTL. Because any Keyv adapter works (Redis, in-memory and
others), results can be reused across requests and processes, not just within
one DataLoader instance.

## Installation

```sh
pnpm add @labdigital/dataloader-cache-wrapper dataloader keyv
# or
npm install @labdigital/dataloader-cache-wrapper dataloader keyv
```

## Requirements

- Node.js >= 22
- Peer dependencies:
  - `dataloader` `>=2.0 <3.0`
  - `keyv` `>=5.3.4 <7.0.0`

The package ships both ESM and CommonJS builds, with type definitions.

## Usage

### `DataLoaderCache`

`DataLoaderCache` has the same API as `DataLoader` (`load`, `loadMany`,
`prime`, `clear`, `clearAll`). It requires a `cacheKeyFn` that turns a key into
a string. That string is used both as the DataLoader key and as the key in the
cache store.

```ts
import { DataLoaderCache } from "@labdigital/dataloader-cache-wrapper";
import { Keyv } from "keyv";

type ProductReference = {
	store: string;
	locale: string;
	currency: string;
	slug: string;
};

type Product = { id: string; slug: string; name: string };

const fetchProducts = async (
	refs: readonly ProductReference[],
): Promise<(Product | null)[]> => {
	// Fetch the products from your API, in the same order as `refs`
	return refs.map(() => null);
};

const productLoader = new DataLoaderCache<ProductReference, Product>(
	fetchProducts,
	{
		cache: {
			storeFn: () => new Keyv(),
			ttl: 3600 * 1000,
		},
		cacheKeyFn: (ref) =>
			`product:${ref.store}-${ref.locale}-${ref.currency}:slug:${ref.slug}`,
		maxBatchSize: 50,
	},
);

const product = await productLoader.load({
	store: "nl",
	locale: "nl-NL",
	currency: "EUR",
	slug: "t-shirt",
});
```

On top of the DataLoader methods, `DataLoaderCache` has two methods that also
clear the cache store:

```ts
// Remove one key from the store and from the DataLoader memo cache
await productLoader.clearCache(ref);

// Clear the whole store (`keyv.clear()`) and the DataLoader memo cache
await productLoader.clearCacheAll();
```

### `dataloaderCache`

If you want to build your own `DataLoader`, use the lower level
`dataloaderCache` function inside your batch load function. Each key can map to
several cache keys through `cacheKeysFn`. All of them are read on lookup and
written on a miss.

```ts
import { dataloaderCache } from "@labdigital/dataloader-cache-wrapper";
import DataLoader from "dataloader";
import { Keyv } from "keyv";

const store = new Keyv<Product | null>();

export const createProductBySlugLoader = () =>
	new DataLoader<ProductReference, Product | null>(
		(keys) =>
			dataloaderCache<ProductReference, Product>({
				keys,
				store,
				ttl: 3600 * 1000,
				batchLoadFn: fetchProducts,
				cacheKeysFn: (ref) => [
					`product:${ref.store}-${ref.locale}-${ref.currency}:slug:${ref.slug}`,
				],
			}),
		{ maxBatchSize: 50 },
	);
```

`dataloaderCache` compares keys in O(n²) time, so it is meant for small batches.

## Configuration

### `DataLoaderCache` options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `cacheKeyFn` | `(key: K) => string` | — | Required. Builds the DataLoader key and the cache store key. |
| `cache` | `{ storeFn, ttl }` | — | Optional. Without it, no store is used and the loader acts like a plain DataLoader. |
| `cache.storeFn` | `() => Keyv` | — | Called once in the constructor to create the store. |
| `cache.ttl` | `number` | — | TTL passed to Keyv for every write. Keyv TTLs are in milliseconds. |
| `maxBatchSize` | `number` | `50` | Passed to DataLoader. |
| `name` | `string \| null` | `null` | Passed to DataLoader and exposed as `loader.name`. |

### `dataloaderCache` options (`cacheOptions<K, V>`)

| Option | Type | Description |
| --- | --- | --- |
| `keys` | `ReadonlyArray<K>` | The keys passed to the DataLoader batch function. |
| `batchLoadFn` | `DataLoader.BatchLoadFn<K,V>` | Called with the keys that were not found in the store. |
| `cacheKeysFn` | `(key: K) => string[]` | Returns the cache keys for a key. |
| `ttl` | `number` | TTL for written entries (milliseconds, as Keyv expects). |
| `store` | `Keyv` | Optional. Without a store, every key goes to `batchLoadFn`. |

### Caching behaviour

- Store lookups use a single `store.get(keys[])` call. Writes go through one
  `store.setMany()` call per batch.
- If your batch function returns `null` for a key, that `null` is cached too.
- `Error` values returned by the batch function are not cached, and that key
  resolves to `null`.
- Falsy results (`0`, `""`, `false`) are returned as `null`.

## Development

```sh
pnpm install
pnpm test     # vitest with coverage
pnpm build    # tsdown
pnpm lint     # biome check
pnpm tsc      # type check
pnpm check    # lint + type check
pnpm format   # biome format --fix
```

## Releasing

Releases are managed with [Changesets](https://github.com/changesets/changesets).
Add a changeset with `pnpm changeset` in your pull request. When it is merged to
`main`, the release workflow opens a version PR. Merging that PR publishes the
package to npm.

## License

[MIT](./LICENSE)
