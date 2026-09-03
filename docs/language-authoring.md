# Language authoring guide

This guide describes the current, scope-resolution-first path for adding a language to GitNexus. It intentionally focuses on the smallest reviewable vertical slice: detection → parsing/captures → scope model → resolution → tests.

> This is one deliverable from #947. It does not attempt to rewrite the broader architecture documentation in the same change.

## 1. Start from the language contract

A language integration should produce stable graph facts without inventing language-specific resolution shortcuts. Before coding, identify:

- file extensions and any shebang/detection rules;
- the tree-sitter grammar and its runtime/ABI constraints;
- definition kinds that map to GitNexus node labels;
- lexical scopes and declaration/binding behavior;
- import forms and whether they execute where written;
- call/reference forms, including receiver-bound and constructor-like sites;
- type evidence that is available statically;
- external imports that must fail closed instead of suffix-matching local files.

Use an already shipped language with similar semantics as the reference implementation rather than starting from a blank provider.

## 2. Register detection and grammar loading

Keep language recognition deterministic and cheap. Add the language to the shared language/detection layer, then wire its parser/grammar through the existing parser loader and worker registration paths.

If the grammar is optional, preserve the repository's optional-grammar contract: unsupported platforms may skip cleanly, while CI must have at least one required lane that proves the grammar is actually present and loadable. Do not let an optional grammar produce a permanently green-by-skip test suite.

## 3. Define structural captures

Add the language's tree-sitter query/capture vocabulary under the ingestion language area. Capture syntax, not guesses. At minimum, cover the definitions and scopes required by the first supported resolution cases.

Useful invariants:

- one source construct should not mint duplicate definitions through overlapping capture rules;
- declaration ranges should place a definition in the scope that owns it;
- names used only as assignment targets must not accidentally become declarations;
- capture changes are parse-cache-sensitive, so follow the repository's cache-version/fingerprint rules when required.

Add focused capture/extractor tests for grammar facts that are easy to misremember (field names, anonymous nodes, wrapper shapes, parameter layout).

## 4. Implement imports as scope facts

Model imports through the scope-resolution contract rather than creating a parallel language-specific DAG. The import interpreter should distinguish named, namespace, wildcard/re-export and side-effect forms when the language has them.

Import resolution must fail closed. In particular, a standard-library/package import that cannot resolve inside the repository should return the provider's explicit external/unresolved result instead of falling through to a suffix match that could connect an unrelated local file.

When package manifests or build files affect resolution, keep that configuration in the language provider/import configuration layer and make ambiguity explicit.

## 5. Build and finalize the scope model

The scope pipeline is the source of truth for name/binding lookup. Feed declarations, bindings, imports and type evidence into the shared scope model so downstream passes can reuse the same facts.

Prefer the public scope-resolution primitives and provider hooks over direct graph scans. Language-specific hooks should be narrow and opt-in so existing languages remain byte-for-byte unchanged unless they enable the new behavior.

When a hook runs after scope finalization, document whether it augments definition bindings, type bindings, namespace exports, or reference evidence. Those channels are not interchangeable.

## 6. Resolve references and calls

Start with the highest-confidence cases:

1. same-scope direct calls/references;
2. imported-name calls;
3. namespace-qualified calls;
4. receiver-bound calls backed by explicit type evidence;
5. constructor/construction forms if the language has a reliable static representation.

Do not add a workspace-wide name fallback merely to improve recall. A missing edge is preferable to a confidently wrong cross-file edge.

If a language needs behavior beyond the shared defaults, add a provider flag/hook that is off for every existing provider. Tests should prove the default-off path remains unchanged.

## 7. Add a minimal resolver fixture

Create a small fixture repository that contains both the positive case and a decoy that would make an unsafe implementation choose the wrong target. A useful first fixture normally covers:

- one local definition and call;
- one cross-file import and call;
- one external import beside a same-named local decoy;
- one receiver/type-driven call when applicable;
- one negative/ambiguous case that must resolve to nothing.

Prefer assertions on exact node/edge identity and reason/provenance, not only aggregate counts.

## 8. Join the cross-language conformance gates

Search the test and benchmark directories for tables that enumerate every registered language. New languages commonly need entries in parser/ABI smoke tests, external-import conformance, callable-flow coverage, import-target reuse/fingerprint checks, capture fingerprints and structural-pair/schema coverage.

A language that passes its own fixture but is absent from repository-wide gates is not finished.

## 9. Validate in layers

Run the narrowest checks first, then broaden:

```bash
cd gitnexus
npx vitest run <language unit tests>
npx vitest run <language resolver/integration tests>
npx tsc --noEmit
npm test
```

Also run any capture/import-target fingerprint gates touched by the provider. If the grammar is optional, run one CI-equivalent path with the grammar explicitly required so skips cannot mask a broken integration.

For a non-trivial resolver change, index a real repository in that language and compare representative graph facts against the fixture assumptions. Record limitations rather than hiding unsupported constructs behind heuristic edges.

## 10. Shipping checklist

Before opening the PR, confirm:

- [ ] language detection and grammar loading are registered;
- [ ] optional grammar behavior and required CI coverage are explicit;
- [ ] captures/extractors have focused grammar-shape tests;
- [ ] imports fail closed for external/ambiguous targets;
- [ ] scope bindings and type evidence use shared contracts;
- [ ] resolver behavior has positive and adversarial/decoy fixtures;
- [ ] provider-specific behavior is opt-in and does not alter other languages;
- [ ] cross-language conformance tables/gates include the new language;
- [ ] parse-cache/fingerprint/schema bumps were considered where applicable;
- [ ] TypeScript compilation and relevant tests/fingerprints were actually run;
- [ ] real-repository smoke evidence exists for a substantial integration;
- [ ] unsupported syntax is documented as a limitation instead of guessed.

## Review strategy

Keep the first PR as small as the language permits. Review shared-code changes first, because they have the widest blast radius; then review provider registration, captures, resolver hooks, and finally the fixtures. If a shared abstraction must change, make the default behavior preserve all existing providers and pin that with tests.

The most valuable language integration is not the one with the highest raw edge count. It is the one whose edges remain trustworthy when the repository contains ambiguous names, external packages, generated code and unfamiliar framework conventions.