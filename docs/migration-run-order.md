# Migration Run Order (Scenario-first)

Use this file to decide **execution order** before running any migration step.
Detailed step prompts are in `docs/migration-step-prompts.md`.
Policy enforcement gates are in `docs/policy-gates.md`.

## 0. Scope

Core track (universal):
- `@mysten/sui` 1.x -> 2.x migration
- dApp Kit -> dApp Kit React migration

Conditional track (selected by scenario):
- `@mysten/walrus` / `@mysten/suins` 0.x -> 1.x
- chain-fixed network exception (only with explicit rationale comment)
- Gate F runtime validation (only when trigger conditions are met)

This file defines scenario detection, case selection, and execution order.
Workflow style contract: keep copy/paste + single-prompt execution as default.

## 1. Core Policies (must hold in every case)

- Do not introduce `SuiJsonRpcClient`.
- Do not introduce imports from `@mysten/sui/jsonRpc`.
- Legacy-detection keywords should be limited to 1.x-era symbols.
- Post-migration quality gates may inspect 2.x loader patterns.
- For digest-based transaction content loading, use a **two-stage loader** (`gRPC getTransaction` first, then GraphQL fallback with explicit error when both miss).
- For shared digest loaders, keep network caller-driven (no module-scope fixed network literals).
- In GraphQL fallback, check `errors` before reading `data`, and include network/stage in loader errors.
- In GraphQL transaction queries, treat `effects` list fields as connection types and use `nodes`/`edges` access.
- For same-flow submit confirmation (`waitForTransaction`), gRPC is allowed.
- Unresolved `BLOCKER` items are fail state (no accepted-blocker shortcut).

## 2. Conditional Policies

- If a flow is chain-fixed, hardcoded network is allowed only with explicit rationale in code comments.
- When GraphQL query fields or two-stage byte loader behavior changes, Gate F runtime validation is required.
- Walrus/SuiNS-specific checks apply only in Case B/C.

## 3. Repository/Workspace Scope (monorepo-safe)

Run this once before scenario detection:

```bash
# package manifests (single package + monorepo)
rg --files . -g "**/package.json" -g "!**/node_modules/**" -g "!.git/**"
```

All scan/gate commands in this kit should use repository root `.` with explicit include/exclude globs:
- include: `**/*.{ts,tsx,js,jsx,mjs,cjs,graphql,gql}` and `**/package.json`
- exclude: `**/node_modules/**`, `.git/**`

## 4. Scenario Detection

Run this detection command first:

```bash
rg "\"@mysten/walrus\"|\"@mysten/suins\"|new WalrusClient|new SuinsClient|\\$extend\\(walrus\\)|\\$extend\\(suins\\)" . -g "**/package.json" -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
```

Then inspect matched package manifests for intended `@mysten/sui` major version.

## 5. Case Selection

### Case A: Sui-only migration

Choose Case A when:
- No Walrus/SuiNS usage is detected
- You are migrating `@mysten/sui` + dApp Kit paths

Execution order:
1. Step 1
2. Step 2
3. Step 3
4. Step 4
5. Step 5

### Case B: Sui + Walrus/SuiNS together

Choose Case B when:
- Walrus/SuiNS usage is detected
- Sui core migration and Walrus/SuiNS migration are both needed

Execution order:
1. Step 1
2. Step 2
3. Step 3
4. Optional Step A
5. Optional Step B
6. Optional Step C
7. Step 3 (rerun once for shared backend integration drift)
8. Step 4
9. Optional Step D
10. Step 5

### Case C: Sui 2.x already done, Walrus/SuiNS-only migration

Choose Case C when:
- `@mysten/sui` migration is already complete
- Only Walrus/SuiNS migration remains

Execution order:
1. Optional Step A
2. Optional Step B
3. Optional Step C
4. Optional Step D
5. Step 5

Post-run recommendation (all cases):
- Run Step 6 (final audit) from `docs/migration-step-prompts.md` as a separate prompt.

## 6. Integration Gates (Case B/C)

Run these checks after Optional Step C and before final validation:

```bash
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" . -g "**/*.{ts,tsx,js,jsx,mjs,cjs}" -g "!**/node_modules/**" -g "!.git/**"
```

If any match remains:
- Keep migration status as incomplete
- Add `BLOCKER` + TODO
- Continue only after explicit fix

## 7. One-Prompt Mode (Confirm-only)

Paste this into your AI agent when you want one continuous run:

```text
Read docs/migration-run-order.md and docs/migration-step-prompts.md.

Do the migration end-to-end with minimal user interruption:
- Detect workspace/package scope first (single repo + monorepo).
- Auto-detect scenario (Case A/B/C) using the detection command in migration-run-order.md.
- Execute exact step order for that case.
- Run validation gates and per-step reports exactly as defined.
- Keep final audit (Step 6) as a separate follow-up prompt for reliability.
- Do not stop between normal steps.
- Ask for confirmation only when:
  1) command permission/escalation is required
  2) destructive action is required
  3) a true ambiguous blocker remains

Final output:
1. selected scenario + evidence
2. commands executed
3. migrated file list
4. key diffs summary
5. validation results
6. blockers/TODOs
```

## 8. Required Step Report Format (every step)

Each completed step must output:
1. step name + completion status
2. changed files or commands executed
3. verify checklist pass/fail
4. TODO/BLOCKER list
5. ready for next step: yes/no
