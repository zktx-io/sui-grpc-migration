# Migration Policy Gates (Standardized Checks)

Use this file to enforce migration policy with repeatable checks.
The goal is to move from doc-only guidance to machine-verifiable gates.
This does not change the user workflow: gates are run inside the same one-shot agent prompt flow.

## 1) Gate Philosophy

- Migration prompts are guidance.
- Policy gates are enforcement.
- A migration is not complete until all gates pass or blockers are explicitly accepted.

## 2) Mandatory Gates

Run from project root after each migration phase and in CI.

```bash
# Gate A: no forbidden JSON-RPC path in migrated code
rg "SuiJsonRpcClient|@mysten/sui/jsonRpc|getJsonRpcFullnodeUrl" src

# Gate B: no legacy Walrus/SuiNS client construction (when optional track applies)
rg "new WalrusClient|new SuinsClient|@mysten/sui/client|getFullnodeUrl|@mysten/sui/experimental" src

# Gate C: React Query direct usage detection
rg "from ['\"]@tanstack/react-query['\"]|require\\(['\"]@tanstack/react-query['\"]\\)" . -g "*.{ts,tsx,js,jsx,mjs,cjs}" -g "!node_modules/**" -g "!.git/**"

# Gate C.1: React Query dependency presence in manifests
rg "\"@tanstack/react-query\"\\s*:" . -g "package.json" -g "!node_modules/**" -g "!.git/**"
```

Pass criteria:
- No matches in Gate A.
- No matches in Gate B for migrated Walrus/SuiNS paths.
- Gate C decision:
  - if Gate C has no source matches, `@tanstack/react-query` must be removed from manifests
  - if Gate C has matches, `@tanstack/react-query` may remain and matched files must be reported

If a gate fails:
- Mark as `BLOCKER`.
- Add TODO with file path and root cause.
- Re-run gate after minimal fix.

## 3) Transaction Loading Contract Gate

For digest-based transaction loading paths:
- Must use two-stage loading: gRPC first, GraphQL fallback.
- Must return explicit error when both sources miss.
- Must avoid silent null/empty success for verification paths.

Suggested manual review checklist:
- [ ] `grpcClient.getTransaction({ digest, include: { bcs: true } })` exists in loader path.
- [ ] GraphQL `transaction(digest)` fallback exists and reads `transactionBcs`.
- [ ] explicit `not found/pruned` error is raised when both miss.

## 4) Runtime Validation Gate

Run these scripts if present:
- `typecheck`
- `build`
- `test`

Use the project's package manager (`npm`, `pnpm`, `yarn`).

Pass criteria:
- all existing scripts pass
- or a clear reproducible blocker is documented

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

## 6) Recommended CI Integration

At minimum, run:
1. Gate A and Gate B pattern checks
2. typecheck/build/test
3. store failures with file path and line number

If your org already uses policy tooling:
- Semgrep or OPA/Conftest can encode these checks as policy-as-code.
- Keep this file as the source contract even when tooling differs.
