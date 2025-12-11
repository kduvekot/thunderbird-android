# Android Test Infrastructure Analysis for Catch-All Identity Feature

## Executive Summary

This document analyzes Thunderbird Android's existing test infrastructure, identity model, and testing patterns to guide the implementation of catch-all identity support. The feature will allow users to reply from wildcard addresses (e.g., `*@domain.com`) matching the originally received address.

## 1. Testing Framework and Patterns

### Framework Stack
- **Test Framework**: JUnit 4
- **Assertion Library**: AssertK
- **Mocking**: Mockito-Kotlin
- **Android Testing**: Robolectric (for unit tests requiring Android context)
- **Test Location**: Tests reside in `src/test` directories (JVM tests) and `src/androidTest` (instrumentation tests)

### Test Naming Conventions
- **JVM Tests**: Use backticks for descriptive names
  ```kotlin
  @Test
  fun `message sent to only our identity`() { ... }
  ```
- **Android Tests**: Use camelCase
  ```kotlin
  @Test
  fun shouldDoXWhenY() { ... }
  ```

### Test Structure Pattern (AAA)
All tests follow Arrange-Act-Assert pattern with comments:
```kotlin
@Test
fun `test description`() {
    // Arrange
    val testSubject = createTestSubject()

    // Act
    val result = testSubject.performAction()

    // Assert
    assertThat(result).isEqualTo(expected)
}
```

### Key Testing Conventions
- Test subject variable named `testSubject` (NOT "sut")
- Use fakes over mocks where possible
- Tests extend `RobolectricTest` for Android-dependent tests
- Mock setup uses `stubbing` and `doReturn` patterns from Mockito-Kotlin

## 2. Existing Identity-Related Tests (Legacy - Reference Only)

⚠️ **IMPORTANT**: All tests listed in this section are in `legacy:*` modules. They are documented here to understand existing patterns and test coverage, but **DO NOT modify these tests**. New catch-all tests must be created in the new `feature/account/identity-matcher` module (see Section 7).

### 2.1 IdentityHelperTest ⚠️ LEGACY
**Location**: `legacy/core/src/test/java/com/fsck/k9/helper/IdentityHelperTest.kt`

**Purpose**: Tests identity selection based on message recipient headers (DO NOT MODIFY)

**Key Test Cases**:
- Header priority testing (TO > CC > X-Original-To > Delivered-To > X-Envelope-To)
- Fallback to first identity when no match found
- Exact email matching (case-insensitive)

**Pattern Example**:
```kotlin
@Test
fun getRecipientIdentityFromMessage_prefersToOverCc() {
    val message = messageWithRecipients(
        RecipientType.TO to IDENTITY_1_ADDRESS,
        RecipientType.CC to IDENTITY_2_ADDRESS,
    )

    val identity = IdentityHelper.getRecipientIdentityFromMessage(account, message)

    assertThat(identity.email).isEqualTo(IDENTITY_1_ADDRESS)
}
```

**Header Priority Order** (as tested):
1. `TO`
2. `CC`
3. `X-Original-To`
4. `Delivered-To`
5. `X-Envelope-To`

**Note**: This differs from Desktop's priority (Desktop: Delivered-To > X-Envelope-To > X-Original-To > To > Cc). We should align with Desktop's priority for consistency.

### 2.2 IdentityHeaderBuilderTest ⚠️ LEGACY
**Location**: `legacy/core/src/test/java/com/fsck/k9/message/IdentityHeaderBuilderTest.kt`

**Purpose**: Tests building identity headers for message composition (DO NOT MODIFY)

**Key Tests**:
- Valid unstructured header field values
- Identity header without identity name
- Long signature handling

### 2.3 IdentityHeaderParserTest ⚠️ LEGACY
**Location**: `legacy/core/src/test/java/com/fsck/k9/message/IdentityHeaderParserTest.kt`

**Purpose**: Tests parsing identity information from message headers (DO NOT MODIFY)

### 2.4 ReplyActionStrategyTest ⚠️ LEGACY
**Location**: `legacy/core/src/test/java/com/fsck/k9/message/ReplyActionStrategyTest.kt`

**Purpose**: Tests determining reply actions (Reply vs Reply All) (DO NOT MODIFY)

**Pattern**: Uses `buildMessage` DSL for creating test messages
```kotlin
val message = buildMessage {
    header("From", "sender@domain.example")
    header("To", IDENTITY_EMAIL_ADDRESS)
}
```

### 2.5 ReplyToPresenterTest ⚠️ LEGACY
**Location**: `legacy/ui/legacy/src/test/java/com/fsck/k9/activity/compose/ReplyToPresenterTest.kt`

**Purpose**: Tests Reply-To field management and identity switching (DO NOT MODIFY)

### 2.6 Non-Legacy Tests ✅ REFERENCE FOR PATTERNS

**Important**: These tests in non-legacy modules show the patterns and conventions to follow for NEW tests.

**Feature Module Tests** (`feature/account/storage/legacy/src/test/kotlin/`):
- `DefaultLegacyAccountWrapperDataMapperTest.kt` - Tests account/identity data mapping
- `LegacyProfileDtoStorageHandlerTest.kt` - Tests profile storage
- `AccountKeyGeneratorTest.kt` - Tests account key generation

**Core Module Tests** (`core/android/account/src/test/kotlin/`):
- `LegacyAccountWrapperTest.kt` - Tests account wrapper functionality

**App-Common Tests** (`app-common/src/test/kotlin/`):
- `DefaultAccountProfileLocalDataSourceTest.kt` - Tests account profile data source
- `DefaultAccountDefaultsProviderTest.kt` - Tests account defaults

**Feature Account Tests** (`feature/account/*/src/test/kotlin/`):
- Account setup UI tests (ViewModels, State)
- Account edit UI tests (ViewModels, State)
- Server settings validation tests

**Key Observation**: No non-legacy tests exist for identity **matching/selection logic**. This functionality currently only has legacy tests (IdentityHelperTest). The new `feature/account/identity-matcher` module will be the first non-legacy implementation with proper tests.

## 3. Identity Data Model

### 3.1 Identity Data Class
**Location**: `core/android/account/src/main/kotlin/net/thunderbird/core/android/account/Identity.kt`

**Current Structure**:
```kotlin
@Parcelize
data class Identity(
    val description: String? = null,
    val name: String? = null,
    val email: String? = null,
    val signature: String? = null,
    val signatureUse: Boolean = false,
    val replyTo: String? = null,
) : Parcelable
```

**Required Addition**:
```kotlin
val catchAll: String? = null  // e.g., "*@domain.com"
```

### 3.2 Storage Implementation
**Location**: `feature/account/storage/legacy/src/main/kotlin/net/thunderbird/feature/account/storage/legacy/LegacyAccountStorageHandler.kt`

**Storage Pattern**: Identities stored in SharedPreferences with indexed keys:
- `{accountUuid}.name.{index}` → Identity name
- `{accountUuid}.email.{index}` → Identity email
- `{accountUuid}.signatureUse.{index}` → Signature usage flag
- `{accountUuid}.signature.{index}` → Signature text
- `{accountUuid}.description.{index}` → Identity description
- `{accountUuid}.replyTo.{index}` → Reply-To address

**New Key Required**:
- `{accountUuid}.catchAll.{index}` → Catch-all pattern

**Load Function** (lines 262-307):
```kotlin
private fun loadIdentities(accountId: AccountId, storage: Storage): List<Identity> {
    // Iterates through index 0, 1, 2... until no email found
    // Creates Identity objects from stored values
}
```

**Save Function** (lines 548-564):
```kotlin
private fun saveIdentities(data: LegacyAccountDto, storage: Storage, editor: StorageEditor) {
    deleteIdentities(data, storage, editor)  // Clear existing
    // Iterate identities and save each field by index
}
```

**Delete Function** (lines 567-586):
```kotlin
private fun deleteIdentities(data: LegacyAccountDto, storage: Storage, editor: StorageEditor) {
    // Removes all identity keys for all indices
}
```

## 4. Identity Matching Logic (Legacy - Reference Only)

⚠️ **DO NOT MODIFY**: This section documents legacy implementation for understanding only. New matching logic goes in `feature/account/identity-matcher` (see Section 6.3).

### 4.1 Current Implementation ⚠️ LEGACY
**Location**: `legacy/core/src/main/java/com/fsck/k9/helper/IdentityHelper.kt` (DO NOT MODIFY)

**Algorithm**:
```kotlin
fun getRecipientIdentityFromMessage(account: LegacyAccountDto, message: Message): Identity {
    val recipient: Identity? = RECIPIENT_TYPES.asSequence()
        .flatMap { recipientType -> message.getRecipients(recipientType).asSequence() }
        .map { address -> account.findIdentity(address) }
        .filterNotNull()
        .firstOrNull()

    return recipient ?: account.getIdentity(0)
}
```

**LegacyAccountDto.findIdentity** (lines 550-554):
```kotlin
fun findIdentity(address: Address): Identity? {
    return identities.find { identity ->
        identity.email.equals(address.address, ignoreCase = true)
    }
}
```

**Limitation**: Only exact email matching; no pattern matching support.

### 4.2 New Implementation Required (In Feature Module)
The NEW `IdentityMatcher` in `feature/account/identity-matcher` will support catch-all patterns:
1. First try exact email match (existing behavior)
2. If no exact match, try catch-all pattern matching
3. Return null if no match found

**DO NOT enhance the legacy findIdentity method** - create new implementation in feature module.

## 5. Message Compose/Reply Flow (Legacy - Reference Only)

⚠️ **REFERENCE ONLY**: This section documents legacy compose flow for understanding integration points. DO NOT add new code here.

### 5.1 MessageActions Entry Points ⚠️ LEGACY
**Location**: `legacy/ui/legacy/src/main/java/com/fsck/k9/activity/compose/MessageActions.kt` (DO NOT MODIFY)

**Key Actions**:
- `actionReply()` - Reply to message
- `actionForward()` - Forward message
- `actionEditDraft()` - Edit draft message

All create intents that launch `MessageCompose` activity with:
- Account UUID
- Message reference
- Action type (REPLY, REPLY_ALL, FORWARD, etc.)

### 5.2 Identity Selection Points
Identity selection occurs in:
1. **Initial compose** - When creating reply/forward
2. **Identity switching** - When user manually changes identity
3. **Draft restoration** - When loading saved draft

## 6. Files That Will Need Modification

**IMPORTANT ARCHITECTURAL RULE**: From `.claude/CLAUDE.md`:
```
legacy:* → DO NOT add new code here
```

### 6.1 Core Data Model ✅ SAFE TO MODIFY
**File**: `core/android/account/src/main/kotlin/net/thunderbird/core/android/account/Identity.kt`
- Add `catchAll: String?` field to the Identity data class
- This is in `core`, NOT in `legacy`, so modifications are allowed

### 6.2 Storage Layer ✅ SAFE TO MODIFY
**File**: `feature/account/storage/legacy/src/main/kotlin/net/thunderbird/feature/account/storage/legacy/LegacyAccountStorageHandler.kt`
- Despite "legacy" in path, this is in `feature/account/storage/legacy/` module
- This is a **bridge** to legacy storage, part of the migration strategy
- Add `IDENTITY_CATCH_ALL_KEY` constant
- Update `loadIdentities()` to read catchAll field (line ~270-306)
- Update `saveIdentities()` to write catchAll field (line ~548-564)
- Update `deleteIdentities()` to remove catchAll field (line ~567-586)

### 6.3 NEW Feature Module ✅ MUST CREATE
**Create**: `feature/account/identity-matcher/` module

Module structure following project conventions:
```
feature/account/identity-matcher/
├── api/
│   ├── build.gradle.kts
│   └── src/commonMain/kotlin/net/thunderbird/feature/account/identity/matcher/api/
│       ├── IdentityMatcher.kt           # Interface for identity matching
│       ├── IdentityMatchResult.kt       # Result with matched identity + address
│       └── CatchAllPattern.kt           # Pattern validation
└── impl/
    ├── build.gradle.kts
    ├── src/commonMain/kotlin/net/thunderbird/feature/account/identity/matcher/
    │   ├── DefaultIdentityMatcher.kt    # Implementation
    │   ├── CatchAllPatternMatcher.kt    # Pattern matching logic
    │   └── HeaderAddressExtractor.kt    # Extract addresses from headers
    └── src/commonTest/kotlin/net/thunderbird/feature/account/identity/matcher/
        ├── CatchAllPatternMatcherTest.kt
        ├── DefaultIdentityMatcherTest.kt
        └── HeaderAddressExtractorTest.kt
```

### 6.4 Integration Layer ✅ SAFE TO MODIFY
**Location**: `app-common/src/main/kotlin/`

Create bridge code in app-common:
- `app-common/.../account/identity/IdentityMatcherBridge.kt`
  - Adapts new IdentityMatcher API for use by legacy code
  - Implements dependency injection setup
- `app-common/.../di/IdentityModule.kt`
  - Koin DI configuration for identity matching

### 6.5 Legacy Integration ⚠️ MINIMAL CHANGES ONLY
**Files that MAY call the new feature**:
- `legacy/core/src/main/java/com/fsck/k9/helper/IdentityHelper.kt`
  - Can CALL the new IdentityMatcher via DI
  - Do NOT add pattern matching logic here
  - Keep changes minimal: inject dependency, delegate to new implementation

**Principle**: Legacy code can USE new features, but new features are NOT implemented IN legacy code.

## 7. Test Files to Create/Modify

### 7.1 New Test Files in New Feature Module ✅ CREATE
**Location**: `feature/account/identity-matcher/impl/src/commonTest/kotlin/`

1. **CatchAllPatternMatcherTest.kt**
   - Test pattern to regex conversion
   - Test pattern matching with various email formats
   - Test cases from Desktop (TC-101 to TC-108):
     - Wildcard local part matching: `*@domain.com`
     - Plus addressing patterns: `user+*@domain.com`
     - Domain mismatch rejection
     - Case-insensitive matching
     - Multiple catch-all identities

2. **DefaultIdentityMatcherTest.kt**
   - Test identity selection with exact email match
   - Test identity selection with catch-all fallback
   - Test header priority order (align with Desktop)
   - Test edge cases (no match, multiple matches, empty hints)

3. **HeaderAddressExtractorTest.kt**
   - Test extracting addresses from various headers
   - Test header priority: Delivered-To > X-Envelope-To > X-Original-To > To > Cc
   - Test multiple addresses in same header
   - Test malformed headers

### 7.2 Storage Tests ✅ CREATE
**Location**: `feature/account/storage/legacy/src/test/kotlin/`

1. **LegacyAccountStorageHandlerCatchAllTest.kt**
   - Test saving identity with catchAll field
   - Test loading identity with catchAll field
   - Test null catchAll (backward compatibility)
   - Test updating catchAll value

### 7.3 Integration Tests ✅ CREATE
**Location**: `app-common/src/test/kotlin/`

1. **IdentityMatcherBridgeTest.kt**
   - Test bridge between new feature and legacy code
   - Test dependency injection setup
   - Test end-to-end identity matching flow

### 7.4 Existing Tests - Reference Only ⚠️ DO NOT MODIFY
These exist in legacy modules and should NOT be modified (frozen):
- `legacy/core/src/test/.../IdentityHelperTest.kt` - Keep as-is
- `legacy/ui/legacy/src/test/.../ReplyActionStrategyTest.kt` - Keep as-is

**Note**: Legacy tests remain unchanged. New functionality is tested in the new feature module.

## 8. Testing Strategy

### Phase 1: Unit Tests (Pattern Matching)
- Test catch-all pattern to regex conversion
- Test pattern matching with various email formats
- Test edge cases (invalid patterns, special characters)
- Test case-insensitive matching

### Phase 2: Integration Tests (Identity Selection)
- Test identity selection with exact match
- Test identity selection with catch-all fallback
- Test multiple identities with overlapping patterns
- Test header priority order

### Phase 3: End-to-End Tests (Reply/Forward)
- Test reply with catch-all identity
- Test reply-all with catch-all identity
- Test forward with catch-all identity
- Test BCC'd messages with Delivered-To header

### Phase 4: Storage Tests
- Test saving identity with catchAll field
- Test loading identity with catchAll field
- Test migration from old schema (no catchAll)
- Test import/export with catchAll

## 9. Desktop Test Cases to Port

### Priority 1: Core Pattern Matching (TC-101 to TC-108)
- Wildcard local part matching
- Plus addressing patterns
- Domain mismatch rejection
- Case-insensitive matching
- Multiple catch-all identities

### Priority 2: Header Priority (TC-201 to TC-207)
- Delivered-To header priority
- X-Envelope-To fallback
- X-Original-To fallback
- To/Cc header fallback
- Multiple headers with same name

### Priority 3: Reply Scenarios (TC-301 to TC-306)
- Reply with exact match
- Reply with catch-all match
- Reply-all with catch-all
- BCC'd message reply
- Forwarded message reply

### Priority 4: Edge Cases (TC-501 to TC-512)
- Invalid pattern handling
- Special characters in email
- Punycode domains
- Draft message handling
- Empty hint strings

## 10. Key Differences from Desktop

| Aspect | Desktop | Android | Action |
|--------|---------|---------|--------|
| **Storage** | Preferences file | SharedPreferences | Add catchAll key to storage handler |
| **Header Priority** | Delivered-To first | To first | Align with Desktop order |
| **Language** | C++/JavaScript | Kotlin | Port algorithm idiomatically |
| **Testing** | XPCShell | JUnit + Robolectric | Create equivalent test suite |
| **Pattern Matching** | Built-in regex | Kotlin Regex | Implement pattern converter |

## 11. Implementation Checklist

### Phase 1: Core Data Model
- [ ] Add `catchAll: String?` field to Identity.kt (core/android/account)
- [ ] Update LegacyAccountStorageHandler to persist catchAll field
- [ ] Add storage tests for catchAll persistence

### Phase 2: New Feature Module
- [ ] Create feature/account/identity-matcher module structure
- [ ] Create api module with IdentityMatcher interface
- [ ] Create impl module with pattern matching logic
- [ ] Implement CatchAllPatternMatcher (pattern to regex conversion)
- [ ] Implement HeaderAddressExtractor (extract from headers in priority order)
- [ ] Implement DefaultIdentityMatcher (orchestrates matching logic)
- [ ] Add comprehensive unit tests for all components

### Phase 3: Integration
- [ ] Create IdentityMatcherBridge in app-common
- [ ] Set up Koin DI configuration
- [ ] Minimal changes to legacy IdentityHelper to call new matcher
- [ ] Add integration tests in app-common

### Phase 4: Testing & Validation
- [ ] Verify all Desktop test cases pass (TC-101 to TC-512)
- [ ] Test with various email providers (Gmail, Fastmail, etc.)
- [ ] Verify backward compatibility (existing identities without catchAll)
- [ ] Run `./gradlew check` to ensure all quality checks pass

### Phase 5: Documentation
- [ ] Add ADR (Architecture Decision Record) for catch-all feature
- [ ] Document pattern syntax for users
- [ ] Update relevant documentation in docs/

## 12. Next Steps

1. **Confirm architecture**: Verify feature module approach aligns with team standards
2. **Phase 1**: Add catchAll to Identity data model and storage
3. **Phase 2**: Create feature/account/identity-matcher module with pattern matching
4. **Phase 3**: Bridge via app-common for legacy integration
5. **Phase 4**: Comprehensive testing across all layers
6. **Phase 5**: Documentation and ADR

## Appendix A: Example Test Structure

**Location**: `feature/account/identity-matcher/impl/src/commonTest/kotlin/net/thunderbird/feature/account/identity/matcher/CatchAllPatternMatcherTest.kt`

```kotlin
package net.thunderbird.feature.account.identity.matcher

import assertk.assertThat
import assertk.assertions.isFalse
import assertk.assertions.isTrue
import kotlin.test.BeforeTest
import kotlin.test.Test

class CatchAllPatternMatcherTest {
    private lateinit var testSubject: CatchAllPatternMatcher

    @BeforeTest
    fun setUp() {
        testSubject = CatchAllPatternMatcher()
    }

    @Test
    fun `wildcard local part matches any address at domain`() {
        // Arrange
        val pattern = "*@domain.com"
        val address = "anything@domain.com"

        // Act
        val result = testSubject.matches(pattern, address)

        // Assert
        assertThat(result).isTrue()
    }

    @Test
    fun `wildcard local part does not match subdomain`() {
        // Arrange
        val pattern = "*@domain.com"
        val address = "user@sub.domain.com"

        // Act
        val result = testSubject.matches(pattern, address)

        // Assert
        assertThat(result).isFalse()
    }

    @Test
    fun `plus addressing pattern matches correctly`() {
        // Arrange
        val pattern = "user+*@domain.com"
        val address = "user+tag@domain.com"

        // Act
        val result = testSubject.matches(pattern, address)

        // Assert
        assertThat(result).isTrue()
    }
}
```

**Note**: Uses `kotlin.test` annotations for multiplatform compatibility (commonTest).

## Appendix B: Relevant Desktop Code References

See `.claude/00-desktop-analysis.md` for:
- Complete GetBestIdentity algorithm
- Pattern matching implementation details
- Header extraction logic
- Test case specifications (TC-001 to TC-512)
