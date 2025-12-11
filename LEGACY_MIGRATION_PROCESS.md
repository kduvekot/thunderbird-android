# Legacy to Modern Code Migration Process

This document describes the process for migrating code from the legacy modules (`legacy:*`, `mail:*`, `backend:*`) to the modern modular architecture in Thunderbird for Android.

## Overview

The Thunderbird for Android project is transitioning from a monolithic architecture to a modular one. During this transition, legacy code is isolated and accessed through controlled interfaces, with the `:app-common` module acting as a bridge between old and new code.

## Architecture Summary

### Module Hierarchy (Dependencies Flow Downward)

```
App Modules (app-thunderbird, app-k9mail)
    ↓
App Common Module (:app-common) ← Central hub & legacy bridge
    ↓
Feature Modules (:feature:*)
    ↓
Core Modules (:core:*)
    ↓
Library Modules (:library:*)

Legacy Modules (:legacy:*, :mail:*, :backend:*) ← Accessed ONLY via app-common
```

### Key Principle

**New modules must NEVER directly depend on legacy modules.** All access to legacy functionality goes through interfaces defined in feature/core modules, with `app-common` providing the bridge implementations.

---

## The 7-Step Migration Process

### Step 1: Identify Functionality

Pinpoint specific functionalities within legacy modules that need to be modernized.

**Questions to ask:**
- What business logic needs to be preserved?
- What data structures are involved?
- What are the current dependencies?

### Step 2: Define Interfaces

Create clear interfaces in the appropriate modern module.

**Location guidelines:**
- Feature-specific interfaces → `feature:*:api` modules
- Cross-cutting functionality → `core:*` modules

**Example:**
```kotlin
// In feature:account:api
interface AccountProfileRepository {
    fun getById(id: AccountId): Flow<AccountProfile?>
    suspend fun update(profile: AccountProfile)
}
```

### Step 3: Entity Modeling

Create proper domain entity models as immutable data classes.

**Guidelines:**
- Use immutable `data class` definitions
- No methods for conversion (use separate mappers)
- Represent the business objects in modern, clean form

**Example:**
```kotlin
// In feature:account:api
data class AccountProfile(
    val id: AccountId,
    val name: String,
    val email: String,
    val avatar: AccountAvatar?,
    val color: Int
)
```

### Step 4: Create Bridge Implementation in app-common

Implement the interface in `app-common`, delegating to legacy code.

**This involves:**
1. Creating wrapper classes around legacy data structures
2. Creating adapter implementations that implement modern interfaces
3. Creating dedicated mapper classes for data conversion

**Example structure:**
```kotlin
// In app-common
class DefaultAccountProfileLocalDataSource(
    private val accountManager: LegacyAccountWrapperManager,
    private val dataMapper: AccountProfileDataMapper
) : AccountProfileLocalDataSource {

    override fun getById(id: AccountId): Flow<AccountProfile?> {
        return accountManager.getById(id)
            .map { wrapper -> wrapper?.let { dataMapper.toDomain(it) } }
    }
}
```

### Step 5: Configure Dependency Injection

Register the bridge implementation in Koin modules.

**Example:**
```kotlin
// In app-common
val appCommonAccountModule = module {
    factory<AccountProfileLocalDataSource> {
        DefaultAccountProfileLocalDataSource(
            accountManager = get(),
            dataMapper = get()
        )
    }
}
```

### Step 6: Implement in New Modules (The Modern Implementation)

Create a full modern implementation in feature modules when ready.

**This is the gradual replacement phase:**
- Re-implement the functionality within new feature `impl` modules
- Use modern patterns (Repository, Use Cases, Clean Architecture)
- Follow Kotlin best practices

### Step 7: Switch and Retire

Once the modern implementation is complete:

1. **Update DI Configuration**: Switch from bridge to modern implementation
2. **Verify Behavior**: Run the same test suite against both implementations
3. **Remove Legacy Code**: Delete unused legacy code and bridge implementations

---

## Implementation Techniques

### Wrapper Classes

Create immutable data classes that wrap legacy data structures:

```kotlin
// In core:android:account
data class LegacyAccountWrapper(
    val id: AccountId,
    val uuid: String,
    val name: String,
    // ... other properties
) {
    // No conversion methods here - use mappers instead
}
```

### Mapper Classes

Dedicated classes for data conversion between legacy and modern structures:

```kotlin
class DefaultAccountProfileDataMapper(
    private val avatarMapper: AccountAvatarDataMapper
) : AccountProfileDataMapper {

    fun toDomain(wrapper: LegacyAccountWrapper): AccountProfile {
        return AccountProfile(
            id = wrapper.id,
            name = wrapper.name,
            // ... map other properties
        )
    }

    fun toDto(profile: AccountProfile): LegacyAccountWrapper {
        // Reverse mapping for updates
    }
}
```

### Adapter Implementations

Classes in `app-common` that implement modern interfaces but delegate to legacy code:

```kotlin
class LegacyMailRepository(
    private val legacyMailController: MailController
) : MailRepository {

    override suspend fun getMessages(folderId: FolderId): List<Message> {
        // Delegate to legacy, convert result to modern types
        return legacyMailController.getMessages(folderId.value)
            .map { legacyMessage -> messageMapper.toDomain(legacyMessage) }
    }
}
```

---

## Testing the Migration

### 1. Unit Testing Bridge Classes

```kotlin
class DefaultAccountProfileLocalDataSourceTest {

    @Test
    fun `getById should return account profile`() = runTest {
        // Use fake implementations of legacy dependencies
        val testSubject = DefaultAccountProfileLocalDataSource(
            accountManager = FakeLegacyAccountWrapperManager(
                initialAccounts = listOf(testAccount)
            ),
            dataMapper = DefaultAccountProfileDataMapper()
        )

        testSubject.getById(accountId).test {
            assertThat(awaitItem()).isEqualTo(expectedProfile)
        }
    }
}
```

### 2. Creating Test Doubles

```kotlin
class FakeLegacyAccountWrapperManager(
    initialAccounts: List<LegacyAccountWrapper> = emptyList()
) : LegacyAccountWrapperManager {

    private val accountsState = MutableStateFlow(initialAccounts)

    override fun getAll(): Flow<List<LegacyAccountWrapper>> = accountsState
    // ... implement other methods
}
```

### 3. Migration Testing

When migrating from bridge to new implementation:
- Run the same test suite against both implementations
- Ensure behavior consistency during transition

---

## Code Placement Rules

### Where to Put NEW Code

| Code Type | Location |
|-----------|----------|
| New features | `feature:*:impl` modules |
| UI components | `core:ui` or `feature:*` (if feature-specific) |
| Business logic | `feature:*:impl` (domain layer) |
| Shared utilities | `core:common` |
| Bridge/adapter code | `app-common` |

### What Should NOT Go in app-common

- Feature-specific business logic
- UI components
- Direct legacy code (keep in legacy modules)
- New feature implementations

### What SHOULD Go in app-common

- Legacy code bridges/adapters
- Feature integration code
- Common dependency injection setup
- Shared application logic (e.g., `BaseApplication`)

---

## Java to Kotlin Conversion

When migrating Java code to Kotlin:

1. **Write tests first** for code that lacks adequate test coverage
2. **Use IDE conversion**: "Convert Java File to Kotlin File" in IntelliJ/Android Studio
3. **Fix compilation issues** after automatic conversion
4. **Commit separately**:
   - First commit: file extension change (`.java` → `.kt`)
   - Second commit: the actual conversion
5. **Refactor for idiomatic Kotlin**:
   - Use `when` expressions instead of `if-else`
   - Use Kotlin standard library functions
   - Apply null safety properly
   - Use `apply`, `also`, and other scope functions

---

## Do's and Don'ts

### DO

- Write modular, testable code with clear boundaries
- Document non-obvious logic and decisions
- Keep module boundaries clean
- Add/update tests for new/changed code
- Run Spotless, Detekt, and Lint checks locally before committing
- Prefer fakes over mocks in tests
- Use Jetpack Compose for UI
- Use Koin for dependency injection

### DON'T

- Commit new code to `legacy:*` modules (unless strictly necessary)
- Bypass architecture/layering (e.g., UI calling data sources directly)
- Introduce circular dependencies between modules
- Create direct dependencies from feature modules to legacy modules
- Add new Java code (use Kotlin)
- Use XML layouts for new UI (use Jetpack Compose)

---

## Quick Reference: Migration Checklist

- [ ] Identify the legacy functionality to migrate
- [ ] Define interface in appropriate `feature:*:api` or `core:*` module
- [ ] Create immutable domain entity models
- [ ] Create wrapper class for legacy data (if needed)
- [ ] Create mapper class for data conversion
- [ ] Implement bridge in `app-common`
- [ ] Register in Koin DI module
- [ ] Write unit tests for bridge
- [ ] (Later) Implement modern version in feature module
- [ ] (Later) Switch DI to use modern implementation
- [ ] (Later) Remove legacy code and bridge

---

## Related Documentation

- [Module Organization](docs/architecture/module-organization.md)
- [Module Structure](docs/architecture/module-structure.md)
- [Legacy Module Integration](docs/architecture/legacy-module-integration.md)
- [Architecture Overview](docs/architecture/README.md)
- [Java to Kotlin Conversion Guide](docs/contributing/java-to-kotlin-conversion-guide.md)
- [Development Guide](docs/contributing/development-guide.md)
