# Thunderbird for Android - Claude Development Guide

## Critical Pre-Flight Checks

**Before ANY commit:**
```bash
./gradlew spotlessApply   # Auto-format code
./gradlew check           # Tests + lint + detekt + spotless
```

**Branch rules:**
- Create branches from latest `main`: `git checkout -b claude/<issue-number>`
- NEVER commit to `main` directly
- Push with: `git push -u origin claude/<issue-number>`

## Authoritative Documentation

Always consult these docs for detailed guidance. They are the source of truth:

| Topic | Location |
|-------|----------|
| Full contributing guide | `docs/CONTRIBUTING.md` |
| Contribution workflow | `docs/contributing/contribution-workflow.md` |
| Development setup | `docs/contributing/development-environment.md` |
| Architecture | `docs/architecture/README.md` |
| Code quality | `docs/contributing/code-quality-guide.md` |
| Testing | `docs/contributing/testing-guide.md` |
| Git commits | `docs/contributing/git-commit-guide.md` |
| Code review | `docs/contributing/code-review-guide.md` |
| String management | `docs/contributing/managing-strings.md` |

Online docs: https://thunderbird.github.io/thunderbird-android/docs/latest/

## Project-Specific Rules

These are non-obvious conventions specific to this project:

### Module Structure
```
feature:foo:api   → Public interfaces (depend on THIS)
feature:foo:impl  → Implementation (NEVER depend externally)
core:*            → Foundational utilities
library:*         → Specific implementations
legacy:*          → DO NOT add new code here
```

### Testing Conventions
- Name test subject: `testSubject` (NOT "sut")
- Use fakes over mocks (see `docs/contributing/testing-guide.md`)
- Use AssertK for assertions
- AAA pattern with comments: `// Arrange`, `// Act`, `// Assert`
- JVM tests: backticks for names `` `should do X when Y` ``
- Android tests: camelCase `shouldDoXWhenY`

### Error Handling
Use `Outcome<T, E>` pattern instead of exceptions for domain errors.

### Dependency Injection
- Use Koin with constructor injection
- No static singletons, service locators, or field injection

### Internationalization
- ONLY modify English strings: `res/values/strings.xml`
- NEVER edit translation files: `res/values-*/strings.xml`
- Weblate handles translations after merge

### Commit Format (Conventional Commits)
```
<type>(<scope>): <description>

<body>

Fixes #<issue>
```
Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

## Quick Commands

```bash
./gradlew spotlessApply          # Format code
./gradlew check                  # All checks
./gradlew test                   # Unit tests
./gradlew :module-name:test      # Module tests
./gradlew lint                   # Android lint
./gradlew detekt                 # Static analysis
./gradlew connectedAndroidTest   # Instrumentation tests
```

## PR Checklist

- [ ] Branch from latest `main`
- [ ] `./gradlew check` passes
- [ ] Tests added for changes
- [ ] No code in `legacy:*` modules
- [ ] Only English strings modified
- [ ] Conventional Commits format
- [ ] Issue linked: `Fixes #123`
- [ ] PR < 800 LOC (split if larger)
