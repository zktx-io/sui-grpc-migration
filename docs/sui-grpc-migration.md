# Sui SDK 1.x → 2.x Migration Guide (gRPC/GraphQL)

> **Package versions**: `@mysten/sui@^2.4.0` · `@mysten/dapp-kit-react@^1.0.2` · `@mysten/dapp-kit-core@^1.0.4`  
> **Scope**: `@mysten/sui` 1.x → 2.x (assumes `@mysten/sui.js` migration already completed)
> **Runtime requirement**: `@mysten/sui@2.4.0` requires Node `>=22`

---

## 0. Validation Baseline

Use your target project's installed `node_modules` as the local source of truth when validating generated migration code.

Validation precedence for this guide:
1. Installed SDK types/source (`node_modules/@mysten/*`) for exact API shape.
2. Official docs for migration direction and architecture guidance.
3. If docs and installed SDK conflict, follow installed SDK and annotate the mismatch.

- Installed baseline example: `@mysten/sui@2.4.0` · `@mysten/dapp-kit-react@1.0.2` · `@mysten/dapp-kit-core@1.0.4`
- gRPC client method surface: `node_modules/@mysten/sui/dist/grpc/client.d.mts`
- Execute/simulate option fields: `node_modules/@mysten/sui/dist/client/types.d.mts`
- Legacy JSON-RPC-only methods: `node_modules/@mysten/sui/dist/jsonRpc/client.d.mts` (verification-only, not migration target)
- React export surface: `node_modules/@mysten/dapp-kit-react/dist/index.mjs`

```bash
# Verify runtime baseline first (must be >= 22 for official support)
node -v

# Verify SuiGrpcClient prototype methods
node -e "import { SuiGrpcClient } from '@mysten/sui/grpc'; console.log(Object.getOwnPropertyNames(SuiGrpcClient.prototype).filter(k => k !== 'constructor').sort().join('\n'));"

# Verify React package exports using static parse
rg '^export' node_modules/@mysten/dapp-kit-react/dist/index.mjs
```

> `@mysten/dapp-kit-react` may fail direct Node runtime import in non-browser environments due to `window` usage.
> Use static export checks for CLI-side validation.

---

## 1. Background

This guide covers migration from `@mysten/sui` 1.x to 2.x.
`@mysten/sui.js` legacy naming (for example `executeTransactionBlock`) is out of scope.

- Transaction execution/simulation path → gRPC (`@mysten/sui/grpc`)
- React wallet integration → migrate to `@mysten/dapp-kit-react`
- Event/complex queries → separate into GraphQL or an indexer
- Migration policy: do not introduce `SuiJsonRpcClient` or imports from `@mysten/sui/jsonRpc`

Reference: https://sdk.mystenlabs.com/sui/clients/grpc

---

## 2. Pre-migration Scan

Run the following to identify files that need migration.

```bash
# Legacy dapp-kit traces
# @mysten/dapp-kit['"] — matches both quote styles, avoids false-positives on dapp-kit-react
rg "@mysten/dapp-kit['\"]" src -g "*.ts" -g "*.tsx"

# Backend migration traces (1.x -> 2.x)
rg "devInspectTransactionBlock|dryRunTransactionBlock|queryEvents|queryTransactionBlocks|multiGetTransactionBlocks|getOwnedObjects|multiGetObjects|getCoins" src

# Disallow in migrated output
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc|getJsonRpcFullnodeUrl" src -g "*.ts" -g "*.tsx"

# Legacy frontend hook traces
rg "useSignAndExecuteTransaction|useSuiClient" src -g "*.ts" -g "*.tsx"
```

> **Monorepo / non-standard layout** (`app/`, `server/`, `lib/`, etc.): replace `src` with your actual source directory.

---

## 3. Package Update

```bash
npm install @mysten/sui@^2.4.0 @mysten/dapp-kit-react @mysten/dapp-kit-core
npm uninstall @mysten/dapp-kit

# Optional cleanup:
# Remove react-query only if your app does not import/use it directly.
npm uninstall @tanstack/react-query
```

```diff
- "@mysten/dapp-kit": "^x.x"
+ "@mysten/dapp-kit-react": "^1.0.2"
+ "@mysten/dapp-kit-core": "^1.0.4"
  "@mysten/sui": "^2.4.0"   # bump to latest
```

> Legacy `@mysten/dapp-kit` required `@tanstack/react-query` as a peer dependency.
> `@mysten/dapp-kit-react` / `@mysten/dapp-kit-core` do not require it.
> Keep `@tanstack/react-query` only when your application code still uses it directly.
> Official migration docs still show `useMutation` / `useQuery` examples for advanced UX patterns.
> Conclusion: `@tanstack/react-query` is optional for dApp Kit React itself and useful as a UI state-management wrapper.
> In `@mysten/dapp-kit-react` 1.x, internal React state hooks are implemented with `nanostores` (`useStore`) rather than React Query hooks.

## 4. Backend Migration (Node.js)

### 4.1 Client Setup

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';

const client = new SuiGrpcClient({
  network: 'testnet', // 'mainnet' | 'testnet' | 'devnet' | 'localnet'
  baseUrl: 'https://fullnode.testnet.sui.io:443',
});
```

`fullnode.<network>.sui.io:443` is the official fullnode host used in gRPC client examples.
Use this `baseUrl` format for `SuiGrpcClient`.

### 4.2 Simulation / Execution API Replacement

```typescript
// simulate (replaces devInspect / dryRun)
const sim = await client.simulateTransaction({
  transaction: txBytes,
});

// If you used devInspect returnValues, include commandResults:
const simWithResults = await client.simulateTransaction({
  transaction: txBytes,
  include: { commandResults: true },
});
// Access results: simWithResults.commandResults?.[n].returnValues

// execute (method name remains executeTransaction in 1.x/2.x paths)
const exec = await client.executeTransaction({
  transaction: txBytes,   // field name: NOT "bytes"
  signatures: [signature], // string[], plural
});
```

### 4.3 Method Mapping

| 1.x pattern | 2.x target pattern |
|----------------------|-------------------|
| `devInspectTransactionBlock` | `client.simulateTransaction({ include: { commandResults: true } })` |
| `dryRunTransactionBlock` | `client.simulateTransaction` |
| `executeTransaction` | `client.executeTransaction({ transaction, signatures })` |
| `getCoins` | `client.listCoins` |
| `multiGetObjects` | `client.getObjects` |
| `getOwnedObjects` | `client.listOwnedObjects` |
| `getTransactionBlock` (digest loading) | GraphQL-first transaction query; avoid gRPC-only loaders |
| `queryEvents` / `queryTransactionBlocks` | GraphQL-first (especially long-term/unknown-age data) |

> `executeTransactionBlock` remains on legacy JSON-RPC client paths and is not a `SuiGrpcClient` method.
> Note: official gRPC docs may still show `getCoins` in some examples. On `@mysten/sui@2.4.0`, use `listCoins` per installed SDK types.

### 4.4 JSON-RPC `options` → Core API `include` Mapping

| 1.x `options` key | 2.x `include` key | Notes |
|-------------------|-------------------|-------|
| `showRawInput: true` | `bcs: true` | Raw BCS bytes |
| `showInput: true` | `transaction: true` | Structured transaction data |
| `showEffects: true` | `effects: true` | Transaction effects |
| `showEvents: true` | `events: true` | Events |
| `showBalanceChanges: true` | `balanceChanges: true` | Balance changes |
| `showObjectChanges: true` | `objectTypes: true` | For changed objects, use `include: { effects: true }` + `effects.changedObjects` |

### 4.5 Transaction Result Shape Changes

Core API returns a discriminated union. Do not assume top-level fields like `res.digest`.
The snippet below is for **core response shape reference** only; for digest-based loading policy, follow section 4.6 (GraphQL-first).

```typescript
const result = await client.getTransaction({
  digest,
  include: { bcs: true, effects: true, transaction: true },
});

const tx =
  result.$kind === 'Transaction' ? result.Transaction : result.FailedTransaction;

const digest = tx.digest;
const bcsBytes = tx.bcs; // Uint8Array | undefined
const changed = tx.effects?.changedObjects ?? [];
```

| 1.x field path | 2.x field path | Notes |
|----------------|----------------|-------|
| `receipt.rawTransaction` | `tx.bcs` | `Uint8Array` (when `include: { bcs: true }`) |
| `receipt.objectChanges` | `tx.effects?.changedObjects` | No top-level `objectChanges` in core result |
| `receipt.effects.created[]` | Filter `tx.effects?.changedObjects` by `idOperation === 'Created'` | Use `outputState` / `outputOwner` as needed |
| `receipt.timestampMs` | `tx.epoch` | Epoch identifier, not millisecond timestamp |

### 4.6 Historical Transaction Loading (Project Policy)

This policy is for loading **already-existing transactions by digest** (for example digest from DB/search/user input).
Treat this separately from immediate post-submit confirmation.
This section defines this guide's migration policy, not a universal protocol mandate.

| Use case | Recommended path |
|----------|------------------|
| Just executed/published in the same flow | gRPC `executeTransaction` + gRPC `waitForTransaction` |
| Any digest-based loading (`getTransaction`-style lookup) | GraphQL-first (project default) |
| Historical lookup (old digest, backfill, archive, analytics) | GraphQL-first (project policy) |
| Unknown age digest | GraphQL-first (project policy) |

- Do not ship gRPC-only digest loaders in this migration policy.
- Keep `grpcClient.getTransaction(...)` as non-default fallback for generic digest loading in this migration policy.
- If age is unknown, classify as historical.

Reason:

- Fullnode data can be pruned over time (older transactions might not be available through gRPC fullnodes).
- GraphQL is indexer-backed and typically offers longer retention for historical transaction retrieval and complex query patterns.
- Retention windows vary by operator, so age-based assumptions must be conservative.

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { SuiGraphQLClient } from '@mysten/sui/graphql';

const grpcClient = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
});

const gqlClient = new SuiGraphQLClient({
  network: 'testnet',
  url: 'https://graphql.testnet.sui.io/graphql',
});

const GET_TX_QUERY = `
  query GetTransaction($digest: String!) {
    transaction(digest: $digest) {
      digest
      transactionBcs
      effects {
        status
      }
    }
  }
`;

// Post-submit confirmation path (not historical loading):
// Prefer gRPC wait in the same execution/publish workflow.
async function executeAndConfirm(txBytes: Uint8Array, signature: string) {
  const exec = await grpcClient.executeTransaction({
    transaction: txBytes,
    signatures: [signature],
  });

  const digest =
    exec.$kind === 'Transaction' ? exec.Transaction.digest : exec.FailedTransaction.digest;

  return grpcClient.waitForTransaction({
    digest,
    include: { effects: true },
  });
}

async function fetchFromGraphQL(digest: string) {
  const r = await gqlClient.query<
    { transaction: { digest: string; transactionBcs: string } | null },
    { digest: string }
  >({
    query: GET_TX_QUERY,
    variables: { digest },
  });
  return { source: 'graphql', tx: r.data?.transaction ?? null };
}

async function loadTransactionByDigest(digest: string) {
  // Default path for digest loading: GraphQL first.
  try {
    return await fetchFromGraphQL(digest);
  } catch {
    // Optional fallback for temporary indexer lag on very fresh transactions.
    // Do not remove GraphQL-first behavior.
    const result = await grpcClient.getTransaction({
      digest,
      include: { bcs: true, effects: true },
    });
    const tx =
      result.$kind === 'Transaction' ? result.Transaction : result.FailedTransaction;
    return { source: 'grpc-fallback', tx };
  }
}
```

> Operational note: if your project parses transaction bytes manually, verify byte shape per source before applying byte-offset logic.
> Some flows return `TransactionData` bytes directly, while others may include signed-envelope bytes.
> `Transaction.from(...)` accepts base64-encoded bytes, so GraphQL `transactionBcs` can usually be passed directly.
> Official `SuiGraphQLClient` docs currently show both:
> - mainnet example: `https://sui-mainnet.mystenlabs.com/graphql`
> - testnet query example: `https://graphql.testnet.sui.io/graphql`
> - Do not assume one fixed hostname pattern for every network. Verify endpoints from current official docs/release notes.
> - Browser apps can hit CORS depending on environment; if needed, route GraphQL through your app/server proxy.
> - Re-validation command example (save output in CI/logs when claiming verification):
>   `curl -sS https://graphql.testnet.sui.io/graphql -H "content-type: application/json" --data '{"query":"{ __type(name:\"Transaction\"){ fields { name } } }"}'`
>   Confirm `transactionBcs` exists.
> - Re-validation command example for effects structure:
>   `curl -sS https://graphql.testnet.sui.io/graphql -H "content-type: application/json" --data '{"query":"{ __type(name:\"TransactionEffects\"){ fields { name } } }"}'`
>   Confirm `objectChanges` exists, then verify `nodes` through your concrete transaction query output.

### 4.7 Single vs Batch Object Reads

- `getObject(...)` is still valid for a single object.
- `getObjects(...)` is the batch API (replacement for legacy `multiGetObjects` patterns).

---

## 5. Frontend Migration (React)

> **Note**: `@mysten/dapp-kit-react` is a **separate package** from `@mysten/dapp-kit`. Having both installed simultaneously may cause conflicts.

### 5.1 dapp-kit.ts — Configuration File

```typescript
import { createDAppKit } from '@mysten/dapp-kit-react';
import { SuiGrpcClient } from '@mysten/sui/grpc';

const GRPC_URLS = {
  testnet: 'https://fullnode.testnet.sui.io:443',
  mainnet: 'https://fullnode.mainnet.sui.io:443',
};

export const dAppKit = createDAppKit({
  networks: ['testnet', 'mainnet'] as const,
  createClient: (network) =>
    new SuiGrpcClient({ network, baseUrl: GRPC_URLS[network] }),
});

// Register for TypeScript autocomplete (recommended)
declare module '@mysten/dapp-kit-react' {
  interface Register {
    dAppKit: typeof dAppKit;
  }
}
```

### 5.2 App.tsx — Provider Replacement

```diff
- import { SuiClientProvider, WalletProvider } from '@mysten/dapp-kit';
+ import { DAppKitProvider } from '@mysten/dapp-kit-react';
+ import { dAppKit } from './dapp-kit';

  function App() {
    return (
-     <SuiClientProvider networks={networkConfig}>
-       <WalletProvider>
-         <YourApp />
-       </WalletProvider>
-     </SuiClientProvider>
+     <DAppKitProvider dAppKit={dAppKit}>
+       <YourApp />
+     </DAppKitProvider>
    );
  }
```

### 5.3 Hook / Sign & Execute Pattern

`@mysten/dapp-kit-react` hook state is backed by `nanostores` (`stores.$wallets`, `stores.$connection`, `stores.$currentClient`, `stores.$currentNetwork`).

```diff
- import { useSignAndExecuteTransaction } from '@mysten/dapp-kit';
+ import { useDAppKit } from '@mysten/dapp-kit-react';

  function MyComponent() {
-   const { mutate: signAndExecute } = useSignAndExecuteTransaction();
+   const dAppKit = useDAppKit();

    async function handleTx() {
-     signAndExecute({ transaction: tx });
+     const result = await dAppKit.signAndExecuteTransaction({ transaction: tx });
    }
  }
```

Official docs show two valid patterns:
- Pattern A (recommended when you want mutation lifecycle state): `useMutation` + `dAppKit.signAndExecuteTransaction(...)`
- Pattern B (alternative): direct `await dAppKit.signAndExecuteTransaction(...)`

If your UI needs mutation lifecycle handling, wrap action calls with `useMutation`:

```typescript
import { useMutation } from '@tanstack/react-query';
import { useDAppKit } from '@mysten/dapp-kit-react';
import { Transaction } from '@mysten/sui/transactions';

function MyComponent() {
  const dAppKit = useDAppKit();
  const { mutateAsync: signAndExecute } = useMutation({
    mutationFn: (tx: Transaction) =>
      dAppKit.signAndExecuteTransaction({ transaction: tx }),
  });

  async function handleTx(tx: Transaction) {
    await signAndExecute(tx);
  }
}
```

If mutation state handling is not needed, direct calls are valid:

```typescript
const result = await dAppKit.signAndExecuteTransaction({ transaction });
```

### 5.4 Common Frontend API Mismatches (Validated)

- There is no `useDisconnectWallet` hook in `@mysten/dapp-kit-react`. Use `await dAppKit.disconnectWallet()` instead.
- `ConnectModal` does not provide a `trigger` prop in `@mysten/dapp-kit-react`.
- Use `ConnectButton` as the default trigger surface.
- Prefer `ConnectModal` `open` prop for direct control in React.
- `show()` / `close()` are Web Component methods on `DAppKitConnectModal`, but are not explicitly exposed as a typed React ref API in `@mysten/dapp-kit-react`; treat ref-method use as runtime-verified only.
- `dAppKit.network` does not exist. Use `dAppKit.networks` for supported networks and `useCurrentNetwork()` for the current network.
- `dAppKit.setNetwork(...)` does not exist. Use `dAppKit.switchNetwork(network)`.
- `dAppKit.client` does not exist. Use `useCurrentClient()` or `dAppKit.getClient()`.
- `dAppKit.signAndExecuteTransaction(...)` returns a discriminated union (`TransactionResult`). Do not assume `res.digest` at top-level.

```typescript
const res = await dAppKit.signAndExecuteTransaction({ transaction: tx });
const digest =
  res.$kind === 'Transaction' ? res.Transaction.digest : res.FailedTransaction.digest;
```

---

## 6. Event / Query Strategy

Do not port legacy polling-based event queries as-is — redesign them.

- `SuiGrpcClient` (2.4.0) does not expose `subscribeEvents`
- gRPC subscription exists at service level (`subscriptionService.subscribeCheckpoints`)
- **Real-time streaming**: check gRPC subscription availability in the release notes before adopting
- **Complex queries / search**: use GraphQL or a dedicated indexer

---

## 7. Infrastructure Checklist

```
✅ gRPC endpoint — port 443, TLS, HTTP/2 required
✅ LB/Nginx — enable grpc_pass or http2
✅ Firewall — allow long-lived connections
✅ ENV variables — update RPC URL to gRPC endpoint
```

| Network | Endpoint |
|---------|---------|
| mainnet | `https://fullnode.mainnet.sui.io:443` |
| testnet | `https://fullnode.testnet.sui.io:443` |
| devnet | `https://fullnode.devnet.sui.io:443` |

---

## 8. Runtime Pitfalls (Result Shapes / Wait APIs)

- `waitForTransaction(...)` uses `include`/`timeout` options (core API style), not JSON-RPC `showObjectChanges`.
- Use `include: { objectTypes: true }` when you need object type information.
- In gRPC/core result shapes, `objectChanges` is not a top-level field. Prefer `effects.changedObjects` first.
- If `effects.changedObjects` is insufficient, add a follow-up digest query and leave a TODO.
- For digest-based follow-up reads, keep GraphQL-first policy. Use `grpcClient.getTransaction(...)` only for explicit fallback or immediate post-submit workflows.

---

## 9. Recommended Rollout Order

```
1. Replace backend code + stabilize compilation
2. Replace frontend Provider / hooks
3. staging E2E (simulate → execute → wallet sign)
4. Phased deployment (partial traffic)
5. Full cutover, then remove all legacy 1.x-only code paths
```

---

## 10. References

- [Data Access / sunset notice](https://docs.sui.io/concepts/data-access/data-serving)
- [Sui Core client API](https://sdk.mystenlabs.com/sui/clients/core)
- [Sui gRPC client API](https://sdk.mystenlabs.com/sui/clients/grpc)
- [Sui GraphQL client API](https://sdk.mystenlabs.com/sui/clients/graphql)
- [Sui Clients overview (compatibility notes)](https://sdk.mystenlabs.com/sui/clients)
- [dApp Kit migration](https://sdk.mystenlabs.com/sui/migrations/sui-2.0/dapp-kit)
- [dApp Kit React hooks](https://sdk.mystenlabs.com/dapp-kit/react/hooks/use-dapp-kit)
- [Sign and Execute Transaction (dApp Kit action)](https://sdk.mystenlabs.com/dapp-kit/actions/sign-and-execute-transaction)
- [dApp Kit React Getting Started (direct sign-and-execute example)](https://sdk.mystenlabs.com/dapp-kit/getting-started/react)
- [Fullnode gRPC reference](https://docs.sui.io/references/fullnode-protocol)

Local package verification sources (for your pinned version):
- `node_modules/@mysten/sui/package.json`
- `node_modules/@mysten/sui/dist/grpc/client.d.mts`
- `node_modules/@mysten/sui/dist/client/types.d.mts`
- `node_modules/@mysten/sui/dist/graphql/client.d.mts`
- `node_modules/@mysten/sui/dist/jsonRpc/client.d.mts`
- `node_modules/@mysten/dapp-kit-react/dist/index.d.mts` (fallback: `dist/index.mjs`)
- `node_modules/@mysten/dapp-kit-core/dist/types-*.d.mts`
