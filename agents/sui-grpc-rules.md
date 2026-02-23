## Sui SDK Rules

This project uses the gRPC client from `@mysten/sui/grpc`.
Assume migration scope is `@mysten/sui` 1.x -> 2.x only.
Do not reason from `@mysten/sui.js` legacy names like `executeTransactionBlock`.

- Do NOT create new JSON-RPC `SuiClient` instances
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
