# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

The project uses pnpm workspaces. Key commands:

- `pnpm build` - Build all packages (runs recursive build command via `zshy`)
- `pnpm vitest run` - Run all tests with Vitest
- `pnpm vitest run <path>` - Run specific test file (e.g., `packages/zod/src/v4/classic/tests/string.test.ts`)
- `pnpm vitest run <path> -t "<pattern>"` - Run specific test(s) within a file (e.g., `-t "MAC"`)
- `pnpm vitest run --update` - Update all test snapshots
- `pnpm vitest run <path> --update` - Update snapshots for specific test file
- `pnpm test:watch` - Run tests in watch mode
- `pnpm vitest run --coverage` - Run tests with coverage report
- `pnpm dev` - Execute code with tsx under `@zod/source` conditions
- `pnpm dev <file>` - Execute `<file>` with tsx & proper resolution conditions. Usually use for `play.ts`.
- `pnpm dev:play` - Quick alias to run play.ts for experimentation
- `pnpm lint` - Run biome linter with auto-fix
- `pnpm format` - Format code with biome
- `pnpm fix` - Run both format and lint
- `pnpm check:circular` - Check for circular dependencies via madge (v4/core excluded)
- `pnpm check:semver` - Validate version consistency across packages

## Rules

- Node.js v24+ required (use nvm if needed); pnpm v10.12.1
- ES modules are used throughout (`"type": "module"`)
- All tests must be written in TypeScript - never use JavaScript
- Use `play.ts` for quick experimentation; use proper tests for all permanent test cases
- Features without tests are incomplete - every new feature or bug fix needs test coverage
- Don't skip tests due to type issues - fix the types instead
- Test both success and failure cases with edge cases
- No log statements (`console.log`, `debugger`) in tests or production code
- Ask before generating new files
- Use `util.defineLazy()` for computed properties to avoid circular dependencies
- Performance is critical - parameter reassignment is allowed for optimization
- ALWAYS use the `gh` CLI to fetch GitHub information (issues, PRs, etc.) instead of relying on web search or assumptions

## Repository Overview

Zod is a TypeScript-first schema declaration and validation library. This is a monorepo (pnpm workspaces) containing:

- **`packages/zod`** — The main `zod` npm package (currently v4.x)
- **`packages/bench`** — Benchmarks comparing Zod against other schema libraries
- **`packages/integration`** — Integration tests for published package shape
- **`packages/treeshake`** — Tree-shaking verification tests
- **`packages/resolution`** — Module resolution tests
- **`packages/tsc`** — TypeScript compilation tests
- **`packages/docs`** — Documentation site source

## Source Layout (`packages/zod/src/`)

```
src/
├── index.ts              ← Main entry: re-exports v4/classic/external
├── v4/
│   ├── core/             ← Shared internals (framework-agnostic base)
│   │   ├── core.ts       ← $constructor, $ZodTrait, $brand, NEVER
│   │   ├── schemas.ts    ← $ZodType, $ZodTypeDef, ParseContext, ParsePayload
│   │   ├── checks.ts     ← $ZodCheck base + all built-in check implementations
│   │   ├── errors.ts     ← $ZodIssue subtypes, $ZodError, $ZodErrorMap
│   │   ├── parse.ts      ← parse/safeParse/parseAsync core logic
│   │   ├── api.ts        ← Public factory function helpers (TypeParams, Params)
│   │   ├── util.ts       ← Shared TypeScript utilities, defineLazy, etc.
│   │   ├── regexes.ts    ← Compiled validation regexes
│   │   ├── registries.ts ← $ZodRegistry, globalRegistry
│   │   ├── config.ts     ← Global config ($ZodConfig, config())
│   │   ├── doc.ts        ← JIT code generation (Doc class, compile())
│   │   ├── to-json-schema.ts        ← JSON Schema conversion types/interfaces
│   │   ├── json-schema-generator.ts ← JSONSchemaGenerator class
│   │   ├── json-schema-processors.ts← toJSONSchema() implementation
│   │   ├── json-schema.ts           ← JSON Schema type definitions
│   │   ├── standard-schema.ts       ← StandardSchemaV1 interface
│   │   ├── versions.ts   ← { major, minor, patch }
│   │   ├── zsf.ts        ← Zod Schema Format (ZSF) type definitions
│   │   └── index.ts      ← Re-exports all core
│   ├── classic/          ← Main "zod" export — full-featured API
│   │   ├── schemas.ts    ← ZodType, ZodString, ZodNumber, etc. (extends core)
│   │   ├── checks.ts     ← Re-exports checks from core with friendly names
│   │   ├── errors.ts     ← ZodError (extends $ZodError), ZodIssue aliases
│   │   ├── parse.ts      ← parse/safeParse (bound to ZodRealError)
│   │   ├── compat.ts     ← Zod v3 compatibility shims (@deprecated)
│   │   ├── iso.ts        ← ZodISODateTime, ZodISODate, ZodISOTime, ZodISODuration
│   │   ├── coerce.ts     ← ZodCoercedString, ZodCoercedNumber, etc.
│   │   ├── from-json-schema.ts ← fromJSONSchema() reverse converter
│   │   ├── external.ts   ← Combines all classic exports + sets EN locale default
│   │   └── index.ts      ← { z } namespace export
│   ├── mini/             ← Lightweight "zod/mini" export (no method chaining)
│   │   ├── schemas.ts    ← ZodMini* types (stripped-down API)
│   │   ├── checks.ts     ← Mini check exports
│   │   ├── parse.ts      ← Mini parse functions
│   │   ├── iso.ts        ← ZodMiniISO* types
│   │   ├── coerce.ts     ← Mini coerce
│   │   ├── external.ts   ← Mini external exports
│   │   └── index.ts      ← { z } namespace export
│   ├── locales/          ← i18n error message maps (50+ languages)
│   │   ├── en.ts         ← English (default locale, auto-configured in classic)
│   │   ├── de.ts, fr.ts, ja.ts, zh-CN.ts … (many more)
│   │   └── index.ts      ← Re-exports all locales
│   └── index.ts          ← Re-exports classic as default v4 entry
├── mini/                 ← Alias entry for "zod/mini"
├── locales/              ← Alias entry for "zod/locales"
└── v3/                   ← Zod v3 compatibility shim (not the v3 source)
```

## Architecture Patterns

### `$constructor` / Trait System

Schemas and checks are created via `$constructor()` in `v4/core/core.ts`. This is not a normal class — it uses a traits system where multiple constructors can "compose" into a single instance via `inst._zod.traits`:

```typescript
export const $ZodString = $constructor("$ZodString", (inst, def) => {
  $ZodType.init(inst, def);
  // ... add string-specific methods/properties
});
```

- Each instance carries `inst._zod.traits: Set<string>` tracking which constructors have initialized it
- `instanceof` checks use `Symbol.hasInstance` to match by trait name
- The `$constructor` function returns a function that acts as a class constructor

### Schema Internals (`_zod`)

Every schema has a non-enumerable `_zod` property containing:
- `_zod.def` — the schema definition object (serializable config)
- `_zod.run(payload, ctx)` — the parse function (may be JIT-compiled via `Doc`)
- `_zod.traits` — Set of trait names
- `_zod.constr` — reference to the constructor
- `_zod.deferred` — deferred initialization callbacks

### Circular Dependency Prevention

Use `util.defineLazy(object, key, getter)` whenever a property on a schema's internals needs to reference another schema or value that might create a circular import. The getter is deferred until first access, with cycle detection to return `undefined` instead of hanging.

### JIT Code Generation

The `Doc` class (`v4/core/doc.ts`) generates JavaScript code strings at schema initialization time. `doc.compile()` uses `new Function(...)` to create optimized parse functions. Use `jitless: true` in `ParseContext` to skip this (e.g., in environments where `eval` is restricted).

### Tree-Shaking Annotations

All public factory functions in `v4/core/api.ts` use `// @__NO_SIDE_EFFECTS__` comments. Schema constructors use `/* @__PURE__ */` on calls to enable bundler dead-code elimination. Both annotations must be preserved when adding or refactoring factory functions.

## Package Exports and Module Resolution

The package has a custom `@zod/source` condition used in development/testing to resolve imports directly to TypeScript source files without building first. This is configured in `vitest.config.ts`, `tsconfig.json`, and package `exports`:

```json
".": {
  "@zod/source": "./src/index.ts",  // dev/test: uses TS source
  "types": "./index.d.cts",          // consumers: built types
  "import": "./index.js",            // consumers: ESM
  "require": "./index.cjs"           // consumers: CJS
}
```

The `pnpm dev` command passes `--conditions @zod/source` to `tsx` to enable this.

Key sub-path exports: `.` (classic), `./mini`, `./v4`, `./v4/mini`, `./v4/core`, `./v4/locales`, `./locales`, `./v3`.

All import paths within `src/` must end in `.js` (even though they resolve to `.ts` at runtime via the `@zod/source` condition). This is required by Node.js ESM and `moduleResolution: nodenext`.

## Error System

- `$ZodIssue` subtypes (in `v4/core/errors.ts`): `invalid_type`, `too_big`, `too_small`, `invalid_format`, `not_multiple_of`, `unrecognized_keys`, `invalid_union`, `invalid_key`, `invalid_element`, `invalid_value`, `custom`
- `$ZodErrorMap<T>` — a function `(issue: T) => string | undefined` for custom messages
- `$ZodError` — base error class (array of finalized issues)
- `ZodError` (classic) — extends `$ZodError` with deprecated v3-compat methods (`.format()`, `.flatten()`)
- Locales are `$ZodErrorMap` implementations; the EN locale is automatically applied in `v4/classic/external.ts` via `config(en())`
- Use `treeifyError`, `prettifyError`, `formatError`, `flattenError` utility functions instead of the deprecated methods on `ZodError`

## Registries

`$ZodRegistry` (`v4/core/registries.ts`) is a `WeakMap`-based metadata store for schemas. Used for JSON Schema generation, documentation metadata, and custom tooling. The `globalRegistry` is the default. Schemas with an `id` property in their metadata are tracked in an `_idmap` for stable `$ref` generation.

## JSON Schema

- `toJSONSchema(schema, params?)` — converts a Zod schema to JSON Schema (Draft 2020-12 by default, also supports Draft 7, Draft 4, OpenAPI 3.0)
- `fromJSONSchema(jsonSchema)` (classic only) — reverse-converts a JSON Schema to a Zod schema
- `JSONSchemaGenerator` — low-level class for custom conversion pipelines with custom `processors`
- `toJSONSchema` on `$ZodRegistry` — bulk-converts all registered schemas with `$defs`
- The `io` param (`"input"` | `"output"`) controls whether input or output type shape is generated (relevant for transforms and defaults)

## Formatting / Linting

Biome is used for both formatting and linting. Key settings (from `biome.jsonc`):
- **Indent**: spaces (not tabs)
- **Line width**: 120 characters
- **Trailing commas**: `es5` (JS/TS), none (JSON)
- `noExplicitAny` is **off** — `any` is used freely in internal code
- `noParameterAssign` is **off** — allowed for performance-critical coercion
- `noUnusedImports` is an **error** (no auto-fix — must be fixed manually)

## Testing Conventions

- All tests live in `packages/zod/src/v4/classic/tests/` or `packages/zod/src/v4/mini/tests/`
- Tests use `vitest` (`import { expect, test, describe } from "vitest"`)
- Type-level assertions use `util.assertEqual<A, B>(true)` / `util.assertNotEqual` helpers from `v4/core/util.ts`
- `vitest.config.ts` enables `typecheck` so type errors in `.test.ts` files are caught by `tsc`
- `scripts/fail-on-console.ts` is registered as a setup file — any `console.*` call in tests will fail the suite
- Snapshot tests: use `--update` flag to refresh snapshots
- Tests are isolated (`isolate: true`) and run without watch by default

## Build System

- Build tool: `zshy` (configured per-package in `package.json`'s `"zshy"` field)
- Output format: dual CJS + ESM
- Post-build: `write-stub-package-jsons.ts` generates `package.json` stubs for sub-path exports; `biome format` is run on output
- TypeScript: `~5.5.4`, strict mode, `moduleResolution: nodenext`, `module: nodenext`
- `tsconfig.build.json` (per package) is used for the build; root `tsconfig.json` is for IDE/typecheck

## v3 Compatibility

`packages/zod/src/v4/classic/compat.ts` exports deprecated v3 APIs with `@deprecated` JSDoc tags. These shims exist for gradual migration — prefer v4 APIs in all new code. The `zod/v3` sub-path re-exports the Zod v3 source as shipped inside this package for users who need to pin to v3 behavior.

## Key Type Utilities (`v4/core/util.ts`)

- `util.defineLazy(obj, key, getter)` — lazy computed property with cycle detection
- `util.assertEqual<A, B>(true)` — compile-time type equality assertion
- `util.assertNotEqual<A, B>(false)` — compile-time type inequality assertion
- `util.assertNever(x)` — exhaustiveness check
- `util.Flatten<T>`, `util.Prettify<T>` — type display helpers
- `util.MaybeAsync<T>` — `T | Promise<T>`
- `util.SafeParseResult<T>` — discriminated union of success/failure
- `util.normalizeParams(params)` — converts string shorthand `"message"` to `{ message }` object form
- `util.cached(getter)` — memoizes a getter (used for expensive computed properties)
