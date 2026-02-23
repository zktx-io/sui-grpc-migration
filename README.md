# sui-grpc-migration

A migration kit for upgrading your Sui SDK project from `@mysten/sui` 1.x to 2.x, designed to be used step-by-step with an AI agent.

> **Scope**: `@mysten/sui` 1.x → 2.x (assumes `@mysten/sui.js` migration is already completed)  
> **Target**: Projects using `@mysten/sui` + legacy `@mysten/dapp-kit`  
> **Target versions**: `@mysten/sui@^2.4.0` · `@mysten/dapp-kit-react@^1.0.2` · `@mysten/dapp-kit-core@^1.0.4`
> **Runtime note**: `@mysten/sui@2.4.0` requires Node `>=22`

---

## Contents

```
sui-grpc-migration/
├── README.md                  ← This file. Full usage guide
├── docs/
│   └── sui-grpc-migration.md  ← Migration reference with API mappings and code examples
└── agents/
    └── sui-grpc-rules.md      ← Rules snippet to inject into your AI agent
```

---

## How to Use

Clone this repo or download the files, then paste each step into your AI agent in order.

```bash
git clone https://github.com/zktx-io/sui-grpc-migration.git
```

---

### ✅ Step 1 — Copy files into your project

The migration reference and AI rules must be in your project before the agent can work with them.

Paste the following into your AI agent:

```
Copy these two items:

1. sui-grpc-migration/docs/sui-grpc-migration.md
   → {your project root}/docs/sui-grpc-migration.md

2. sui-grpc-migration/agents/sui-grpc-rules.md contents
   → append to {your project root}/AGENTS.md (create if missing)
   → if using Cursor, also copy to .cursor/rules/sui-grpc.md

Report the actual paths created when done.
```

**Verify:**
- `docs/sui-grpc-migration.md` exists in your project
- `AGENTS.md` or `.cursor/rules/sui-grpc.md` contains the Sui rules

---

### ✅ Step 2 — Scan for migration targets

Identify affected files before making any changes. Missing items in the scan will cause incomplete migrations.

Paste the following into your AI agent:

```
Read docs/sui-grpc-migration.md and find all files in this project
that need @mysten/sui 1.x -> 2.x migration. Do not modify anything.
Report only in this format:

| File | Line | Pattern | Backend / Frontend |
```

**Verify:**
- Full `src/` directory was scanned
- Backend (Node.js) and frontend (React) files are distinguished
- The list of files makes sense for the scope of your project

---

### ✅ Step 3 — Migrate backend files

Process all backend files detected in Step 2 in one batch.

Paste the following into your AI agent:

```
Read docs/sui-grpc-migration.md.
Migrate all backend files detected in Step 2 based on section 4.
- Show output as a diff
- Do not guess at changed response type fields — mark them with a TODO comment instead
Output:
1. migrated backend file list
2. per-file diff
3. TODO list for manual type checks
```

**Verify:**
- `SuiGrpcClient` usage is valid for your target path
- `executeTransaction(...)` call shape matches section 4 guidance
- `devInspect` / `dryRun` patterns are replaced with `simulateTransaction` where used
- Code that used returnValues includes `include: { commandResults: true }`
- TODO comments are reviewed against `SuiClientTypes` type definitions manually

---

### ✅ Step 4 — Migrate frontend files

Process all frontend files detected in Step 2 in one batch.

Paste the following into your AI agent:

```
Read docs/sui-grpc-migration.md.
Migrate all frontend files detected in Step 2 based on section 5.
- Show output as a diff
- Replace all @mysten/dapp-kit imports with @mysten/dapp-kit-react
Output:
1. migrated frontend file list
2. per-file diff
3. remaining manual checks
```

**Verify:**
- `SuiClientProvider` + `WalletProvider` → `DAppKitProvider`
- `useSuiClient` → `useDAppKit()`
- `useSignAndExecuteTransaction` → `dAppKit.signAndExecuteTransaction`
- All `@mysten/dapp-kit` imports are removed
- `@tanstack/react-query` is removed only if your app no longer imports it directly
- Keep `@tanstack/react-query` when you use `useMutation` / `useQuery` for UI state management
- `@mysten/dapp-kit-react` state hooks are nanostores-based; React Query is optional, not required
- If migration errors mention missing hooks/props/fields, check docs section `5.4` and section `8`

---

### ✅ Step 5 — Build and Error Check

Run validation commands after migration and confirm there are no compile/runtime blockers.

Paste the following into your AI agent:

```
Run post-migration validation from the project root.

1. Run `typecheck` script if it exists.
2. Run `build` script if it exists.
3. Run `test` script if it exists.

Rules:
- Use the package manager already used by this repo (npm/pnpm/yarn).
- If a script does not exist, skip it and note it.
- Report errors with file path + line number + likely root cause.
- Apply minimal fixes and rerun until commands pass or a clear blocker remains.

Output:
1. commands executed
2. pass/fail status per command
3. fixed issues summary
4. remaining blockers (if any)
```

**Verify:**
- No TypeScript compile errors remain
- Build completes successfully
- Test command status is reported (pass/fail/skipped if not defined)
- Remaining blockers are concrete and reproducible

---

## AI Tool — Rules File Location

The rules file location depends on your AI tool.

| AI Tool | Rules file path |
|---------|----------------|
| Cursor | `.cursor/rules/sui-grpc.md` |
| Claude Code | `CLAUDE.md` |
| Codex / Gemini / general | `AGENTS.md` |

---

## Validation Policy

Use these rules when generated migration code and docs do not match perfectly:

1. Official docs define the intended migration direction.
2. Installed SDK package types/source define the executable truth for your pinned version.
3. If they conflict, follow installed SDK definitions first and leave a TODO or note.

Validation snapshot should always include:
- package versions (for example `@mysten/sui@2.4.0`)
- Node version (must satisfy package `engines`)
- verification date

Recommended local verification files:
- `node_modules/@mysten/sui/package.json`
- `node_modules/@mysten/sui/dist/grpc/client.d.mts`
- `node_modules/@mysten/sui/dist/client/types.d.mts`
- `node_modules/@mysten/sui/dist/graphql/client.d.mts`
- `node_modules/@mysten/sui/dist/jsonRpc/client.d.mts`
- `node_modules/@mysten/dapp-kit-react/dist/index.d.mts` (fallback: `dist/index.mjs`)
- `node_modules/@mysten/dapp-kit-core/dist/types-*.d.mts`

---

## References

- [Data Access / sunset notice](https://docs.sui.io/concepts/data-access/data-serving)
- [Sui Core client API](https://sdk.mystenlabs.com/sui/clients/core)
- [Sui gRPC client API](https://sdk.mystenlabs.com/sui/clients/grpc)
- [Sui GraphQL client API](https://sdk.mystenlabs.com/sui/clients/graphql)
- [dApp Kit migration](https://sdk.mystenlabs.com/sui/migrations/sui-2.0/dapp-kit)
- [dApp Kit React hooks](https://sdk.mystenlabs.com/dapp-kit/react/hooks/use-dapp-kit)
- [Sign and Execute Transaction (dApp Kit action)](https://sdk.mystenlabs.com/dapp-kit/actions/sign-and-execute-transaction)
- [dApp Kit React Getting Started (direct sign-and-execute example)](https://sdk.mystenlabs.com/dapp-kit/getting-started/react)
