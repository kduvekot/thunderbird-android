# Thunderbird Desktop Catch-All Identity Analysis

## Overview

This document analyzes Thunderbird Desktop's catch-all identity feature implementation, based on Bug 1518025 and related source files. The feature allows users to reply from the same email address they received a message on, even when using wildcard/catch-all domains.

## References

- **Primary Bug**: [Bug 1518025](https://bugzilla.mozilla.org/show_bug.cgi?id=1518025) - Allow identity to be marked as "Catch all" address
- **Source Files**:
  - `mailnews/base/public/nsIMsgIdentity.idl` - Identity interface definition
  - `mailnews/base/src/nsMsgAccountManager.cpp` - Account manager with GetBestIdentity
  - `mailnews/compose/test/unit/test_detectIdentity.js` - Test cases for identity detection
  - `mail/components/compose/content/MsgComposeCommands.js` - Compose window commands

---

## Algorithm: GetBestIdentity

### Pseudocode

```
FUNCTION GetBestIdentity(identities[], hint, useDefault=false):
    
    IF identities.length == 0:
        RETURN null
    
    IF identities.length == 1:
        RETURN identities[0]
    
    // Step 1: Try exact match with identity email
    IF hint is not empty:
        hints[] = hint.toLowerCase().split(",")
        
        FOR EACH identity IN identities:
            IF identity.email is empty:
                CONTINUE
                
            email = identity.email.toLowerCase()
            
            FOR EACH h IN hints:
                h = h.trim()
                
                // Exact match: "user@domain.com" or "<user@domain.com>"
                IF h == email OR h.contains("<" + email + ">"):
                    RETURN identity
    
    // Step 2: Try catch-all pattern matching
    IF hint is not empty:
        FOR EACH identity IN identities:
            catchAll = identity.catchAll  // e.g., "*@domain.com"
            
            IF catchAll is not empty:
                // Check if any hint matches the catchAll pattern
                matchingAddress = FindMatchingCatchAllAddress(hints, catchAll)
                
                IF matchingAddress is not empty:
                    // Return identity with matched address as "from hint"
                    RETURN {identity, matchingAddress}
    
    // Step 3: Fallback
    IF useDefault:
        RETURN defaultIdentity
    ELSE:
        RETURN identities[0]
```

### GetIdentityForHeader (Compose Context)

```
FUNCTION GetIdentityForHeader(msgHeader, composeType):
    
    // Step 1: Build hint string from message headers
    hintForIdentity = ""
    
    IF composeType == ReplyToList:
        // Special handling for list replies
        hintForIdentity = FindDeliveredToIdentityEmail(msgHeader)
    
    ELSE IF composeType == Template:
        hintForIdentity = msgHeader.author
    
    ELSE:
        // For Reply, ReplyAll, Forward, etc.
        // Collect addresses from delivery headers
        deliveryHint = GetDeliveryAddressesFromHeaders(msgHeader)
        hintForIdentity = msgHeader.recipients + "," + 
                         msgHeader.ccList + "," + 
                         deliveryHint
    
    // Step 2: Determine server from message
    server = null
    identity = null
    folder = msgHeader.folder
    
    IF folder:
        server = folder.server
        identity = folder.customIdentity  // May be set by user
    
    // Check accountKey header for original receiving account
    accountKey = msgHeader.accountKey
    IF accountKey is not empty:
        account = accountManager.getAccount(accountKey)
        IF account:
            server = account.incomingServer
    
    // Step 3: Get best identity for this server
    IF server AND identity is null:
        identity = GetIdentityForServer(server, hintForIdentity)
    
    // Step 4: If still no match, try all identities
    IF identity is null:
        identity = GetBestIdentity(allIdentities, hintForIdentity, useDefault=true)
    
    RETURN identity
```

### FindMatchingCatchAllAddress (Catch-All Pattern Matching)

```
FUNCTION FindMatchingCatchAllAddress(headers, catchAllPattern):
    // catchAllPattern examples: "*@domain.com", "user+*@domain.com"
    
    // Convert pattern to regex
    // "*" matches any sequence of characters (excluding @)
    regex = PatternToRegex(catchAllPattern)
    
    FOR EACH header IN headers:
        addresses = ParseEmailAddresses(header)
        
        FOR EACH address IN addresses:
            IF regex.matches(address):
                RETURN address
    
    RETURN null

FUNCTION PatternToRegex(pattern):
    // Escape special regex characters except *
    escaped = EscapeRegex(pattern)
    
    // Replace * with [^@]+ (match anything except @)
    regexPattern = escaped.replace("*", "[^@]+")
    
    RETURN new Regex(regexPattern, CASE_INSENSITIVE)
```

---

## Header Checking Order

The catch-all feature checks headers in a specific priority order, controlled by the preference `mail.compose.catchAllHeaders`:

| Priority | Header Name | Description |
|----------|-------------|-------------|
| 1 | `Delivered-To` | Set by final delivery MTA, most reliable |
| 2 | `Envelope-To` | Original envelope recipient |
| 3 | `X-Original-To` | Original recipient before forwarding |
| 4 | `To` | Message To header |
| 5 | `Cc` | Message CC header |

### Header Processing Logic

```
FUNCTION GetDeliveryAddressesFromHeaders(msgHeader):
    catchAllHeaders = Prefs.get("mail.compose.catchAllHeaders")
    // Default: "delivered-to, envelope-to, x-original-to, to, cc"
    
    headerNames = catchAllHeaders.split(",")
    addresses = []
    
    FOR EACH headerName IN headerNames:
        headerName = headerName.trim().toLowerCase()
        
        headerValue = msgHeader.getHeader(headerName)
        IF headerValue:
            // Some headers may have multiple values
            addresses.append(headerValue)
    
    RETURN addresses.join(",")
```

---

## Data Model Requirements

### nsIMsgIdentity Interface Extensions

```idl
interface nsIMsgIdentity : nsISupports {
    // Existing attributes...
    
    /**
     * Catch-all pattern for this identity.
     * When set, this identity will be used for replies to any address
     * matching this pattern.
     * 
     * Pattern format:
     * - "*@domain.com" - matches any local part at domain.com
     * - "prefix+*@domain.com" - matches prefix+anything@domain.com
     * - "user@*" - matches user at any domain (less common)
     *
     * Empty string means catch-all is disabled.
     */
    attribute AString catchAll;
    
    // ... other attributes
};
```

### Identity Storage (Preferences)

```
mail.identity.<id>.catchAll = "*@domain.com"
```

### Related Preferences

| Preference | Type | Default | Description |
|------------|------|---------|-------------|
| `mail.compose.catchAllHeaders` | string | `"delivered-to, envelope-to, x-original-to, to, cc"` | Headers to check for catch-all matching |
| `mailnews.reply_to_self_check_all_ident` | bool | `true` | Check all identities when replying to self |

---

## Test Cases to Port

### 1. Basic Identity Selection

| Test ID | Description | Input | Expected Output |
|---------|-------------|-------|-----------------|
| TC-001 | Select identity by exact email match | Hint: "user@domain.com", Identity email: "user@domain.com" | Identity selected |
| TC-002 | Select identity with angle brackets | Hint: "Name <user@domain.com>", Identity email: "user@domain.com" | Identity selected |
| TC-003 | Case insensitive matching | Hint: "USER@DOMAIN.COM", Identity email: "user@domain.com" | Identity selected |
| TC-004 | Multiple hints, first match | Hint: "other@x.com, user@domain.com", Identity email: "user@domain.com" | Identity selected |
| TC-005 | No match, return first identity | Hint: "unknown@other.com" | First identity returned |
| TC-006 | No match with useDefault flag | Hint: "unknown@other.com", useDefault=true | Default identity returned |
| TC-007 | Empty identity list | Hint: "user@domain.com", identities: [] | null returned |
| TC-008 | Single identity, no hint needed | Hint: "", identities: [id1] | id1 returned |

### 2. Catch-All Pattern Matching

| Test ID | Description | Pattern | Address | Expected |
|---------|-------------|---------|---------|----------|
| TC-101 | Wildcard local part | `*@domain.com` | `anything@domain.com` | Match |
| TC-102 | Wildcard local part, subdomain | `*@domain.com` | `user@sub.domain.com` | No match |
| TC-103 | Plus addressing pattern | `user+*@domain.com` | `user+tag@domain.com` | Match |
| TC-104 | Plus addressing, wrong base | `user+*@domain.com` | `other+tag@domain.com` | No match |
| TC-105 | Domain mismatch | `*@domain.com` | `user@other.com` | No match |
| TC-106 | Case insensitive pattern | `*@Domain.COM` | `USER@domain.com` | Match |
| TC-107 | Empty pattern | `` | `user@domain.com` | No match (disabled) |
| TC-108 | Multiple catch-all identities | id1: `*@a.com`, id2: `*@b.com` | `x@b.com` | id2 selected |

### 3. Header Priority

| Test ID | Description | Headers Present | Expected Source |
|---------|-------------|-----------------|-----------------|
| TC-201 | Only Delivered-To | Delivered-To: user@domain.com | Delivered-To |
| TC-202 | Delivered-To wins over To | Delivered-To: a@d.com, To: b@d.com | Delivered-To (a@d.com) |
| TC-203 | X-Original-To when no Delivered-To | X-Original-To: user@d.com, To: other@d.com | X-Original-To |
| TC-204 | To header as fallback | To: user@domain.com | To |
| TC-205 | Cc header as last resort | Cc: user@domain.com | Cc |
| TC-206 | Multiple Delivered-To | Delivered-To: a@d.com, Delivered-To: b@d.com | First matching one |
| TC-207 | Header with display name | To: "Name" <user@domain.com> | user@domain.com extracted |

### 4. Reply Scenarios

| Test ID | Description | Setup | Expected From |
|---------|-------------|-------|---------------|
| TC-301 | Reply with exact identity match | To: user@domain.com, Identity: user@domain.com | user@domain.com |
| TC-302 | Reply with catch-all match | To: anything@domain.com, CatchAll: *@domain.com | anything@domain.com |
| TC-303 | Reply-all with catch-all | To: list@x.com, Cc: me-alias@domain.com | me-alias@domain.com |
| TC-304 | Reply to BCC'd message | No visible recipient, Delivered-To: secret@domain.com | secret@domain.com |
| TC-305 | Reply to forwarded message | X-Original-To: original@domain.com | original@domain.com |
| TC-306 | Reply-to-list | List-Post: <mailto:list@x.com>, Delivered-To: me@d.com | me@d.com |

### 5. Forward Scenarios

| Test ID | Description | Setup | Expected From |
|---------|-------------|-------|---------------|
| TC-401 | Forward with catch-all | To: anything@domain.com, CatchAll: *@domain.com | anything@domain.com |
| TC-402 | Forward inline | Same as TC-401 | anything@domain.com |
| TC-403 | Forward as attachment | Same as TC-401 | anything@domain.com |

### 6. Edge Cases

| Test ID | Description | Setup | Expected Behavior |
|---------|-------------|-------|-------------------|
| TC-501 | Mailing list in Cc | To: someone@other.com, Cc: list@domain.com | Avoid selecting list address as From |
| TC-502 | Multiple matching identities | id1: *@a.com, id2: *@a.com | First matching identity |
| TC-503 | Catch-all with invalid pattern | CatchAll: "invalid[pattern" | Pattern ignored, fallback to default |
| TC-504 | Address with special chars | To: user+tag/test@domain.com | Correctly parsed and matched |
| TC-505 | Very long local part | To: aaaaaa...aaa@domain.com (256 chars) | Handled without overflow |
| TC-506 | Unicode in email | To: üser@domain.com | Proper handling (if supported) |
| TC-507 | Punycode domain | To: user@xn--nxasmq5b.com | Properly decoded and matched |
| TC-508 | Draft message | savedMsg.From: custom@d.com | Preserve custom From |
| TC-509 | Template message | Template author used | Template author as hint |
| TC-510 | News post | No email header | Default identity used |
| TC-511 | Empty hint string | hint: "" | Default/first identity |
| TC-512 | Whitespace in hint | hint: "  user@domain.com  " | Trimmed and matched |

---

## Implementation Notes for Android

### Key Differences to Consider

1. **Storage**: Android uses SQLite database for account/identity storage instead of preferences file
2. **Threading**: Header parsing must be done on background thread
3. **Memory**: Cache compiled regex patterns for catch-all matching
4. **UI**: Need settings UI to configure catch-all pattern per identity

### Suggested Implementation Order

1. **Phase 1**: Data model
   - Add `catchAll` field to Identity class
   - Add database migration for new field
   - Add UI to edit catch-all pattern in identity settings

2. **Phase 2**: Core algorithm
   - Implement `GetBestIdentity` with catch-all support
   - Implement header extraction from messages
   - Implement pattern matching logic

3. **Phase 3**: Integration
   - Hook into compose flow (reply, forward)
   - Handle "From" address selection based on catch-all
   - Add warning for non-exact identity matches

4. **Phase 4**: Testing
   - Port test cases from desktop
   - Add Android-specific integration tests
   - Test with various email providers (Gmail, Fastmail, etc.)

### Android-Specific Considerations

```kotlin
data class Identity(
    val id: Long,
    val email: String,
    val name: String,
    val replyTo: String?,
    val signature: String?,
    val signatureUse: Boolean,
    // New field for catch-all support
    val catchAll: String? = null  // e.g., "*@domain.com"
)
```

---

## References

### Related Bugs

- Bug 1518025: Main catch-all implementation
- Bug 1652147: Reply-all sets From to original recipient
- Bug 1849531: Catch-all fails with forward
- Bug 1869057: Catch-all fails on x64 version
- Bug 1806065: UI label not clear
- Bug 1659071: Allow * to match subdomains
- Bug 461669: Reply to list auto-detect From
- Bug 1288363: Default account selection fallback

### Source File Locations

```
comm-central/
├── mailnews/
│   ├── base/
│   │   ├── public/
│   │   │   ├── nsIMsgIdentity.idl        # Identity interface
│   │   │   └── nsIMsgAccountManager.idl  # Account manager interface
│   │   └── src/
│   │       └── nsMsgAccountManager.cpp   # GetBestIdentity implementation
│   └── compose/
│       ├── public/
│       │   └── nsIMsgComposeService.idl  # Compose service interface
│       ├── src/
│       │   └── nsMsgCompose.cpp          # Compose implementation
│       └── test/
│           └── unit/
│               └── test_detectIdentity.js # Test cases
└── mail/
    └── components/
        └── compose/
            └── content/
                └── MsgComposeCommands.js  # JS compose commands
```

---

## Appendix: Pattern Matching Examples

### Valid Patterns

| Pattern | Matches | Does Not Match |
|---------|---------|----------------|
| `*@domain.com` | `a@domain.com`, `user@domain.com` | `user@sub.domain.com` |
| `user+*@domain.com` | `user+tag@domain.com`, `user+@domain.com` | `other@domain.com` |
| `*+service@domain.com` | `john+service@domain.com` | `john@domain.com` |

### Pattern Grammar (Simplified)

```
pattern     := local-part "@" domain
local-part  := (char | "*")+
domain      := label ("." label)*
label       := alphanum+
char        := alphanum | "+" | "-" | "_" | "."
alphanum    := [a-zA-Z0-9]
"*"         := wildcard (matches [^@]+)
```
