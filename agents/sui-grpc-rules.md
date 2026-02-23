## Sui SDK Rules

This project uses the gRPC client from `@mysten/sui/grpc`.
Assume migration scope is `@mysten/sui` 1.x -> 2.x only.
Do not reason from `@mysten/sui.js` legacy names like `executeTransactionBlock`.

- Do NOT import from `@mysten/sui/jsonRpc` in migrated code
- Do NOT create or keep `SuiJsonRpcClient` (or legacy `SuiClient`) instances
- If a path appears JSON-RPC-only, do NOT implement it with JSON-RPC fallback. Leave a `BLOCKER` + TODO for manual redesign.
- Transaction loading policy:
- Immediate post-submit confirmation (same execute/publish flow): use gRPC `waitForTransaction`
- Any `getTransaction`-style digest loading: use two-stage loading (gRPC first, GraphQL fallback)
- Historical/unknown-age digest loading: do not use single-source loaders
- Shared digest loaders: keep network caller-driven (no module-scope fixed network literal)
- Chain-fixed flows may hardcode network only with an explicit rationale comment
- Do not use deployment-target config as implicit chain-verification context unless explicitly intended
- GraphQL fallback must check `response.errors` before consuming `response.data`
- Loader errors must include network and failed stage (`grpc` or `graphql`)
- In GraphQL queries, treat `effects.objectChanges` / `effects.balanceChanges` as connection fields and access items via `nodes`/`edges`
- Do not assume `query<Result>` generic types prove schema validity; validate GraphQL fields with introspection when changed
- If both gRPC and GraphQL miss, return explicit `pruned/not found` error (no silent failure)
- If GraphQL query fields or two-stage byte loader behavior changes, run Gate F runtime validation (schema introspection, query smoke check, and fallback-path execution evidence)
- Unresolved blocker/gate findings are fail state (do not mark migration complete).
- Client: `import { SuiGrpcClient } from '@mysten/sui/grpc'`
- Simulation: `client.simulateTransaction({ transaction })`
- If replacing devInspect: `client.simulateTransaction({ transaction, include: { commandResults: true } })`
- Execution: `client.executeTransaction({ transaction: txBytes, signatures: [sig] })`
- React Provider: `DAppKitProvider` from `@mysten/dapp-kit-react`
- Wallet hook: `useDAppKit()` from `@mysten/dapp-kit-react`
- `@tanstack/react-query` usage gate:
- if app code has no direct imports from `@tanstack/react-query`, remove the dependency
- if app code has direct imports, run hook-level triage:
- hook usage exists (`useQuery`, `useMutation`, `useInfiniteQuery`, `useQueryClient`, etc.) -> keep and report usage files
- if only `QueryClient`/`QueryClientProvider` wrappers remain and old `@mysten/dapp-kit` is removed while `@mysten/dapp-kit-react` is present -> treat as vestigial wrapper, remove wrapper, rerun scan, then remove dependency if clean
- do not decide keep/remove from import presence alone
- Events / complex queries: use GraphQL or an indexer
- Full migration reference: `docs/sui-grpc-migration.md`
- If official docs and installed SDK API differ, follow installed SDK definitions in `node_modules/@mysten/*/dist/*.d.mts`
