[make.fun API](../README.md) / **TypeScript SDK**

> Generated from [make.fun/developers](https://make.fun/developers), docs version 1.1.1. An edit here is overwritten by the next update: changes belong upstream, in make.fun itself.

# TypeScript SDK

The [TypeScript SDK](https://github.com/makedotfun/makefun-sdk) makes these calls from your own code: every route typed with what it takes and what it answers, lists walked page by page, a 429 waited out for its `Retry-After`, and keys and entries signed with any Solana wallet. It has no dependencies and runs on Node 22.13 or later and in current browsers. Install it from npm:

```sh
npm install @makedotfun/sdk
```

Or from source, straight from its repository: `npm install github:makedotfun/makefun-sdk` (with npm 12 or newer, add `--allow-git=root`: those versions turn git dependencies off by default). Then list the bounties taking entries:

```ts
import { createClient } from '@makedotfun/sdk';

const client = createClient({ origin: 'https://make.fun' }); // no key: the free tier
for await (const bounty of client.bounties.iterate({ status: 'open' })) {
  console.log(bounty.slug, bounty.title);
}
```

Pass `apiKey` to act as your account. The [README](https://github.com/makedotfun/makefun-sdk) covers getting a key with a wallet, entering a bounty, live streams and the errors each refusal throws.

---

Next: [Authentication](authentication.md)
