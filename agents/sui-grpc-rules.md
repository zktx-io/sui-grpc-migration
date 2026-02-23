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
- If both gRPC and GraphQL miss, return explicit `pruned/not found` error (no silent failure)
- Client: `import { SuiGrpcClient } from '@mysten/sui/grpc'`
- Simulation: `client.simulateTransaction({ transaction })`
- If replacing devInspect: `client.simulateTransaction({ transaction, include: { commandResults: true } })`
- Execution: `client.executeTransaction({ transaction: txBytes, signatures: [sig] })`
- React Provider: `DAppKitProvider` from `@mysten/dapp-kit-react`
- Wallet hook: `useDAppKit()` from `@mysten/dapp-kit-react`
- `@tanstack/react-query` is optional (use it only for mutation/query UI state wrappers)
- Events / complex queries: use GraphQL or an indexer
- Full migration reference: `docs/sui-grpc-migration.md`
- If official docs and installed SDK API differ, follow installed SDK definitions in `node_modules/@mysten/*/dist/*.d.mts`
