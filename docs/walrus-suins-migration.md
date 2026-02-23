# Walrus & SuiNS Migration Guide (0.x -> 1.x)

> **Validated date**: 2026-02-23  
> **Validated target**: `@mysten/walrus@1.0.3`, `@mysten/suins@1.0.2`, `@mysten/sui@2.4.0`
> **Migration policy**: migrated code must not use `SuiJsonRpcClient` or imports from `@mysten/sui/jsonRpc`

This guide is a verified migration reference for projects moving:
- `@mysten/walrus` `0.x` -> `1.x`
- `@mysten/suins` `0.x` -> `1.x`
- together with `@mysten/sui` `1.x` -> `2.x`

## 0. Verification Baseline

Commands used during verification:

```bash
# Latest + peer dependency check
npm view @mysten/walrus version peerDependencies --json
npm view @mysten/suins version peerDependencies --json

# 0.x existence check
npm view @mysten/walrus versions --json
npm view @mysten/suins versions --json

# Runtime extension smoke checks
node -e 'import { SuiGrpcClient } from "@mysten/sui/grpc"; import { walrus } from "@mysten/walrus"; import { suins } from "@mysten/suins"; const c = new SuiGrpcClient({ network: "testnet", baseUrl: "https://fullnode.testnet.sui.io:443" }); console.log(!!c.$extend(walrus()), !!c.$extend(suins()));'
```

Verified facts:
- `@mysten/walrus` latest: `1.0.3`, peer dependency: `@mysten/sui@^2.3.2`
- `@mysten/suins` latest: `1.0.2`, peer dependency: `@mysten/sui@^2.3.2`
- Both packages have real `0.x` release lines and `1.x` release lines.

## 1. What Changed (0.x vs 1.x)

### 1.1 Dependency Model

`0.x` (validated on late 0.x releases `@mysten/walrus@0.9.0`, `@mysten/suins@0.9.13`):
- Walrus and SuiNS carried `@mysten/sui` `1.45.2` in `dependencies`.

`1.x`:
- Walrus and SuiNS moved `@mysten/sui` to `peerDependencies` (`^2.3.2`).
- Result: your app owns the single Sui SDK version (`2.x`) instead of pulling legacy `1.x` transitively.

### 1.2 Walrus Changes

Important correction:
- The `walrus()` extension pattern already existed in late `0.x` (for example `0.9.0`).
- `1.x` did not introduce `$extend` itself; it aligned with Sui `2.x` and now documents gRPC-first setup.

Practical migration differences:
- `0.x` code commonly relied on legacy Sui client paths.
- `1.0.x` docs show `SuiGrpcClient`-based setup.
- `WalrusClientConfig` in `0.9.x` allowed `suiRpcUrl` fallback; `1.0.x` requires `suiClient: ClientWithCoreApi`.
- `systemObject()` in `1.0.3` returns an object shape:
  - `{ id: string; version: string; package_id: string; new_package_id: string | null }`
  - Access the object ID via `const systemObj = await client.walrus.systemObject(); const id = systemObj.id;`

### 1.3 SuiNS Changes

Real API shift:
- `0.9.x`: class-first API (`new SuinsClient({ client: SuiClient, network })`)
- `1.0.x`: extension helper added (`suins()`), and client type switched to `ClientWithCoreApi`

Internal read path change:
- `0.9.x`: `client.getDynamicFieldObject(...)`
- `1.0.x`: `client.core.getDynamicField(...)` + BCS-backed parsing

## 2. Migration Patterns

### 2.1 Walrus

Before (0.x-style legacy client setup):

```typescript
import { SuiClient, getFullnodeUrl } from '@mysten/sui/client';
import { WalrusClient } from '@mysten/walrus';

const suiClient = new SuiClient({ url: getFullnodeUrl('testnet') });
const walrusClient = new WalrusClient({ network: 'testnet', suiClient });

const epoch = (await walrusClient.systemState()).committee.epoch;
```

After (1.x + Sui 2.x, gRPC-first):

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { walrus } from '@mysten/walrus';

const client = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
}).$extend(walrus());

const epoch = (await client.walrus.systemState()).committee.epoch;
const systemObj = await client.walrus.systemObject(); // returns object with string `id` field
```

### 2.2 SuiNS

Before (0.x):

```typescript
import { SuiClient } from '@mysten/sui/client';
import { SuinsClient } from '@mysten/suins';

const suiClient = {} as SuiClient; // your app's SuiClient in 1.x era
const suinsClient = new SuinsClient({ client: suiClient, network: 'mainnet' });

const record = await suinsClient.getNameRecord('example.sui');
```

After (1.x + Sui 2.x):

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { suins } from '@mysten/suins';

const client = new SuiGrpcClient({
  network: 'mainnet',
  baseUrl: 'https://fullnode.mainnet.sui.io:443',
}).$extend(suins());

const record = await client.suins.getNameRecord('example.sui');
```

## 3. JSON-RPC Policy

For Walrus/SuiNS migration output:
- Do not introduce `SuiJsonRpcClient`.
- Do not import from `@mysten/sui/jsonRpc`.
- If a path appears JSON-RPC-only, leave a `BLOCKER` + TODO for manual redesign.

## 4. Common Build Errors and Fixes

| Error | Root cause | Minimal fix |
|---|---|---|
| `The requested module '@mysten/sui/client' does not provide an export named 'SuiClient'` | `SuiClient` class is not exported from `@mysten/sui/client` in Sui 2.x | Migrate code to gRPC/core clients (`SuiGrpcClient`, `SuiGraphQLClient`) |
| `... does not provide an export named 'getFullnodeUrl'` | Old helper names no longer exported from modern paths | Replace with explicit gRPC `baseUrl` constants (`https://fullnode.<network>.sui.io:443`) |
| `Package subpath './experimental' is not defined by "exports"` | Legacy Walrus 0.x imported from `@mysten/sui/experimental` | Upgrade Walrus to `1.x` |
| Type imports like `DynamicFieldInfo`, `MoveValue`, `SuiParsedData` fail from `@mysten/sui/client` | Those are JSON-RPC types, not client-core exports | Migrate away from JSON-RPC-specific types; if still unresolved, keep as `BLOCKER` |

## 5. Recommended Rollout

```text
1. Upgrade @mysten/sui to 2.x first.
2. Upgrade @mysten/walrus and @mysten/suins to 1.x.
3. Replace direct legacy client imports/constructors.
4. Switch call sites to client.$extend(walrus()) / client.$extend(suins()).
5. Run `rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" src` and ensure no matches remain.
6. Run typecheck/build/test and fix remaining blockers.
```

## 6. References

- [Sui Clients overview (compatibility note includes Walrus and SuiNS)](https://sdk.mystenlabs.com/sui/clients)
- [Sui gRPC client docs](https://sdk.mystenlabs.com/sui/clients/grpc)
- [Walrus SDK docs](https://sdk.mystenlabs.com/walrus)
- [SuiNS docs portal](https://docs.suins.io)
- [npm: @mysten/walrus](https://www.npmjs.com/package/@mysten/walrus)
- [npm: @mysten/suins](https://www.npmjs.com/package/@mysten/suins)

Local verification file targets (Sui 2.x + package 1.x):
- `node_modules/@mysten/walrus/dist/client.d.mts`
- `node_modules/@mysten/walrus/dist/types.d.mts`
- `node_modules/@mysten/suins/dist/suins-client.d.mts`
- `node_modules/@mysten/suins/dist/types.d.mts`
- `node_modules/@mysten/sui/dist/client/client.d.mts`
- `node_modules/@mysten/sui/dist/grpc/client.d.mts`
- `node_modules/@mysten/sui/dist/jsonRpc/index.d.mts` (error-origin verification only, not migration target)
