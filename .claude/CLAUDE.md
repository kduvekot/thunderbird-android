# Claude AI Assistant Guide for Thunderbird for Android

This guide ensures all AI-assisted contributions to Thunderbird for Android strictly adhere to project standards and workflows.

---

## 🚨 CRITICAL RULES - NEVER VIOLATE

### Git Branch Management
- **ALWAYS** develop on: `claude-issue-935` branch
- **NEVER** commit or push to `main` branch
- **ALWAYS** create branches from the latest `main`
- **ALWAYS** use `git push -u origin <branch-name>` for pushing
- Branch names must start with `claude/` and match session ID pattern

### Pre-Commit Checks (MANDATORY)
Before ANY commit, you MUST run:
```bash
./gradlew check
```

This runs:
- All tests (`./gradlew test`)
- Code formatting checks (`./gradlew spotlessCheck`)
- Static analysis (`./gradlew detekt`)
- Lint checks (`./gradlew lint`)

If checks fail, fix issues before committing.

### Code Formatting (AUTO-FIX)
Before committing, ALWAYS run:
```bash
./gradlew spotlessApply
```

This auto-formats code to project standards.

---

## 📋 Git Commit Guidelines

### Conventional Commits Format (REQUIRED)
```
<type>(<scope>): <description>

<body>

<footer>
```

**Commit Types:**
- `feat`: New features
- `fix`: Bug fixes
- `docs`: Documentation only
- `style`: Code style (no logic changes)
- `refactor`: Code changes (no features/fixes)
- `test`: Adding/editing tests
- `chore`: Tooling, CI, dependencies
- `revert`: Reverting previous commits

**Examples:**
```bash
feat(email): add validation for email input

fix(auth): handle null response from login endpoint

Checks for missing tokens to prevent app crash during login.

Fixes #123
```

**Commit Best Practices:**
- ✅ One commit, one purpose
- ✅ Keep commits manageable (<200 lines)
- ✅ Each commit should leave codebase buildable
- ✅ Reference issue numbers: "Fixes #123", "Resolves #456"
- ❌ Don't mix unrelated changes
- ❌ Never commit broken code or failing tests

---

## 🏗️ Architecture Rules (STRICT)

### Module Organization
- **New code goes in:**
  - `feature:*` - Feature modules
  - `core:*` - Core utilities
  - `library:*` - Specific implementations
- **NEVER add code to `legacy:*` modules** (unless strictly necessary and justified)

### Module Structure
- **ALWAYS maintain API/impl separation:**
  - `feature:foo:api` - Public interfaces, models
  - `feature:foo:impl` - Concrete implementations
- **External dependencies:**
  - ✅ Depend on `:feature:foo:api`
  - ❌ NEVER depend on `:feature:foo:impl`

### Clean Architecture Layers
**ALWAYS respect layer boundaries:**

```
UI Layer (Presentation)
    ↓ (can call)
Domain Layer (Business Logic)
    ↓ (can call)
Data Layer (Storage/Network)
```

**UI Layer:**
- Jetpack Compose screens
- ViewModels (MVI pattern)
- UI State (immutable data classes)
- Events (user interactions)
- Effects (one-time side effects)

**Domain Layer:**
- Use Cases (business logic)
- Domain Models (entities)
- Repository Interfaces (contracts)

**Data Layer:**
- Repository Implementations
- Data Sources (API, database, preferences)
- Data Transfer Objects (DTOs)

**NEVER:**
- ❌ Call data sources directly from UI
- ❌ Put business logic in UI layer
- ❌ Create circular dependencies

### Dependency Injection
- **ALWAYS use Koin** with constructor injection
- **NEVER use:**
  - Static singletons
  - Service locators
  - Field injection

Example:
```kotlin
class FeatureViewModel(
    private val useCase: FeatureUseCase,
    private val logger: Logger,
) : ViewModel()

val featureModule = module {
    viewModel { FeatureViewModel(get(), get()) }
    single<FeatureRepository> { FeatureRepositoryImpl(get(), get()) }
}
```

---

## 🧪 Testing Requirements (MANDATORY)

### Test Structure
**ALWAYS use AAA pattern:**
```kotlin
@Test
fun `feature should return expected result when given valid input`() {
    // Arrange
    val input = "test"
    val testSubject = SystemUnderTest()

    // Act
    val result = testSubject.process(input)

    // Assert
    assertThat(result).isEqualTo("expected")
}
```

### Testing Conventions (STRICT)
- ✅ Name object under test as `testSubject` (NOT "sut")
- ✅ Use backticks for test names (JVM tests only)
- ✅ Use camelCase for Android instrumentation tests
- ✅ Prefer **fakes** over mocks
- ✅ Use **AssertK** for assertions
- ✅ Add comments separating Arrange/Act/Assert sections

### Fake Implementation Pattern
```kotlin
// Interface
interface DataRepository {
    fun getData(): List<String>
}

// Fake for testing (PREFERRED)
class FakeDataRepository(
    initialData: List<String> = emptyList()
) : DataRepository {
    var dataToReturn = initialData
    override fun getData(): List<String> = dataToReturn
}

// In test
@Test
fun `processor should transform data correctly`() {
    // Arrange
    val fakeRepo = FakeDataRepository(listOf("item1", "item2"))
    val testSubject = DataProcessor(fakeRepo)

    // Act
    val result = testSubject.process()

    // Assert
    assertThat(result).containsExactly("ITEM1", "ITEM2")
}
```

### Test Requirements
- ✅ **ALWAYS** add tests for new/changed code
- ✅ Add unit tests for business logic
- ✅ Add integration tests for component interactions
- ✅ Add UI tests for user interface changes
- ✅ Cover edge cases and error handling
- ❌ NEVER commit without tests

### Test Types
- **Unit Tests:** `src/test/` (JVM tests - PREFERRED)
- **Integration Tests:** `src/test/` or `src/androidTest/` (only if Android-specific)
- **UI Tests:** `src/test/` (Compose) or `src/androidTest/` (Espresso)

**Prefer `src/test/` over `src/androidTest/`:**
- Faster execution (JVM vs emulator)
- Better CI/CD integration
- Use Robolectric for Android framework classes

---

## 🎨 Code Style Guidelines

### Kotlin Style (MANDATORY)
- **Naming:**
  - `camelCase` for variables, functions, methods
  - `PascalCase` for classes, interfaces, enums, type parameters
  - `UPPER_SNAKE_CASE` for constants
  - Prefix implementations: `Default`, `InMemory`, specific name
    - `DefaultEmailRepository` implements `EmailRepository`
    - `InMemoryCache` implements `Cache`

- **Formatting:**
  - 4 spaces for indentation (NOT tabs)
  - 120 character line length limit
  - Use `./gradlew spotlessApply` to auto-format

- **Comments:**
  - KDoc for public APIs
  - Include parameter descriptions and return values
  - Document exceptions
  - Document non-obvious logic

### Kotlin Best Practices
- ✅ Prefer `val` (immutable) over `var`
- ✅ Use null-safety: `?.`, `?:`, `requireNotNull`, `checkNotNull`
- ✅ Use extension functions for utilities
- ✅ Use functional programming: `map`, `filter`, `reduce`
- ✅ Use `data classes` for model objects
- ✅ Use `sealed classes` for finite sets
- ✅ Use `coroutines` for async operations
- ✅ Use `Flow` for reactive programming
- ✅ Keep functions small and focused

### Android Best Practices
- ✅ Follow Android app architecture guidelines
- ✅ Use Jetpack libraries appropriately
- ✅ Manage lifecycle properly (ViewModel, LifecycleOwner)
- ✅ Handle configuration changes (rotation, locale, dark mode)
- ✅ Optimize for different screen sizes
- ✅ Follow Material 3 design guidelines

---

## 🔒 Security & Privacy (CRITICAL)

### Input Validation
- ✅ **ALWAYS** validate all user input
- ✅ Prevent injection attacks (SQL, command, XSS)
- ✅ Check for OWASP top 10 vulnerabilities

### Data Protection
- ✅ Use HTTPS/TLS for ALL network traffic
- ✅ Store secrets securely (Android Keystore, EncryptedSharedPreferences)
- ❌ **NEVER** log sensitive data or PII (Personally Identifiable Information)
- ❌ **NEVER** store passwords in plain text

### Error Handling
**ALWAYS use Outcome pattern instead of exceptions:**

```kotlin
sealed class Outcome<out T, out E> {
    data class Success<T>(val value: T) : Outcome<T, Nothing>()
    data class Failure<E>(val error: E) : Outcome<Nothing, E>()
}

// Define domain errors
sealed class AccountError {
    data class AuthenticationFailed(val reason: String) : AccountError()
    data class NetworkError(val exception: Exception) : AccountError()
    data class ValidationError(val field: String, val message: String) : AccountError()
}

// Use in repository
fun authenticate(credentials: Credentials): Outcome<AuthResult, AccountError> {
    return try {
        val result = apiClient.authenticate(credentials)
        Outcome.Success(result)
    } catch (e: HttpException) {
        val error = when (e.code()) {
            401 -> AccountError.AuthenticationFailed("Invalid credentials")
            else -> AccountError.NetworkError(e)
        }
        logger.error(e) { "Authentication failed: ${error::class.simpleName}" }
        Outcome.Failure(error)
    }
}

// Handle in ViewModel
viewModelScope.launch {
    val outcome = loginUseCase.execute(credentials)
    when (outcome) {
        is Outcome.Success -> {
            _uiState.update { it.copy(isLoggedIn = true) }
        }
        is Outcome.Failure -> {
            val errorMessage = when (val error = outcome.error) {
                is AccountError.AuthenticationFailed ->
                    stringProvider.getString(R.string.error_auth_failed, error.reason)
                is AccountError.NetworkError ->
                    stringProvider.getString(R.string.error_network)
                is AccountError.ValidationError ->
                    stringProvider.getString(R.string.error_validation, error.field)
            }
            _uiState.update { it.copy(error = errorMessage) }
        }
    }
}
```

### Logging
**ALWAYS inject Logger and use appropriate levels:**

```kotlin
class AccountRepository(
    private val apiClient: ApiClient,
    private val logger: Logger,
) {
    fun syncAccount(account: Account) {
        logger.info { "Syncing account: ${account.email}" }
        try {
            apiClient.sync(account)
            logger.debug { "Sync completed for: ${account.email}" }
        } catch (e: Exception) {
            logger.error(e) { "Sync failed for: ${account.email}" }
        }
    }
}
```

**Log Levels:**
- `verbose` - Detailed debugging (debug builds only)
- `debug` - General debugging
- `info` - Important events (visible in production)
- `warn` - Potential issues
- `error` - Functionality issues

**Logging Best Practices:**
- ✅ Use lambda syntax: `logger.debug { "Message: $var" }`
- ✅ Include relevant context
- ✅ Log exceptions with context
- ❌ NEVER log PII or sensitive data
- ❌ Avoid string concatenation: `logger.debug("Message: " + var)` ❌

---

## 🌐 Internationalization (i18n)

### String Management (STRICT RULES)
- ✅ **ONLY modify English source strings** in `res/values/strings.xml`
- ❌ **NEVER** edit translation files (`res/values-*/strings.xml`)
- ❌ **NEVER** concatenate localized strings

### Adding Strings
```xml
<!-- res/values/strings.xml -->
<string name="new_string_key">English text here</string>
```
- Do NOT add translations
- Weblate will handle translations after merge

### Changing Strings

**Typos/Grammar (keep same key):**
```xml
<!-- Before -->
<string name="action_check">Recieve emails</string>

<!-- After (same key) -->
<string name="action_check">Receive emails</string>
```

**Meaning Changes (new key required):**
1. Add new key with new string
2. Update all code references to new key
3. Delete old key from `res/values/strings.xml`
4. Delete old key from ALL `res/values-*/strings.xml`
5. Build to verify no references remain

### Removing Strings
1. Delete key from `res/values/strings.xml`
2. Delete key from ALL `res/values-*/strings.xml`
3. Build to verify no references remain

---

## 📬 Pull Request Requirements

### PR Size & Scope
- ✅ Keep PRs focused on single concern
- ✅ Aim for <800 lines of code (LOC)
- ✅ Split large or mixed changes
- ✅ Use Draft PRs for early feedback

### PR Description (MANDATORY)
```markdown
## Title
fix(email): add validation for email input

## Description
Fixes #123

This PR adds email validation to the login form. It:
- Implements regex-based validation for email inputs
- Shows error messages for invalid emails
- Adds unit tests for the validation logic

## Screenshots
[Include for UI changes]

## Testing
1. Enter invalid email (e.g., "test@")
2. Verify error message appears
3. Enter valid email
4. Verify error disappears

## Checklist
- [x] Tests added/updated
- [x] CI green (./gradlew check passed)
- [x] Spotless applied (./gradlew spotlessApply)
- [x] Architecture respected (layer boundaries)
- [x] Security reviewed (input validation, no PII logging)
- [x] Accessibility considered (TalkBack, contrast, touch targets)
- [x] Documentation updated
- [x] Issues linked (Fixes #123)
```

### PR Checklist (Self-Review)
- [ ] Focused scope (<800 LOC)
- [ ] Clear description with rationale
- [ ] UI changes: screenshots/videos included
- [ ] Accessibility: TalkBack, contrast, touch targets verified
- [ ] Tests added/updated (AAA pattern, AssertK, fakes preferred)
- [ ] CI green (`./gradlew check` passed)
- [ ] Architecture: business logic outside UI, module API/impl respected
- [ ] DI: constructor injection with Koin
- [ ] Performance: no main-thread blocking, reasonable Compose recompositions
- [ ] Security: inputs validated, no PII in logs, TLS, secure storage
- [ ] i18n: only English source strings modified
- [ ] Docs/CHANGELOG updated
- [ ] Issues linked (Fixes #123)
- [ ] Commits follow Conventional Commits

### For UI Changes
- ✅ Include screenshots or videos
- ✅ Verify accessibility:
  - TalkBack support
  - Sufficient contrast
  - Touch targets (48dp minimum)
  - Dynamic text sizing (up to 200%)
- ✅ Provide `contentDescription` for images/icons

### Performance Considerations
- ✅ Use coroutines with appropriate dispatchers
- ✅ Avoid blocking main thread
- ✅ Watch allocations in hot paths
- ✅ Minimize Compose recompositions
- ✅ Use `remember`, `derivedStateOf` appropriately

---

## 🔄 Development Workflow

### Step-by-Step Process

1. **Find/Create Issue**
   - Browse [GitHub Issues](https://github.com/thunderbird/thunderbird-android/issues)
   - Look for `good first issue` labels if new
   - Avoid `unconfirmed` labeled issues
   - Comment on issue to claim it

2. **Setup Branch**
   ```bash
   # Ensure on main
   git checkout main

   # Pull latest
   git pull origin main

   # Create feature branch
   git checkout -b claude-issue-935
   ```

3. **Make Changes**
   - Follow architecture guidelines
   - Write clear, documented code
   - Add/update tests
   - Run `./gradlew spotlessApply` frequently

4. **Pre-Commit Checks**
   ```bash
   # Auto-format code
   ./gradlew spotlessApply

   # Run all checks
   ./gradlew check
   ```

5. **Commit Changes**
   ```bash
   git add .
   git commit -m "feat(component): add feature X

   Detailed description of changes.

   Fixes #123"
   ```

6. **Push to Branch**
   ```bash
   # First push
   git push -u origin claude-issue-935

   # Subsequent pushes
   git push
   ```

7. **Create Pull Request**
   - Target: `thunderbird/thunderbird-android` main branch
   - Source: your `claude-issue-935` branch
   - Fill out PR description template
   - Link issues (Fixes #123)
   - Add screenshots for UI changes

8. **Address Review Feedback**
   - Make requested changes
   - Commit with descriptive messages
   - Push to same branch
   - Request re-review

9. **After Merge**
   ```bash
   # Update local main
   git checkout main
   git pull origin main

   # Delete feature branch
   git branch -d claude-issue-935
   ```

---

## 🚫 Common Mistakes to AVOID

### Code Organization
- ❌ Adding code to `legacy:*` modules
- ❌ Bypassing architecture layers (UI calling data sources)
- ❌ Creating circular dependencies
- ❌ Leaking implementation details across module boundaries

### Testing
- ❌ Using mocks instead of fakes
- ❌ Naming test subject as "sut" (use `testSubject`)
- ❌ Skipping tests for new code
- ❌ Not following AAA pattern

### Git & Commits
- ❌ Committing to `main` branch
- ❌ Not running `./gradlew check` before commit
- ❌ Mixing unrelated changes in one commit
- ❌ Not using Conventional Commits format
- ❌ Forgetting to reference issue numbers

### Code Style
- ❌ Using `var` when `val` suffices
- ❌ Not running `./gradlew spotlessApply`
- ❌ Exceeding 120 character line length
- ❌ Using tabs instead of 4 spaces

### Security & i18n
- ❌ Logging PII or sensitive data
- ❌ Editing translation files
- ❌ Concatenating localized strings
- ❌ Storing secrets in plain text
- ❌ Not validating user input

### Pull Requests
- ❌ PRs over 800 LOC
- ❌ Mixing multiple unrelated changes
- ❌ Missing screenshots for UI changes
- ❌ Not linking issues
- ❌ Ignoring accessibility

---

## 📚 Quick Reference

### Essential Commands
```bash
# Format code (ALWAYS before commit)
./gradlew spotlessApply

# Run all checks (MANDATORY before commit)
./gradlew check

# Run tests only
./gradlew test

# Run lint
./gradlew lint

# Run Detekt
./gradlew detekt

# Run specific module tests
./gradlew :module-name:test
```

### File Locations
- **Documentation:** `docs/`
- **Contributing Guide:** `docs/CONTRIBUTING.md`
- **Architecture:** `docs/architecture/README.md`
- **English Strings:** `res/values/strings.xml`
- **Translations:** `res/values-*/strings.xml` (DO NOT EDIT)
- **Detekt Config:** `config/detekt/detekt.yml`
- **Lint Config:** `config/lint/lint.xml`
- **EditorConfig:** `.editorconfig`

### Key Documentation Links
- [Contribution Workflow](docs/contributing/contribution-workflow.md)
- [Development Guide](docs/contributing/development-guide.md)
- [Code Quality Guide](docs/contributing/code-quality-guide.md)
- [Testing Guide](docs/contributing/testing-guide.md)
- [Git Commit Guide](docs/contributing/git-commit-guide.md)
- [Code Review Guide](docs/contributing/code-review-guide.md)
- [Managing Strings](docs/contributing/managing-strings.md)
- [Architecture](docs/architecture/README.md)

---

## ✅ Pre-Commit Checklist

Before EVERY commit, verify:

- [ ] `./gradlew spotlessApply` executed
- [ ] `./gradlew check` passed (all tests, lint, detekt)
- [ ] Committing to `claude-issue-935` branch (NOT main)
- [ ] Using Conventional Commits format
- [ ] Tests added/updated for changes
- [ ] No code added to `legacy:*` modules
- [ ] Architecture layers respected
- [ ] Only English strings modified (if applicable)
- [ ] No PII in logs
- [ ] No sensitive data in plain text
- [ ] Issue number referenced (Fixes #123)

---

## 🎯 Success Criteria

Your contribution is successful when:

1. ✅ All CI checks pass (green build)
2. ✅ Code follows all architecture guidelines
3. ✅ Tests provide good coverage (AAA pattern, fakes, AssertK)
4. ✅ Security and privacy requirements met
5. ✅ Accessibility considered for UI changes
6. ✅ Documentation updated appropriately
7. ✅ Commits follow Conventional Commits
8. ✅ PR description is clear and complete
9. ✅ Review feedback addressed promptly
10. ✅ Mozilla Community Participation Guidelines followed

---

## 🆘 When in Doubt

- **Ask questions** in PR comments or GitHub issue
- **Check existing code** for patterns and examples
- **Review documentation** thoroughly
- **Follow Mozilla Community Participation Guidelines**
- **Be patient** with review process
- **Be respectful** in all interactions

---

**Remember:** Quality over speed. Take time to understand the architecture, write good tests, and follow all guidelines. This ensures maintainability and helps the entire community.
