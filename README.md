# sui-grpc-migration

Migration kit for upgrading Sui SDK projects from `@mysten/sui` 1.x to 2.x with AI-assisted execution.

> Scope: `@mysten/sui` 1.x -> 2.x (assumes `@mysten/sui.js` rename migration already done)
> 
> Target versions: `@mysten/sui@^2.4.0`, `@mysten/dapp-kit-react@^1.0.2`, `@mysten/dapp-kit-core@^1.0.4`
> 
> Runtime: Node `>=22`

## Repository Map

```text
sui-grpc-migration/
- README.md
- docs/
  - migration-run-order.md      <- scenario detection + execution order
  - migration-step-prompts.md   <- Step 1~5 and Optional A~D prompt blocks
  - policy-gates.md             <- machine-verifiable migration policy gates
  - sui-grpc-migration.md       <- Sui SDK migration reference (API details)
  - walrus-suins-migration.md   <- Walrus/SuiNS migration reference (optional track)
- agents/
  - sui-grpc-rules.md           <- rules to inject into target project AI rules file
```

## Quick Start

```bash
git clone https://github.com/zktx-io/sui-grpc-migration.git
```

## Scope Model (Core vs Conditional)

Core (universal, always apply):
- `@mysten/sui` 1.x -> 2.x migration rules
- dApp Kit -> dApp Kit React migration rules
- mandatory policy gates (A-E) and validation flow
- no unresolved blocker in PASS status

Conditional (apply only when triggered):
- Walrus/SuiNS migration track (Case B/C)
- chain-fixed network exception (allowed only with explicit rationale comment)
- Gate F runtime validation (required only when GraphQL field selection or multi-source loader byte behavior changed)

## Recommended Execution (detailed prompt, all repo types)

Paste this once into your AI agent:

```text
Run migration end-to-end in one pass (no step-by-step confirmation).

Read first:
- docs/migration-run-order.md
- docs/migration-step-prompts.md
- docs/policy-gates.md
- docs/sui-grpc-migration.md
- docs/walrus-suins-migration.md (only if Walrus/SuiNS is detected)

1) Detect repository/workspace scope first (monorepo-safe):
- List package manifests:
  rg --files . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"
- Treat search scope as repository root `.` with globs:
  - include: `**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}` and `**/package.json`
  - exclude: `**/node_modules/**`, `.git/**`
- Print detected package roots and scope first.

2) Auto-detect scenario now:
- Run:
  rg "\"@mysten/walrus\"|\"@mysten/suins\"|new WalrusClient|new SuinsClient|\\$extend\\(walrus\\)|\\$extend\\(suins\\)" . -g "**/package.json" -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
- Also inspect matched package manifests for `@mysten/sui` major version intent.
- Select exactly one: Case A / Case B / Case C.
- Print scenario evidence first.

3) Execute exact step order for selected case from docs/migration-run-order.md.

4) For each step, use the exact prompt/checklist from docs/migration-step-prompts.md and execute immediately.

4.1) Apply policy gates from docs/policy-gates.md after each migration phase and in final validation.

5) Do not stop between normal steps. Ask only when:
- permission/escalation is required
- destructive action is required
- a real ambiguous blocker remains

6) Apply minimal fixes during validation and rerun until pass.
   If blockers remain, mark final status as FAIL.

Hard policies:
- Never introduce SuiJsonRpcClient.
- Never import from @mysten/sui/jsonRpc.
- Digest transaction content loading must use a two-stage loader: gRPC `getTransaction(...)` first, then GraphQL fallback with explicit error when both miss.
- In shared digest loaders, do not hardcode network at module scope; use caller-provided network context.
- If a flow is intentionally chain-fixed, hardcoded network is allowed only with an explicit rationale comment.
- In GraphQL fallback, check `errors` before consuming `data`, and include network + failed stage in thrown errors.
- In GraphQL transaction queries, treat `effects` list fields as connection types (`nodes`/`edges` access).
- When GraphQL query fields or two-stage loader byte behavior changes, Gate F runtime validation is required.
- Apply React Query usage gate: remove `@tanstack/react-query` when no direct app imports exist; keep only with usage evidence.
- Unresolved blockers or unresolved gate findings are not allowed in a PASS result.

Per-step output (mandatory):
1. step name + completion status
2. changed files or commands executed
3. verify checklist pass/fail
4. TODO/BLOCKER list
5. ready for next step: yes/no

Final output format:
1. selected scenario + evidence
2. commands executed
3. migrated file list
4. key diffs summary
5. validation results
6. blockers/TODOs
```

## Recommended Execution (compact prompt, all repo types)

Paste this once when you want a shorter control prompt:

```text
Run migration end-to-end in one pass.

Read and follow exactly:
- docs/migration-run-order.md
- docs/migration-step-prompts.md
- docs/policy-gates.md
- docs/sui-grpc-migration.md
- docs/walrus-suins-migration.md (only when Walrus/SuiNS is detected)

Required flow:
1) Detect workspace/package scope (monorepo-safe).
2) Detect scenario (Case A/B/C) with evidence.
3) Execute exact step order from migration-run-order.md.
4) Use exact step prompts/checklists from migration-step-prompts.md.
5) Run policy gates after each migration phase and in final validation.
6) Apply minimal fixes and rerun validation until pass.
7) If any blocker or unresolved gate finding remains, final status must be FAIL.
8) After the main flow, run Step 6 from migration-step-prompts.md as a separate final audit pass.

Output:
1. selected scenario + evidence
2. commands executed
3. migrated file list
4. key diffs summary
5. validation results
6. blockers/TODOs
```

## Which Prompt to Use

Important:
- Prompt choice is about **detail level**, not repo type.
- Both prompts support single-package repos and monorepos.

- Use `Recommended Execution (detailed prompt, all repo types)` when:
  - You want full policy text inline in one prompt.
  - You are onboarding a new team/member and want fewer assumptions.
  - You need maximum reproducibility in audit logs.

- Use `Recommended Execution (compact prompt, all repo types)` when:
  - You already trust this repo's docs as the source of truth.
  - You want faster repeated runs with less prompt noise.
  - You are running frequent iterations in any repo layout.

- In both cases:
  - Scenario selection (Case A/B/C) is automatic from `docs/migration-run-order.md`.
  - Step 6 final audit should run as a separate follow-up pass.

Execution contract:
- This repository is designed for copy/paste usage.
- Keep one-shot prompt execution as the primary workflow.
- Added policy gates are internal checks run by the agent in the same flow, not an extra user workflow.
- For reliability, run final audit (Step 6) as a separate second prompt after the main flow.
- The same one-shot prompt supports both single-package repos and monorepos.

## Final Audit (separate prompt, recommended)

Paste this after the main migration prompt completes:

```text
Run Step 6 from docs/migration-step-prompts.md as a separate final audit pass.

Rules:
- run in read-only audit mode first
- apply only minimal fixes if required
- rerun audit after fixes
- include Gate F status (runtime schema/format validation) as `PASS` or `N/A` with reason

Output:
1. PASS/FAIL summary
2. findings by severity with file:line
3. minimal patch diff (if any)
4. rerun result after patch
```

## Manual Mode (if needed)

1. Open `docs/migration-run-order.md` and select Case A/B/C.
2. Open `docs/migration-step-prompts.md` and execute each step prompt in that order.
3. Use `docs/sui-grpc-migration.md` and `docs/walrus-suins-migration.md` as API truth references.

## AI Rules File Path

| Tool | Path |
|---|---|
| Cursor | `.cursor/rules/sui-grpc.md` |
| Claude Code | `CLAUDE.md` |
| Codex / Gemini / general | `AGENTS.md` |
