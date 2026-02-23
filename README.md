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

## Recommended Execution (single prompt)

Paste this once into your AI agent:

```text
Run migration end-to-end in one pass (no step-by-step confirmation).

Read first:
- docs/migration-run-order.md
- docs/migration-step-prompts.md
- docs/policy-gates.md
- docs/sui-grpc-migration.md
- docs/walrus-suins-migration.md (only if Walrus/SuiNS is detected)

1) Auto-detect scenario now:
- Run:
  rg "\"@mysten/walrus\"|\"@mysten/suins\"|new WalrusClient|new SuinsClient|\\$extend\\(walrus\\)|\\$extend\\(suins\\)" package.json src -g "package.json" -g "*.ts" -g "*.tsx" -g "*.js" -g "*.mjs"
- Also inspect package.json for @mysten/sui major version intent.
- Select exactly one: Case A / Case B / Case C.
- Print scenario evidence first.

2) Execute exact step order for selected case from docs/migration-run-order.md.

3) For each step, use the exact prompt/checklist from docs/migration-step-prompts.md and execute immediately.

3.1) Apply policy gates from docs/policy-gates.md after each migration phase and in final validation.

4) Do not stop between normal steps. Ask only when:
- permission/escalation is required
- destructive action is required
- a real ambiguous blocker remains

5) Apply minimal fixes during validation and rerun until pass or clear blocker.

Hard policies:
- Never introduce SuiJsonRpcClient.
- Never import from @mysten/sui/jsonRpc.
- Digest transaction content loading must use a two-stage loader: gRPC `getTransaction(...)` first, then GraphQL fallback with explicit error when both miss.

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

Execution contract:
- This repository is designed for copy/paste usage.
- Keep one-shot prompt execution as the primary workflow.
- Added policy gates are internal checks run by the agent in the same flow, not an extra user workflow.

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
