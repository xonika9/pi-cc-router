# Phase 6: Testing and Release Pipeline - Context

**Gathered:** 2026-03-15
**Status:** Ready for planning

<domain>
## Phase Boundary

Comprehensive unit tests with gap analysis, cross-platform CI (lint, typecheck, tests on Windows/macOS/Linux), and automated npm publishing on tagged releases. Covers requirements RLSE-01, RLSE-02, RLSE-03, RLSE-04.

</domain>

<decisions>
## Implementation Decisions

### Lint Tooling

- ESLint + Prettier — ESLint for logic/correctness rules, Prettier for formatting enforcement
- ESLint rules: `typescript-eslint/recommended` defaults — catches real bugs without being noisy
- Prettier config: defaults (2-space indent, 80 char width, semicolons, double quotes)
- CI behavior: fail on violations, no auto-fix commits — developers fix locally

### Release Trigger

- Manual `v*` git tags trigger the publish workflow (e.g., `git tag v0.1.0 && git push --tags`)
- CI validates that the tag version matches `package.json` version — fail if mismatched
- After successful npm publish, auto-create a GitHub Release with generated changelog/notes
- Semver tag format: `v*` prefix (v0.1.0, v1.0.0-beta.1)

### npm Publishing

- Unscoped package name: `pi-claude-cli` (matches current package.json)
- Public on npm (required for pi extension discovery)
- Auth via `NPM_TOKEN` GitHub Actions secret (npm automation token)
- No build step — publish TypeScript source as-is (pi runtime handles TS natively)
- `mcp-schema-server.cjs` already ships as plain JS (Claude CLI spawns with node)

### Test Coverage

- Gap analysis: researcher identifies untested paths (edge cases, error paths) across all 9 modules
- Fill meaningful gaps only — no padding tests for coverage numbers
- Coverage threshold enforced in CI — fail if coverage drops below minimum
- Researcher determines appropriate threshold from current coverage baseline
- 248 tests already passing across all modules (as of Phase 5 completion)

### CI Pipeline (RLSE-02)

- GitHub Actions workflow on every push and PR to main
- Matrix: Windows, macOS, Linux runners
- Steps: lint (ESLint + Prettier check), typecheck (`tsc --noEmit`), test (`vitest run`), coverage check
- Separate publish workflow triggered only by `v*` tags

### Package Metadata (RLSE-04)

- Already satisfied: `"pi-package"` keyword, `pi.extensions` entry, correct peer dependencies
- No changes needed — verify during planning that nothing regressed

### Claude's Discretion

- Exact ESLint rule overrides if recommended defaults conflict with existing code patterns
- Coverage provider choice (v8 vs istanbul)
- Exact coverage threshold percentage (based on gap analysis results)
- GitHub Actions workflow file structure (single file vs split CI/publish)
- Node.js version matrix for CI runners

</decisions>

<specifics>
## Specific Ideas

- Phase 5 completed — all 248 tests pass, no failing tests to handle
- `mcp-schema-server.cjs` is the only non-TypeScript source file and already exists
- `vitest.config.ts` already configured with `globals: true` — extend for coverage
- `package.json` already has `test` and `typecheck` scripts — add `lint` script

</specifics>

<code_context>

## Existing Code Insights

### Reusable Assets

- `vitest.config.ts`: Already configured, needs coverage settings added
- `package.json`: Has `test` and `typecheck` scripts, needs `lint` and `format:check` scripts
- `tsconfig.json`: Includes `src/**/*.ts`, `index.ts`, `tests/**/*.ts` — typecheck scope is set

### Established Patterns

- Tests in `tests/` directory (not `src/__tests__/`)
- Vitest with globals enabled (no explicit imports needed)
- Mocked subprocess I/O using PassThrough streams for readline compatibility
- `vi.hoisted()` for mock references surviving vitest hoisting

### Integration Points

- `.github/workflows/ci.yml` — new file for CI pipeline
- `.github/workflows/publish.yml` — new file for npm publish on tags
- `.eslintrc.*` or `eslint.config.*` — new ESLint config
- `.prettierrc` — new Prettier config (or defaults)

</code_context>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

_Phase: 06-testing-and-release-pipeline_
_Context gathered: 2026-03-15_
