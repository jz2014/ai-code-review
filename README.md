# Perspective-Driven Code Review

> Extracted from 80+ real AI-assisted code review sessions. Tested against Claude Code, Codex, and manual review — consistently catches bugs that single-perspective AI reviews miss.

**Core principle:** Review quality = number of independent perspectives, not number of rounds. Switching perspectives finds more bugs than repeating the same one.

**Why AI reviews miss bugs:** AI code reviewers tend to lock into one perspective per pass — usually correctness. They'll miss impact radius (changed a return value but didn't grep all callers), adversarial angles (can this input be injected?), and cross-module consistency. This methodology forces systematic perspective rotation.

## How to Use

- **With AI tools:** Add this file to your AI coding assistant's context (e.g., reference it in CLAUDE.md, .cursorrules, Codex instructions, or equivalent). The AI will follow the structure automatically.
- **As a human reviewer:** Use the checklists as a review guide — run each perspective as a separate mental pass.
- **Hybrid (recommended):** Let AI do the first review, then run this methodology on top. The perspective switching catches what the AI's default review missed.

---

## Severity

| Level | Definition | Action |
|-------|-----------|--------|
| **Critical** | Security vuln, data loss, runtime crash | Must fix |
| **Important** | Logic error, type unsafety, missing error handling | Should fix |
| **Suggestion** | Code quality, naming, refactor | Optional |

## Pre-Implementation (Plan Phase)

1. `grep -ri` all references before any rename — list each in plan
2. Read full implementation of called functions (return values, side effects)
3. Check all interface fields when creating structs, not just required ones
4. Review plan for injection, resource leaks, trust boundaries
5. Consider simpler/safer alternatives

---

## Universal Perspectives (always run, 4 total)

### 1. Correctness

- [ ] Happy path end-to-end walkthrough
- [ ] Error/exception path walkthrough (every throw/early return/catch)
- [ ] Silent failure: search empty `catch {}`, swallowed errors, no-log failure paths
- [ ] Search `!.` — confirm null guarantee for each non-null assertion
- [ ] Search `as Type` — confirm runtime validation for each type assertion
- [ ] Boundary values: empty array, empty string, undefined

### 2. Impact Radius

Will existing code break?

- [ ] Grep modified function/variable/type names, check every caller
- [ ] Callers compatible with new return values / throw behavior?
- [ ] Import paths correct?
- [ ] Type signature changes backward-compatible?

### 3. Consistency

Contradicts rest of system?

- [ ] Naming style matches similar functions
- [ ] Output format (JSON/YAML/text) matches other endpoints
- [ ] Error handling pattern matches similar scenarios
- [ ] User-visible terms unified — grep concept words (not just command names)

### 4. Completeness

What's missing?

- [ ] Tests cover new/changed functionality
- [ ] Test quality: assertions verify correct results, not just "function ran"
- [ ] README / API doc / CHANGELOG needs update?
- [ ] Project tracking (issues, roadmap) needs update?
- [ ] Interface/type needs new fields?

---

## Domain Perspectives (triggered by change characteristics)

### Trigger Checklist

- [ ] Receives external input? (HTTP body / CLI args / file content / env vars / remote API) → **Security**
- [ ] Produces user-visible output? (console / HTTP response / HTML / throw message) → **UX**
- [ ] Changes public API? (function sig / HTTP endpoint / CLI command / npm exports) → **Compatibility**
- [ ] Changes persistent data? (DB schema / file format / storage) → **Data Integrity**
- [ ] Changes deployment/infra? (DNS / CDN / CI / Docker) → **Ops**
- [ ] Adds/upgrades/removes dependency? → **Supply Chain**
- [ ] Hot path? (HTTP handler / DB query / high-frequency loop) → **Performance**

### Security (trust boundary)

- [ ] POST endpoints have body size limit (413 on exceed)
- [ ] User input → SQL: parameterized queries
- [ ] User input → HTML: escapeHtml
- [ ] User input → shell: escaped
- [ ] Remote data: type guard before use, never blind type assertion
- [ ] IDs/tokens/nonces: cryptographically random
- [ ] LIKE queries: escape `%_\`, add ESCAPE clause

### UX

- [ ] Error messages clear and helpful
- [ ] HTTP status codes correct (413 not 500)
- [ ] JSON and text output formats both covered
- [ ] CLI help text accurate

### Compatibility

- [ ] Existing consumers (SDK users, CLI scripts) affected?
- [ ] Deprecation transition needed?
- [ ] Version bump needed?

### Data Integrity

- [ ] Migration script handles existing data
- [ ] Rollback safe?
- [ ] Old/new format coexistence handled

### Ops

- [ ] Dev / staging / prod differences
- [ ] Reversible (can rollback)?
- [ ] Cost impact

### Supply Chain

- [ ] Dependency CHANGELOG: breaking changes?
- [ ] Security advisories
- [ ] Lock file updated

### Performance

- [ ] N+1 queries, unnecessary loops, redundant computation on hot paths
- [ ] Algorithm complexity reasonable at scale
- [ ] Cache strategy appropriate (TTL, invalidation)
- [ ] Resource allocation (memory, connection pools) reasonable

---

## Round Rules

**Formula:** `rounds = universal_handling + triggered_domains + 1 (fix verification)`

### Universal Perspective Scaling

| Risk | Characteristics | Universal Handling |
|------|----------------|-------------------|
| Low | Single file, no trust boundary, internal logic | All 4 in 1 round |
| Medium | Multi-file, or trust boundary, or public API change | 2 rounds (correctness+impact / consistency+completeness) |
| High | Cross-module, conceptual change, security-related | 4 separate rounds |

### Per-Round Rules

- Each round is a **fresh start**, not "re-check previous findings"
- Each round **must switch perspective**
- Fix verification is always the **last round**
- If fix verification finds issues → fix → add another verification round (recurse until clean)

---

## Fix Verification Round

1. Trace each fix's impact chain: throw → catch → status code / user message
2. Quick scan files/paths not covered in earlier rounds
3. New issues found → fix → recurse

---

## Automated Security Validation (final step)

After fix verification is clean, run your static analysis / security scanning tools (e.g., Semgrep, CodeQL, Snyk, or your AI tool's security review mode).

**Run AFTER manual/perspective review** to avoid anchoring bias. If tools find missed issues → fix → re-run fix verification → add to common misses table below.

---

## Scenario Walkthrough (for Correctness perspective)

1. List all user scenarios (including edge cases and error cases)
2. Walk code line-by-line per scenario: track variable values + side effects + early return behavior
3. Check output consistency: duplicates/contradictions? JSON and text covered?
4. Cross-file compatibility: grep concept words, distinguish user-visible vs internal

---

## Common Misses

Accumulated from real review sessions — each entry represents a bug that was missed by at least one AI review pass.

| Miss | Prevention |
|------|-----------|
| XSS/injection | Check escapeHtml on every HTML `${}` |
| Resource leak | Confirm cleanup on all paths, try/finally for readline/fd |
| Silent failure | Search empty catch, swallowed errors, no-log failures |
| Duplicate output | Read callee fully before writing caller |
| Missed text refs | Grep concept words, not just command names |
| Missing fields | Open interface, check every field |
| Insufficient return | Design functions for what callers need |
| `!` assertion | Search `!.`, confirm each has null check |
| Unvalidated remote data | Trust boundary → type guard, never blind type assertion |
| SDK error message | Thrown message is user-visible text |
| Unbounded body DoS | POST endpoints need body limit → 413 |
| LIKE wildcard injection | Escape `%_\`, add ESCAPE clause |
| Weak randomness | Use cryptographically random values for IDs/tokens/nonces |
| Fix-induced regression | Trace error propagation: throw → catch → status code |
| Error path not walked | Correctness must explicitly cover error paths |
| Fake test coverage | Assertions must verify behavior, not just execution |
| Hardcoded policy values | Limits, thresholds, sizes should be configurable with named defaults |

---

## Contributing

Found a common miss pattern? PRs welcome — add it to the table with a concrete prevention action.

## License

MIT
