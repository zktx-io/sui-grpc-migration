# Migration Step Prompts

Use these prompts with the execution order selected in `docs/migration-run-order.md`.

## Step 1 - Copy migration docs/rules

Paste into AI agent:

```text
Copy these two items:

1. sui-grpc-migration/docs/sui-grpc-migration.md
   -> {your project root}/docs/sui-grpc-migration.md

2. sui-grpc-migration/agents/sui-grpc-rules.md contents
   -> append to {your project root}/AGENTS.md (create if missing)
   -> if using Cursor, also copy to .cursor/rules/sui-grpc.md

Report the actual paths created when done.
```

Verify:
- `docs/sui-grpc-migration.md` exists in target project
- `AGENTS.md` or `.cursor/rules/sui-grpc.md` contains Sui migration rules

Step TODO:
- [ ] Step 1 prompt executed
- [ ] copy paths confirmed
- [ ] verify checks passed

Step report:
1. step name + completion status
2. actual paths created/updated
3. verify checklist pass/fail
4. blockers/TODOs
5. ready for next step: yes/no

## Step 2 - Scan migration targets

Paste into AI agent:

```text
Read docs/sui-grpc-migration.md and find all files in this project
that need @mysten/sui 1.x -> 2.x migration. Do not modify anything.
Report only in this format:

| File | Line | Pattern | Backend / Frontend |
```

Verify:
- Full source directories were scanned
- Backend and frontend are classified correctly
- Candidate file list is plausible

Step TODO:
- [ ] Step 2 prompt executed
- [ ] scan output saved
- [ ] backend/frontend classification reviewed

Step report:
1. step name + completion status
2. scanned directories
3. detected file count (backend/frontend)
4. suspicious misses/assumptions
5. ready for next step: yes/no

## Step 3 - Migrate backend

Paste into AI agent:

```text
Read docs/sui-grpc-migration.md.
Migrate all backend files detected in Step 2 based on section 4.
- Never introduce `SuiJsonRpcClient` or imports from `@mysten/sui/jsonRpc`
- If a path seems JSON-RPC-only, stop and mark it as `BLOCKER` + TODO (do not add fallback client)
- For transaction loading code: use two-stage digest loading (`gRPC getTransaction` first, then GraphQL fallback; no single-source loaders)
- For post-deploy/post-execute confirmation in the same flow, use gRPC `waitForTransaction`
- Show output as a diff
- Do not guess at changed response type fields; mark unknown fields with TODO comments
Output:
1. migrated backend file list
2. per-file diff
3. TODO list for manual type checks
```

Verify:
- `executeTransaction(...)` call shape is correct (`transaction`, `signatures[]`)
- Legacy `devInspect` / `dryRun` patterns were migrated
- `commandResults` consumers include `include: { commandResults: true }`
- No forbidden JSON-RPC imports remain in backend outputs
- Digest transaction loading uses two-stage fallback (`gRPC -> GraphQL`) and explicit error when both miss
- gRPC `waitForTransaction` is used for same-flow confirmation

Step TODO:
- [ ] Step 3 prompt executed
- [ ] backend diff reviewed
- [ ] no forbidden JSON-RPC imports
- [ ] manual TODO list tracked

Step report:
1. step name + completion status
2. migrated backend file list
3. key diff summary
4. verify checklist pass/fail
5. TODO/BLOCKER list

## Step 4 - Migrate frontend

Paste into AI agent:

```text
Read docs/sui-grpc-migration.md.
Migrate all frontend files detected in Step 2 based on section 5.
- Show output as a diff
- Replace all @mysten/dapp-kit imports with @mysten/dapp-kit-react
- Never introduce `SuiJsonRpcClient` or imports from `@mysten/sui/jsonRpc`
Output:
1. migrated frontend file list
2. per-file diff
3. remaining manual checks
```

Verify:
- `SuiClientProvider`/`WalletProvider` -> `DAppKitProvider`
- `useSuiClient` -> `useDAppKit()`
- `useSignAndExecuteTransaction` -> `dAppKit.signAndExecuteTransaction`
- Legacy `@mysten/dapp-kit` imports are removed
- `@tanstack/react-query` is removed only when app code does not use it directly
- No forbidden JSON-RPC imports remain in frontend outputs

Step TODO:
- [ ] Step 4 prompt executed
- [ ] frontend diff reviewed
- [ ] provider/hook migration confirmed
- [ ] no forbidden JSON-RPC imports

Step report:
1. step name + completion status
2. migrated frontend file list
3. key diff summary
4. verify checklist pass/fail
5. TODO/BLOCKER list

## Step 5 - Validation

Paste into AI agent:

```text
Run post-migration validation from the project root.

1. Run `typecheck` script if it exists.
2. Run `build` script if it exists.
3. Run `test` script if it exists.

Rules:
- Use the package manager already used by the target project (npm/pnpm/yarn).
- If a script does not exist, skip it and note it.
- Report errors with file path + line number + likely root cause.
- Apply minimal fixes and rerun until commands pass or a clear blocker remains.

Output:
1. commands executed
2. pass/fail status per command
3. fixed issues summary
4. remaining blockers (if any)
```

Also run:

```bash
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc|getJsonRpcFullnodeUrl" src
```

Verify:
- typecheck/build/test statuses are reported
- blockers (if any) are concrete and reproducible
- forbidden JSON-RPC check is clean

Step TODO:
- [ ] Step 5 prompt executed
- [ ] validation results captured
- [ ] blockers documented with reproduction

Step report:
1. step name + completion status
2. commands executed
3. pass/fail status per command
4. fixed issues summary
5. remaining blockers + reproduction

## Optional Step A - Copy Walrus/SuiNS guide

Paste into AI agent:

```text
Copy this item:

1. sui-grpc-migration/docs/walrus-suins-migration.md
   -> {your project root}/docs/walrus-suins-migration.md

Report the actual path created when done.
```

Verify:
- `docs/walrus-suins-migration.md` exists in target project

Step TODO (optional):
- [ ] Optional Step A prompt executed
- [ ] guide copy path confirmed

Step report:
1. step name + completion status
2. actual paths created/updated
3. verify checklist pass/fail
4. blockers/TODOs
5. ready for next step: yes/no

## Optional Step B - Scan Walrus/SuiNS targets

Paste into AI agent:

```text
Read docs/walrus-suins-migration.md and find all files in this project
that need Walrus/SuiNS migration (0.x -> 1.x). Do not modify anything.
Report only in this format:

| File | Line | Pattern |
```

Verify:
- Files using `new WalrusClient(...)`, `new SuinsClient(...)`, or `@mysten/sui/client` are included
- Actual source directories are covered

Step TODO (optional):
- [ ] Optional Step B prompt executed
- [ ] scan output saved
- [ ] candidate list reviewed

Step report:
1. step name + completion status
2. scanned directories
3. detected file count
4. suspicious misses/assumptions
5. ready for next step: yes/no

## Optional Step C - Migrate Walrus/SuiNS files

Paste into AI agent:

```text
Read docs/walrus-suins-migration.md.
Migrate all files detected in Optional Step B.
- Use `SuiGrpcClient` + `$extend(walrus())` / `$extend(suins())` patterns
- Never introduce `SuiJsonRpcClient` or imports from `@mysten/sui/jsonRpc`
- If a path seems JSON-RPC-only, stop and mark it as `BLOCKER` + TODO
- Show output as a diff
Output:
1. migrated file list
2. per-file diff
3. TODO/BLOCKER list
```

Run integration gates:

```bash
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc|getJsonRpcFullnodeUrl" src
rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" src
```

Verify:
- No forbidden JSON-RPC imports introduced
- Legacy constructors/imports removed where migrated
- Integration gates are clean or blockers logged

Step TODO (optional):
- [ ] Optional Step C prompt executed
- [ ] diff reviewed
- [ ] integration gates checked

Step report:
1. step name + completion status
2. migrated file list
3. key diff summary
4. integration gate results
5. TODO/BLOCKER list

## Optional Step D - Validate Walrus/SuiNS migration

Paste into AI agent:

```text
Run post-migration validation from the project root.

1. Run `typecheck` script if it exists.
2. Run `build` script if it exists.
3. Run `test` script if it exists.

Also run:
- `rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" src`
- `rg "SuiJsonRpcClient|@mysten/sui/jsonRpc|getJsonRpcFullnodeUrl" src`

Rules:
- Use the package manager already used by the target project (npm/pnpm/yarn).
- If a script does not exist, skip it and note it.
- Report errors with file path + line number + likely root cause.

Output:
1. commands executed
2. pass/fail status per command
3. remaining blockers (if any)
```

Verify:
- typecheck/build/test status is reported
- legacy Walrus/SuiNS patterns are removed
- JSON-RPC disallow check is clean

Step TODO (optional):
- [ ] Optional Step D prompt executed
- [ ] validation results captured
- [ ] cleanup status confirmed

Step report:
1. step name + completion status
2. commands executed
3. pass/fail status per command
4. legacy pattern check results
5. remaining blockers + reproduction
