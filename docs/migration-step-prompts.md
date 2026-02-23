# Migration Step Prompts

Use these prompts with the execution order selected in `docs/migration-run-order.md`.
Run policy checks from `docs/policy-gates.md` after each migration phase.

Core steps:
- Step 1, Step 2, Step 3, Step 4, Step 5, Step 6

Conditional steps:
- Optional Step A, Optional Step B, Optional Step C, Optional Step D (only for Case B/C)

Monorepo note:
- Use repository root `.` as scan scope (not `src`-only assumptions).
- For `rg`, use include/exclude globs so commands work in both single-package and monorepo layouts.

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
Scan all workspace package source directories (for example `src`, `app`, `server`, `lib`) and report the actual scanned roots first.
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
- For shared digest loader helpers: do not hardcode network at module scope; receive network from caller context
- For chain context selection: if the flow is chain-fixed, keep explicit network constant with a short rationale comment; otherwise accept network from caller context
- Do not use deployment-only config values as implicit chain-verification network without explicit rationale
- In GraphQL fallback, check `response.errors` before reading `response.data`
- In GraphQL queries, treat `effects.objectChanges` / `effects.balanceChanges` as connection fields and access item fields via `nodes` or `edges`
- Include network and failed stage (`grpc`/`graphql`) in thrown loader errors
- If GraphQL query fields or multi-source byte parsing are changed, add `TODO[Gate-F]` with affected files and runtime validation plan
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
- Digest loader does not use module-scope hardcoded network literals in shared helpers
- Network source rationale is explicit (chain-fixed constant with comment, or caller-provided network)
- GraphQL fallback checks `response.errors` before `response.data`
- GraphQL `effects` connection fields use `nodes`/`edges` access (for example `objectChanges.nodes`)
- Loader errors include network and failed stage
- Gate F applicability is explicit (`required` or `N/A`) for GraphQL/loader changes
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
- Run React Query usage gate:
  1) `rg "from ['\"]@tanstack/react-query['\"]|require\\(['\"]@tanstack/react-query['\"]\\)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
  2) if no matches, uninstall `@tanstack/react-query`
  3) if matches exist, run hook-level triage:
     - `rg "\\buse(Query|Mutation|InfiniteQuery|Queries|SuspenseQuery|SuspenseInfiniteQuery|QueryClient)\\b" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
     - if hook usage exists, keep dependency and report matched files with hook names
     - if hook usage does not exist, check wrapper-only pattern:
       `rg "QueryClientProvider|new QueryClient\\(|<QueryClientProvider" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
       then check package source:
       `rg "\"@mysten/dapp-kit\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"`
       `rg "\"@mysten/dapp-kit-react\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"`
       if wrapper-only + old dapp-kit removed + dapp-kit-react present:
       remove vestigial wrapper code, rerun step 1, then uninstall `@tanstack/react-query` if clean
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
- React Query usage gate is applied and result is explicit:
  - no direct imports -> dependency removed
  - hook usage exists -> dependency kept with file list and hook names
  - provider/wrapper-only with removed old dapp-kit -> wrapper removed, then dependency removed
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
- In monorepos, run scripts per workspace package that declares the script.
- If a script does not exist, skip it and note it.
- Report errors with file path + line number + likely root cause.
- Apply minimal fixes and rerun until commands pass.
- If blockers remain, mark overall status as `FAIL` (do not mark migration complete).

Output:
1. commands executed
2. pass/fail status per command
3. fixed issues summary
4. remaining blockers (if any)
```

Also run:

```bash
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -P "new\\s+SuiGrpcClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -P "new\\s+SuiGraphQLClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -U -P "objectChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -U -P "balanceChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"
rg -n "transactionBcs|transaction\\s*\\(digest:|getTransaction\\(|SuiGraphQLClient|query\\s*:\\s*`" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"
rg "from ['\"]@tanstack/react-query['\"]|require\\(['\"]@tanstack/react-query['\"]\\)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg "\"@tanstack/react-query\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"
```

Verify:
- typecheck/build/test statuses are reported
- monorepo runs include package/workspace path in the status report
- blockers (if any) are concrete, reproducible, and force overall `FAIL` status
- forbidden JSON-RPC check is clean
- Gate D/E findings are resolved (fixed) or documented as false-positive with file-level evidence
- React Query gate result is reported (removed if unused, kept with usage evidence if used)
- Gate F status is reported:
  - `N/A` with reason when no GraphQL query/loader changes exist
  - `PASS` only with runtime evidence (schema introspection + query smoke + fallback-path execution)

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

Policy gate reminder:
- Run mandatory checks from `docs/policy-gates.md` and include pass/fail in this step report.

## Step 6 - Final Independent Audit (run as separate prompt)

Use this as a separate second prompt after the main migration flow completes.

Paste into AI agent:

```text
Run a final independent migration audit in read-only mode first.
Apply minimal fixes only if needed, then rerun the audit.

Audit scope:
- migrated source files
- migration-related docs/rules in this project

Checks:
1. Workflow/policy consistency
- no conflict between README, run-order, step-prompts, rules, and policy gates
- one-shot copy/paste flow remains intact

2. Mandatory pattern gates
- `rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg -n -P "new\\s+SuiGrpcClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg -n -P "new\\s+SuiGraphQLClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg -n -U -P "objectChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg -n -U -P "balanceChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg "from ['\"]@tanstack/react-query['\"]|require\\(['\"]@tanstack/react-query['\"]\\)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg "\"@tanstack/react-query\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"`

3. Transaction loading contract check
- digest loading uses two-stage path (gRPC getTransaction -> GraphQL fallback)
- explicit error exists when both sources miss
- shared digest loaders avoid module-scope fixed network literals
- network source semantics are explicit (chain-fixed rationale or caller-provided network)
- GraphQL path checks `response.errors` before `response.data`
- GraphQL connection fields under effects use `nodes`/`edges` access
- loader errors include network and failed stage

4. Gate F runtime validation (conditional)
- if GraphQL query/loader changes exist, verify runtime evidence for:
  - schema introspection for used fields
  - query smoke check on actual endpoint
  - fallback-path execution proof
- use commands from `docs/policy-gates.md` section "Gate F Runtime Validation (Conditional)"
- if no relevant changes exist, mark `N/A` with explicit reason

Output:
1. PASS/FAIL summary
2. findings by severity with file:line
3. minimal patch diff (if any)
4. rerun result after patch
```

Verify:
- audit output includes concrete evidence and file paths
- all mandatory gates are accounted for
- unresolved findings are tracked as blockers and force audit `FAIL`

Step TODO:
- [ ] Step 6 prompt executed
- [ ] audit report captured
- [ ] post-fix rerun status captured

Step report:
1. step name + completion status
2. commands executed
3. PASS/FAIL summary
4. findings (file:line, severity)
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
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
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
- `rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`
- `rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"`

Rules:
- Use the package manager already used by the target project (npm/pnpm/yarn).
- In monorepos, run scripts per workspace package that declares the script.
- If a script does not exist, skip it and note it.
- Report errors with file path + line number + likely root cause.

Output:
1. commands executed
2. pass/fail status per command
3. remaining blockers (if any)
```

Verify:
- typecheck/build/test status is reported
- monorepo runs include package/workspace path in the status report
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
