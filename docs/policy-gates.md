# Migration Policy Gates (Standardized Checks)

Use this file to enforce migration policy with repeatable checks.
The goal is to move from doc-only guidance to machine-verifiable gates.
This does not change the user workflow: gates are run inside the same one-shot agent prompt flow.

## 1) Gate Philosophy

- Migration prompts are guidance.
- Policy gates are enforcement.
- A migration is complete only when all required gates pass.
- Unresolved blockers are a fail state (no accepted-blocker shortcut).

## 2) Gate Sets (Core vs Conditional)

Run from project root after each migration phase and in CI.
Keyword scope note:
- Legacy-detection gates should use 1.x-era migration signals (avoid 2.x-only legacy tokens).
- Post-migration quality gates may inspect 2.x loader patterns (for example `SuiGrpcClient`/`SuiGraphQLClient` usage).

Core gates (always apply): Gate A, Gate C, Gate D, Gate E.
Conditional gates:
- Gate B only when Walrus/SuiNS track is in scope.
- Gate F only when GraphQL query fields or multi-source digest loader byte behavior changed.

```bash
# Gate A: no forbidden JSON-RPC path in migrated code
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate B: no legacy Walrus/SuiNS client construction (when optional track applies)
rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate C: React Query direct usage detection
rg "from ['\"]@tanstack/react-query['\"]|require\\(['\"]@tanstack/react-query['\"]\\)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate C.1: React Query dependency presence in manifests
rg "\"@tanstack/react-query\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"

# Gate C.2: hook-level usage triage
rg "\\buse(Query|Mutation|InfiniteQuery|Queries|SuspenseQuery|SuspenseInfiniteQuery|QueryClient)\\b" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate C.3: provider/wrapper-only pattern detection
rg "QueryClientProvider|new QueryClient\\(|<QueryClientProvider" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate C.4: legacy peer source check
rg "\"@mysten/dapp-kit\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"
rg "\"@mysten/dapp-kit-react\"\\s*:" . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"

# Gate D: hardcoded-network detector for digest loader paths (review required)
rg -n -P "new\\s+SuiGrpcClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -P "new\\s+SuiGraphQLClient\\(\\s*\\{[\\s\\S]{0,240}?network\\s*:\\s*['\\\"](?:mainnet|testnet|devnet|localnet)['\\\"]" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"

# Gate E: GraphQL connection-shape detector (review required)
rg -n -U -P "objectChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"
rg -n -U -P "balanceChanges\\s*\\{\\s*(?!nodes\\b|edges\\b|\\.\\.\\.)" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"

# Gate F trigger detector (conditional runtime gate)
rg -n "transactionBcs|transaction\\s*\\(digest:|getTransaction\\(|SuiGraphQLClient|query\\s*:\\s*`" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}" -g "!**/node_modules/**" -g "!.git/**"
```

Pass criteria:
- No matches in Gate A.
- No matches in Gate B for migrated Walrus/SuiNS paths.
- Gate C decision:
  - step 1: if Gate C has no source matches, `@tanstack/react-query` must be removed from manifests
  - step 2: if Gate C has matches, run Gate C.2 hook-level triage
    - if hook usage exists, dependency may remain and matched files/hook names must be reported
    - if hook usage does not exist, run Gate C.3 and Gate C.4
      - if only provider/wrapper patterns are present and legacy `@mysten/dapp-kit` is removed while `@mysten/dapp-kit-react` is present:
        treat as vestigial peer-dep wrapper, remove wrapper code, rerun Gate C, then remove `@tanstack/react-query` if clean
      - otherwise mark as `BLOCKER` for manual review (import exists but non-hook usage is unclear)
- Gate D decision:
  - Gate D matches require triage; unresolved matches are fail state.
  - For digest-based loaders, hardcoded literal network in client instantiation is a `BLOCKER` unless it is a documented chain-fixed flow with explicit rationale in code.
  - For digest-based loaders, require function-level network context (`network` parameter or equivalent caller-provided context), not module-scope fixed network.
- Gate E decision:
  - Gate E matches require triage; unresolved matches are fail state.
  - If query is confirmed to access connection item fields directly (without `nodes`/`edges`), mark as `BLOCKER` and fix query shape.
  - If query uses fragments or advanced wrappers, keep only as reviewed false-positive with file-level evidence.
- Gate F decision (conditional):
  - If migration changed GraphQL query fields in digest-loading paths, or changed two-stage byte loader behavior, Gate F runtime checks are required.
  - Missing Gate F evidence is fail state for applicable migrations.

If a gate fails:
- Mark as `BLOCKER`.
- Add TODO with file path and root cause.
- Re-run gate after minimal fix.
- Keep migration status as `FAIL` until blockers are resolved.

## 3) Transaction Loading Contract Gate

For digest-based transaction loading paths:
- Must use two-stage loading: gRPC first, GraphQL fallback.
- Must return explicit error when both sources miss.
- Must avoid silent null/empty success for verification paths.
- Must preserve caller network context (avoid fixed network in shared digest loader helpers).
- Must not confuse deployment target config with chain-verification context.
- Must check GraphQL `errors` before consuming `data`.
- Must include network and failed stage in error messages.
- For GraphQL list fields on `effects` (for example `objectChanges`, `balanceChanges`), treat them as connection types and access items via `nodes`/`edges`.

Suggested manual review checklist:
- [ ] `grpcClient.getTransaction({ digest, include: { bcs: true } })` exists in loader path.
- [ ] GraphQL `transaction(digest)` fallback exists and reads `transactionBcs`.
- [ ] explicit `not found/pruned` error is raised when both miss.
- [ ] loader receives network context from caller (parameter/config), not fixed module-scope literal.
- [ ] network source semantics are documented (chain-fixed flow vs caller-provided network); deployment config is not used as implicit chain context without explicit rationale.
- [ ] GraphQL path checks `response.errors` before `response.data`.
- [ ] thrown errors include `network=<value>` and failing stage (`grpc` or `graphql`).
- [ ] GraphQL connection fields under `effects` are accessed through `nodes`/`edges` (not direct item-field access).

## 4) Runtime Validation Gate

Run these scripts if present:
- `typecheck`
- `build`
- `test`

Use the project's package manager (`npm`, `pnpm`, `yarn`).

Pass criteria:
- all existing scripts pass
- unresolved blockers are not allowed in pass status

## 5) Endpoint/CORS Revalidation Gate

Do not assume endpoint or CORS behavior from old notes.
Revalidate in your own environment.

```bash
# Preflight check example
curl -sSI -X OPTIONS "<your-graphql-url>" \
  -H "Origin: http://localhost:3000" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type"

# Field check: transactionBcs should exist
curl -sS "<your-graphql-url>" \
  -H "content-type: application/json" \
  --data '{"query":"{ __type(name:\"Transaction\"){ fields { name } } }"}'
```

Record:
- checked URL
- check date
- origin used
- pass/fail result

## 6) Gate F Runtime Validation (Conditional)

Apply Gate F when migration changes:
- GraphQL query field selection in digest-loading paths, or
- two-stage loaders that combine gRPC and GraphQL bytes.

Why this gate exists:
- TypeScript does not validate GraphQL field names inside query strings by default.
- `SuiGraphQLClient.query<Result>()` generic types are developer-declared and can diverge from real schema fields.
- Byte arrays from different sources can have different binary contracts; static grep/type checks cannot prove runtime parse safety.

Required checks:
1. Schema introspection for target types before finalizing new query fields.
2. Query-level smoke check for `transaction(digest)` field paths used by your loader.
3. Fallback-path runtime test that actually executes the GraphQL branch (not only gRPC happy-path).
4. Source-aware byte normalization test (no unconditional shared byte-offset assumptions).

Suggested commands (replace placeholders):

```bash
# F.1 Schema field check
curl -sS "<your-graphql-url>" \
  -H "content-type: application/json" \
  --data '{"query":"{ __type(name:\"Transaction\"){ fields { name } } }"}'

curl -sS "<your-graphql-url>" \
  -H "content-type: application/json" \
  --data '{"query":"{ __type(name:\"TransactionEffects\"){ fields { name } } }"}'

# F.2 Query smoke check (field names + connection shape)
curl -sS "<your-graphql-url>" \
  -H "content-type: application/json" \
  --data '{"query":"query Q($digest:String!){ transaction(digest:$digest){ transactionBcs effects{ objectChanges{ nodes { outputState { address owner { __typename } } } } } } }","variables":{"digest":"<digest>"}}'
```

Pass criteria:
- Gate F is explicitly marked `N/A` with evidence when no relevant code was changed.
- For applicable changes, introspection and query smoke outputs are attached to the step/audit report.
- GraphQL fallback path execution is evidenced (for example log/source marker, integration test output, or fixture replay).
- Byte normalization logic is source-aware and covered by runtime tests on both sources.

## 7) Recommended CI Integration

At minimum, run:
1. Gate A and Gate B pattern checks
2. Gate F checks when GraphQL query fields or two-stage loader code changed
3. typecheck/build/test
4. store failures with file path and line number

If your org already uses policy tooling:
- Semgrep or OPA/Conftest can encode these checks as policy-as-code.
- Keep this file as the source contract even when tooling differs.
