# Phase 6: Testing and Release Pipeline - Research

**Researched:** 2026-03-15
**Domain:** CI/CD, lint tooling, test coverage, npm publishing, GitHub Actions
**Confidence:** HIGH

## Summary

Phase 6 adds ESLint + Prettier enforcement, extends the existing Vitest test suite with coverage analysis, builds GitHub Actions CI/CD workflows (cross-platform lint/typecheck/test + automated npm publish on tags), and verifies existing package metadata. The project already has 248 passing tests across 9 test files covering all 9 source modules. The primary work is adding lint tooling, coverage configuration, and two GitHub Actions workflow files -- plus filling any meaningful test gaps identified during gap analysis.

ESLint 10 (released February 2026) is the current major version and is compatible with typescript-eslint v8.56.0+. The flat config format is the only option in ESLint 10 (eslintrc is completely removed). Node.js 20.19.0+ is required by ESLint 10, which aligns with the active LTS schedule (Node 20, 22, 24 all active in March 2026).

**Primary recommendation:** Use ESLint 10 + typescript-eslint 8 with flat config (`eslint.config.mjs`), Prettier 3.8 for formatting (run separately via `prettier --check`), `@vitest/coverage-v8` for coverage with CI threshold enforcement, and two separate GitHub Actions workflow files (ci.yml + publish.yml).

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- ESLint + Prettier -- ESLint for logic/correctness rules, Prettier for formatting enforcement
- ESLint rules: `typescript-eslint/recommended` defaults -- catches real bugs without being noisy
- Prettier config: defaults (2-space indent, 80 char width, semicolons, double quotes)
- CI behavior: fail on violations, no auto-fix commits -- developers fix locally
- Manual `v*` git tags trigger the publish workflow (e.g., `git tag v0.1.0 && git push --tags`)
- CI validates that the tag version matches `package.json` version -- fail if mismatched
- After successful npm publish, auto-create a GitHub Release with generated changelog/notes
- Semver tag format: `v*` prefix (v0.1.0, v1.0.0-beta.1)
- Unscoped package name: `pi-claude-cli` (matches current package.json)
- Public on npm (required for pi extension discovery)
- Auth via `NPM_TOKEN` GitHub Actions secret (npm automation token)
- No build step -- publish TypeScript source as-is (pi runtime handles TS natively)
- `mcp-schema-server.cjs` already ships as plain JS (Claude CLI spawns with node)
- Gap analysis: researcher identifies untested paths across all 9 modules
- Fill meaningful gaps only -- no padding tests for coverage numbers
- Coverage threshold enforced in CI -- fail if coverage drops below minimum
- 248 tests already passing across all modules (as of Phase 5 completion)
- GitHub Actions workflow on every push and PR to main
- Matrix: Windows, macOS, Linux runners
- Steps: lint (ESLint + Prettier check), typecheck (`tsc --noEmit`), test (`vitest run`), coverage check
- Separate publish workflow triggered only by `v*` tags
- Package metadata (RLSE-04) already satisfied: `"pi-package"` keyword, `pi.extensions` entry, correct peer dependencies -- verify, no changes needed

### Claude's Discretion
- Exact ESLint rule overrides if recommended defaults conflict with existing code patterns
- Coverage provider choice (v8 vs istanbul)
- Exact coverage threshold percentage (based on gap analysis results)
- GitHub Actions workflow file structure (single file vs split CI/publish)
- Node.js version matrix for CI runners

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| RLSE-01 | Unit tests with mocked subprocess I/O covering NDJSON parsing, event bridging, tool name/argument mapping, control protocol logic, and prompt building | 248 tests already exist across all modules; gap analysis section identifies remaining edge cases to fill |
| RLSE-02 | GitHub Actions CI runs lint, typecheck, and unit tests on Windows, macOS, Linux | CI workflow pattern documented with matrix strategy; ESLint 10 + Prettier + tsc + vitest commands specified |
| RLSE-03 | Tagged releases automatically publish to npm with correct pi-package metadata | Publish workflow documented with v* tag trigger, version validation, npm publish, and GitHub Release creation |
| RLSE-04 | package.json includes "pi-package" keyword, pi.extensions entry, and correct peer dependencies | Already satisfied in current package.json -- verify only, no changes needed |

</phase_requirements>

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| eslint | ^10.0.0 | Linting logic/correctness rules | Current major version (Feb 2026), flat config only, better monorepo support |
| @eslint/js | ^10.0.0 | ESLint recommended JS rules | Companion package for ESLint 10 recommended config |
| typescript-eslint | ^8.57.0 | TypeScript ESLint rules and parser | Latest v8 with ESLint 10 support (peer dep: eslint ^8.57.0 \|\| ^9.0.0 \|\| ^10.0.0) |
| prettier | ^3.8.0 | Code formatting enforcement | Current stable (3.8.1), opinionated defaults match user decisions |
| eslint-config-prettier | ^10.1.0 | Disables ESLint rules that conflict with Prettier | Required to prevent ESLint vs Prettier conflicts in flat config |
| @vitest/coverage-v8 | ^3.0.0 | Coverage collection via V8 | Default vitest coverage provider; fast, accurate (AST-based since v3.2.0), no instrumentation needed |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| actions/checkout | v6 | GitHub Actions checkout step | Every workflow job |
| actions/setup-node | v6 | GitHub Actions Node.js setup with registry-url | Every workflow job; v6 supports automatic npm caching |
| softprops/action-gh-release | v2 | Create GitHub Release after publish | Publish workflow, after npm publish succeeds |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| ESLint 10 | ESLint 9 | ESLint 9 still maintained (v9.39.3) but 10 is current; no reason to use old version for new project |
| v8 coverage | istanbul | v8 is faster, lower memory; since vitest 3.2.0 produces identical reports to istanbul |
| Separate ESLint + Prettier | eslint-plugin-prettier | Plugin approach is slower (runs Prettier inside ESLint); separate commands are faster and simpler |
| softprops/action-gh-release | gh release create | Both work; softprops/action-gh-release handles --generate-notes cleanly in YAML |

**Installation:**
```bash
npm install --save-dev eslint @eslint/js typescript-eslint prettier eslint-config-prettier @vitest/coverage-v8
```

## Architecture Patterns

### Recommended Project Structure (New Files)
```
.github/
  workflows/
    ci.yml              # Push/PR CI: lint, typecheck, test, coverage
    publish.yml         # Tag-triggered: npm publish + GitHub Release
eslint.config.mjs       # ESLint 10 flat config
.prettierrc             # (Optional - use defaults; empty file or omit entirely)
```

### Pattern 1: ESLint 10 Flat Config with TypeScript
**What:** Single `eslint.config.mjs` file using `defineConfig` from eslint/config
**When to use:** Always -- the only config format ESLint 10 supports

```javascript
// eslint.config.mjs
// @ts-check
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import eslintConfigPrettier from "eslint-config-prettier/flat";
import { defineConfig, globalIgnores } from "eslint/config";

export default defineConfig([
  globalIgnores(["node_modules/", "dist/", "coverage/", "package/"]),
  eslint.configs.recommended,
  tseslint.configs.recommended,
  eslintConfigPrettier,
  {
    files: ["**/*.ts"],
    rules: {
      // Overrides if needed after checking against existing code
      "@typescript-eslint/no-explicit-any": "off", // Project uses `any` intentionally (Context type, etc.)
    },
  },
  {
    files: ["**/*.cjs"],
    languageOptions: {
      sourceType: "commonjs",
    },
  },
]);
```

**Key decisions for this project:**
- `@typescript-eslint/no-explicit-any` MUST be turned off -- the codebase uses `any` intentionally in 20+ places (Context parameter, pi ExtensionAPI, tool args, message content)
- `eslint-config-prettier/flat` must be LAST in the array to properly override conflicting rules
- The `.cjs` file (`mcp-schema-server.cjs`) needs a separate block with `sourceType: "commonjs"`
- Use `globalIgnores` for directory exclusions (replaces `.eslintignore`)

### Pattern 2: Vitest Coverage Configuration
**What:** Extend existing `vitest.config.ts` with coverage settings
**When to use:** Enables coverage in CI and local development

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    coverage: {
      provider: "v8",
      reporter: ["text", "json-summary", "html"],
      include: ["src/**/*.ts", "index.ts"],
      exclude: ["src/mcp-schema-server.cjs"],
      thresholds: {
        lines: 80,    // Adjust based on gap analysis results
        functions: 80,
        branches: 70,
        statements: 80,
      },
    },
  },
});
```

### Pattern 3: CI Workflow (Matrix Strategy)
**What:** GitHub Actions workflow with OS matrix for cross-platform testing
**When to use:** On every push to main and every PR

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: "22"
      - run: npm ci
      - run: npx eslint .
      - run: npx prettier --check .

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: "22"
      - run: npm ci
      - run: npm run typecheck

  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: "22"
      - run: npm ci
      - run: npm run test -- --coverage
```

**Design decisions:**
- Lint + typecheck run only on Ubuntu (no need for OS matrix -- they're platform-independent)
- Tests run on all 3 OS platforms (the actual cross-platform validation requirement)
- Node 22 is the recommended target (active LTS through Apr 2027; ESLint 10 requires >= 20.19.0)
- `npm ci` for reproducible installs (uses lockfile)

### Pattern 4: Publish Workflow (Tag-Triggered)
**What:** Separate workflow triggered by v* tags
**When to use:** When maintainer pushes a version tag

```yaml
# .github/workflows/publish.yml
name: Publish
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: "22"
          registry-url: "https://registry.npmjs.org"
      - run: npm ci

      # Version validation: tag must match package.json version
      - name: Validate version match
        run: |
          TAG_VERSION="${GITHUB_REF#refs/tags/v}"
          PKG_VERSION=$(node -p "require('./package.json').version")
          if [ "$TAG_VERSION" != "$PKG_VERSION" ]; then
            echo "::error::Tag version ($TAG_VERSION) does not match package.json version ($PKG_VERSION)"
            exit 1
          fi

      # Run full test suite before publishing
      - run: npm run test

      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

**Key details:**
- `permissions: contents: write` is required for creating GitHub Releases
- `permissions: id-token: write` enables npm provenance (supply chain security)
- `registry-url` in setup-node creates `.npmrc` with `NODE_AUTH_TOKEN` placeholder
- Version validation uses shell comparison -- no third-party action needed
- `--provenance` flag provides npm attestation (trusted publishing)
- `--access public` required for first publish of unscoped package
- `softprops/action-gh-release@v2` with `generate_release_notes: true` auto-generates changelog

### Anti-Patterns to Avoid
- **Running Prettier through ESLint plugin:** Slower than running separately; also harder to debug. Use `eslint-config-prettier` to disable conflicts, run `prettier --check` separately.
- **Auto-fix commits in CI:** Creates noise in git history, masks developer responsibility. CI should fail, developer fixes locally.
- **Coverage threshold too high initially:** Will block all PRs until every edge case is tested. Start conservatively (70-80%), ratchet up over time.
- **Single workflow for CI and publish:** Makes the CI workflow more complex with conditional steps. Two small files are clearer.
- **Using `npm install` in CI:** Non-deterministic. Always use `npm ci` for reproducible builds.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ESLint config format detection | Custom config loader | `eslint.config.mjs` (flat config) | ESLint 10 removed eslintrc entirely |
| Prettier conflict resolution | Manual rule disabling | `eslint-config-prettier` | Maintains itself as rules change |
| Coverage collection | Custom instrumentation | `@vitest/coverage-v8` | V8 native coverage, zero overhead |
| Version tag validation | Custom GitHub Action | Shell script in workflow step | 3 lines of bash, no dependency needed |
| GitHub Release creation | Custom API calls | `softprops/action-gh-release` | Handles release notes, assets, permissions |
| npm auth in CI | Manual `.npmrc` creation | `actions/setup-node` with `registry-url` | Handles auth token injection securely |

**Key insight:** The tooling ecosystem handles all CI/CD complexity. The project's job is to configure it correctly, not build custom solutions.

## Common Pitfalls

### Pitfall 1: ESLint `no-explicit-any` False Positives
**What goes wrong:** ESLint's `@typescript-eslint/no-explicit-any` rule flags dozens of intentional `any` uses across the codebase (Context parameter, pi ExtensionAPI, message content arrays, tool arguments).
**Why it happens:** The project intentionally uses `any` to avoid requiring `@mariozechner/pi-ai` types at dev time and for flexible message content handling.
**How to avoid:** Disable the rule globally in `eslint.config.mjs`. The project has 20+ intentional `any` usages -- suppressing each with inline comments would be worse than disabling the rule.
**Warning signs:** ESLint reports 20+ violations on first run, all in type positions.

### Pitfall 2: ESLint Treating .cjs as ESM
**What goes wrong:** ESLint tries to parse `mcp-schema-server.cjs` as an ES module and fails on `require()` / `module.exports`.
**Why it happens:** Default config assumes all JS is ESM when `tsconfig.json` has `"module": "ESNext"`.
**How to avoid:** Add a separate config block for `*.cjs` files with `languageOptions: { sourceType: "commonjs" }`.
**Warning signs:** Parse errors on `mcp-schema-server.cjs` mentioning `require is not defined`.

### Pitfall 3: Prettier Double Quotes vs Existing Code
**What goes wrong:** Prettier defaults to double quotes, but existing code might use single quotes in some places. First `prettier --check` run may report many files.
**Why it happens:** User decided on Prettier defaults (double quotes), but code may not be consistently formatted yet.
**How to avoid:** Run `npx prettier --write .` once to format all files before adding the CI check. Include this formatting commit as part of the implementation.
**Warning signs:** CI immediately fails on the first PR after adding Prettier check.

### Pitfall 4: Coverage Threshold Too Aggressive
**What goes wrong:** Setting coverage threshold to 90%+ before running coverage analysis causes immediate CI failure.
**Why it happens:** Not all code paths are easily testable (subprocess spawning, filesystem operations, process exit handlers).
**How to avoid:** Run `vitest run --coverage` locally first to establish a baseline, then set thresholds 5-10% below the baseline. Ratchet up as gaps are filled.
**Warning signs:** Coverage report shows untestable code paths (process.exit handlers, real filesystem operations).

### Pitfall 5: npm Publish Without `--access public`
**What goes wrong:** First publish of an unscoped package defaults to public, but can be surprising. Subsequent publishes work without it.
**Why it happens:** npm's default access for unscoped packages is `public`, but being explicit avoids any configuration-level overrides.
**How to avoid:** Always include `--access public` in the publish command.
**Warning signs:** Publish step fails with access error.

### Pitfall 6: Missing `npm-shrinkwrap.json` / `package-lock.json`
**What goes wrong:** `npm ci` fails if there's no lockfile. The project currently has no lockfile visible.
**Why it happens:** Pi extensions may not traditionally use lockfiles, but CI requires them for `npm ci`.
**How to avoid:** Run `npm install` once to generate `package-lock.json` and commit it. Alternatively, use `npm install` in CI instead of `npm ci` (less reproducible but works without lockfile).
**Warning signs:** CI fails at the `npm ci` step with "no package-lock.json found".

### Pitfall 7: Windows Path Separators in Coverage Include
**What goes wrong:** Coverage `include` patterns like `src/**/*.ts` may not match on Windows if paths use backslashes.
**Why it happens:** Glob patterns use forward slashes but Windows paths use backslashes.
**How to avoid:** Vitest/minimatch handles this internally. Just use forward slashes in config -- vitest normalizes them cross-platform.
**Warning signs:** Coverage shows 0% on Windows but works on Linux/macOS.

## Code Examples

### ESLint Config (eslint.config.mjs)
```javascript
// Source: ESLint 10 flat config docs + typescript-eslint getting-started
// @ts-check
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import eslintConfigPrettier from "eslint-config-prettier/flat";
import { defineConfig, globalIgnores } from "eslint/config";

export default defineConfig([
  globalIgnores(["node_modules/", "dist/", "coverage/", "package/"]),
  eslint.configs.recommended,
  tseslint.configs.recommended,
  eslintConfigPrettier,
  {
    files: ["**/*.ts"],
    rules: {
      "@typescript-eslint/no-explicit-any": "off",
    },
  },
  {
    files: ["**/*.cjs"],
    languageOptions: {
      sourceType: "commonjs",
    },
  },
]);
```

### Vitest Coverage Config
```typescript
// Source: vitest.dev/config/coverage
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    coverage: {
      provider: "v8",
      reporter: ["text", "json-summary"],
      include: ["src/**/*.ts", "index.ts"],
      exclude: ["src/mcp-schema-server.cjs"],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80,
      },
    },
  },
});
```

### Package.json Script Additions
```json
{
  "scripts": {
    "test": "vitest run --reporter=verbose",
    "test:coverage": "vitest run --coverage",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format:check": "prettier --check ."
  }
}
```

### Version Validation (Shell Script for CI)
```bash
# Source: Standard CI pattern for tag-to-package.json validation
TAG_VERSION="${GITHUB_REF#refs/tags/v}"
PKG_VERSION=$(node -p "require('./package.json').version")
if [ "$TAG_VERSION" != "$PKG_VERSION" ]; then
  echo "::error::Tag version ($TAG_VERSION) does not match package.json version ($PKG_VERSION)"
  exit 1
fi
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `.eslintrc.json` | `eslint.config.mjs` (flat config) | ESLint 10 (Feb 2026) | eslintrc completely removed in v10; flat config is mandatory |
| `eslint-plugin-prettier` | Separate Prettier + `eslint-config-prettier` | Community consensus 2024+ | Faster CI, cleaner separation of concerns |
| istanbul coverage | v8 coverage | vitest 3.2.0 (2025) | v8 now produces identical reports to istanbul, with faster execution |
| `actions/setup-node@v4` | `actions/setup-node@v6` | Oct 2024 | v6 supports automatic npm caching, Node 24 |
| `actions/checkout@v4` | `actions/checkout@v6` | Nov 2024 | v6 stores credentials separately for security |

**Deprecated/outdated:**
- `.eslintrc.*` files: Completely ignored by ESLint 10
- `eslint-plugin-prettier`: Still works but community consensus is "don't run Prettier inside ESLint"
- `actions/create-release` (GitHub official): Archived, replaced by `softprops/action-gh-release` or `gh release create`

## Test Gap Analysis

### Current Coverage (248 tests across 9 files)

All 9 source modules have corresponding test files:

| Source Module | Test File | Approximate Coverage |
|---------------|-----------|---------------------|
| `stream-parser.ts` | `stream-parser.test.ts` | HIGH -- all parse paths tested |
| `control-handler.ts` | `control-handler.test.ts` | HIGH -- allow/deny/malformed tested |
| `tool-mapping.ts` | `tool-mapping.test.ts` | HIGH -- bidirectional mapping, MCP prefix, custom tool detection |
| `thinking-config.ts` | `thinking-config.test.ts` | HIGH -- effort mapping for all levels, Opus detection |
| `mcp-config.ts` | `mcp-config.test.ts` | HIGH -- custom tool filtering, config file generation |
| `event-bridge.ts` | `event-bridge.test.ts` | HIGH -- all event types, tool use, thinking, signatures |
| `prompt-builder.ts` | `prompt-builder.test.ts` | HIGH -- history building, images, custom tools, system prompt |
| `process-manager.ts` | `process-manager.test.ts` | MEDIUM -- spawn, kill, cleanup; some edge cases may be untestable |
| `provider.ts` | `provider.test.ts` | MEDIUM -- orchestration, break-early, timeouts; complex async flows |

### Potential Test Gaps to Investigate

The planner should direct the implementer to:

1. **Run `vitest run --coverage` first** to get exact line/branch numbers before writing any new tests
2. **Focus on branch coverage gaps** -- these are where meaningful bugs hide:
   - `provider.ts`: abort signal paths (already aborted, abort during stream), hasPendingCustomToolResult edge cases, effectiveReason override logic
   - `process-manager.ts`: validateCliPresence/validateCliAuth error paths (these call `execSync` which is hard to mock cleanly)
   - `prompt-builder.ts`: edge cases in `resolveAgentsMdPath` (walk-up resolution), `sanitizeAgentsContent` regex edge cases
   - `event-bridge.ts`: unknown event types, missing index lookups, delta without matching block
3. **Do NOT add tests for**:
   - Coverage number padding (trivial getters/constructors)
   - Process exit handlers (untestable without real subprocess)
   - `console.warn`/`console.error` output (already covered implicitly)

### Coverage Threshold Recommendation

Based on the test file sizes and module complexity:
- The project likely has 75-85% line coverage already (248 tests across 9 modules is thorough)
- **Recommended initial threshold: 80% lines, 80% functions, 70% branches, 80% statements**
- After gap analysis, the implementer should adjust upward based on actual baseline
- Use `thresholds` in vitest config to enforce -- CI fails if coverage drops

## Open Questions

1. **Package lockfile existence**
   - What we know: No `package-lock.json` visible in the project root
   - What's unclear: Whether pi extension projects conventionally use lockfiles
   - Recommendation: Generate `package-lock.json` with `npm install` and commit it. Required for `npm ci` in CI. If lockfile is undesirable, use `npm install` in CI instead.

2. **Exact coverage baseline**
   - What we know: 248 tests pass; coverage provider not yet installed
   - What's unclear: Exact line/branch/function percentages
   - Recommendation: First task in the plan should install `@vitest/coverage-v8` and run coverage to establish baseline before setting thresholds

3. **ESLint rule overrides beyond `no-explicit-any`**
   - What we know: `any` is used extensively and intentionally
   - What's unclear: Whether other recommended rules conflict with existing code patterns
   - Recommendation: Run `npx eslint .` after config creation; address any other conflicts with targeted rule overrides

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | vitest 3.x |
| Config file | `vitest.config.ts` (exists, needs coverage settings added) |
| Quick run command | `npx vitest run --reporter=verbose` |
| Full suite command | `npx vitest run --coverage --reporter=verbose` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RLSE-01 | Unit tests cover NDJSON parsing, event bridging, tool mapping, control protocol, prompt building | unit | `npx vitest run --reporter=verbose` | Yes -- 9 test files, 248 tests |
| RLSE-01 (gap) | Coverage gaps filled after baseline analysis | unit | `npx vitest run --coverage` | Partially -- gaps TBD after baseline |
| RLSE-02 | CI workflow runs lint + typecheck + test on 3 OS | smoke | Push to PR branch, verify GitHub Actions pass | No -- `.github/workflows/ci.yml` is Wave 0 |
| RLSE-03 | Tagged release publishes to npm + creates GitHub Release | manual-only | Push `v*` tag to verify; cannot fully automate without npm credentials | No -- `.github/workflows/publish.yml` is Wave 0 |
| RLSE-04 | package.json has pi-package keyword, pi.extensions, peer deps | unit | `node -e "const p=require('./package.json'); assert(p.keywords.includes('pi-package'))"` | No explicit test, but verifiable by inspection |

### Sampling Rate
- **Per task commit:** `npx vitest run --reporter=verbose`
- **Per wave merge:** `npx vitest run --coverage --reporter=verbose`
- **Phase gate:** Full suite green + lint/typecheck pass + coverage above threshold before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `.github/workflows/ci.yml` -- CI workflow file (RLSE-02)
- [ ] `.github/workflows/publish.yml` -- Publish workflow file (RLSE-03)
- [ ] `eslint.config.mjs` -- ESLint flat config
- [ ] `@vitest/coverage-v8` install -- coverage provider
- [ ] `eslint`, `@eslint/js`, `typescript-eslint`, `prettier`, `eslint-config-prettier` install -- lint tooling
- [ ] Coverage thresholds in `vitest.config.ts` -- after baseline measurement
- [ ] `coverage/` added to `.gitignore` -- prevent committing coverage artifacts

## Sources

### Primary (HIGH confidence)
- [typescript-eslint getting-started](https://typescript-eslint.io/getting-started/) -- ESLint flat config setup with TypeScript
- [typescript-eslint dependency-versions](https://typescript-eslint.io/users/dependency-versions/) -- Confirms ESLint 10 support (^8.57.0 || ^9.0.0 || ^10.0.0)
- [ESLint v10.0.0 release blog](https://eslint.org/blog/2026/02/eslint-v10.0.0-released/) -- Breaking changes, Node.js requirements
- [ESLint configuration files docs](https://eslint.org/docs/latest/use/configure/configuration-files) -- Flat config format, defineConfig, globalIgnores
- [Vitest coverage guide](https://vitest.dev/guide/coverage.html) -- @vitest/coverage-v8 setup
- [Vitest coverage config](https://vitest.dev/config/coverage) -- Thresholds, reporters, include/exclude
- [eslint-config-prettier GitHub](https://github.com/prettier/eslint-config-prettier) -- Flat config import path (`eslint-config-prettier/flat`)
- [GitHub Actions publishing Node.js packages](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages) -- Official npm publish workflow
- [actions/setup-node releases](https://github.com/actions/setup-node/releases) -- v6.3.0 is latest
- [actions/checkout releases](https://github.com/actions/checkout/releases) -- v6.0.2 is latest
- [Prettier CLI docs](https://prettier.io/docs/cli) -- `--check` flag for CI

### Secondary (MEDIUM confidence)
- [softprops/action-gh-release](https://github.com/softprops/action-gh-release) -- GitHub Release creation action
- [Node.js releases](https://nodejs.org/en/about/previous-releases) -- LTS schedule (20, 22, 24 active in March 2026)
- [npm package versions](https://www.npmjs.com/package/eslint) -- ESLint 10.0.3 latest

### Tertiary (LOW confidence)
- None -- all findings verified with primary or secondary sources

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all versions verified via official docs and npm
- Architecture: HIGH -- patterns follow official documentation examples
- Pitfalls: HIGH -- based on direct code analysis of all 11 source files and known ESLint 10 changes
- Test gap analysis: MEDIUM -- based on code review, not actual coverage numbers (requires running coverage tool)

**Research date:** 2026-03-15
**Valid until:** 2026-04-15 (30 days -- stable tooling, unlikely to change significantly)
