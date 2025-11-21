# Phase 1 Test Impact Analysis

**Date:** 2025-11-21  
**Status:** Post-Phase 1 Implementation

## Summary

After implementing Phase 1 (core persistence layer changes for case-sensitive usernames/emails), the following test failures are **expected** based on code analysis.

## Expected Test Failures

### Category 1: User Registration Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterTest.java` (9 instances)
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterWithUserProfileTest.java` (2 instances)

**Failure Pattern:**
```java
// OLD CODE (will fail):
assertEquals(email.toLowerCase(), user.getEmail());
assertThat(email.toLowerCase(), is(user.getEmail()));

// EXPECTED BEHAVIOR NOW:
// user.getEmail() returns ORIGINAL case, not lowercase
// Tests comparing with toLowerCase() will fail
```

**Specific Test Methods Likely to Fail:**
- `RegisterTest.registerUserSuccess()` - Line 984: `assertThat(email.toLowerCase(), is(user.getEmail()))`
- Any test that creates a user and verifies email/username with `.toLowerCase()` comparison

**Example Failure:**
```
Expected: john.doe@example.com
Actual: John.Doe@Example.com

AssertionError: Email mismatch - test expects lowercase but user stored original case
```

### Category 2: WebAuthn Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/webauthn/WebAuthnRegisterAndLoginTest.java` (3 instances)

**Failure Pattern:**
```java
// Lines 476-477 will fail:
assertThat(user.getUsername(), is(username.toLowerCase()));
assertThat(user.getEmail(), is(email.toLowerCase()));
```

**Expected Failure:**
- Tests create users with mixed-case usernames/emails
- Assertions expect lowercase but now get original case

### Category 3: Broker/SSO Login Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/KcOidcFirstBrokerLoginTest.java` (7 instances)

**Failure Pattern:**
```java
// Lines 891, 917, 923 will fail:
assertEquals(userRepresentation.getUsername(), expectedBrokeredUserName.toLowerCase());
assertEquals(expectedBrokeredUserName.toLowerCase(), federatedIdentity.getUserName());
```

**Expected Issues:**
- Broker creates users with original case from identity provider
- Tests expect lowercase normalization
- Federation identity linking may fail if case doesn't match

### Category 4: LDAP Federation Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPUserProfileTest.java` (4 instances)
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPMSADFullNameTest.java` (1 instance)
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPSpecialCharsTest.java` (1 instance)

**Failure Pattern:**
```java
// Lines 355, 392, 364, 401:
johnResource = ApiUtil.findUserByUsernameId(testRealm(), upperCaseUsername.toLowerCase());
loginPage.login(upperCaseUsername.toLowerCase(), "Password1");
```

**Expected Issues:**
- LDAP sync may import users with original case
- Lookups using toLowerCase() will fail to find users
- Login with lowercase username may fail if user stored with uppercase

### Category 5: User Storage Provider Tests

**Files Affected:**
- `testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/UserPropertyFileStorage.java` (6 instances)
- `testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/UserMapStorage.java` (3 instances)
- `tests/custom-providers/src/main/java/org/keycloak/testsuite/federation/UserMapStorage.java` (3 instances)

**Failure Pattern:**
```java
// Line 206:
username = user.getUsername().toLowerCase();
```

**Expected Issues:**
- Custom storage providers still using toLowerCase()
- Phase 3 will need to update these
- Tests using these providers will fail

### Category 6: Authorization Policy Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/UserPolicyManagementTest.java` (1 instance)
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/AggregatePolicyManagementTest.java` (1 instance)
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/ClientPolicyManagementTest.java` (1 instance)

**Expected Issues:**
- Authorization policies may reference users by lowercase username
- Policy evaluation may fail if username case doesn't match

### Category 7: User Profile Tests

**Files Affected:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/user/profile/UserProfileTest.java` (2 instances)

**Expected Issues:**
- User profile validation may have case-sensitivity issues
- Attribute comparisons may fail

## Unexpected Failures to Watch For

### 1. Database Constraint Violations

**Risk:** High  
**Description:** If database has unique constraints on lowercase username/email columns:
```sql
-- Existing constraint might be:
CREATE UNIQUE INDEX idx_user_username ON user_entity(LOWER(username), realm_id);

-- With case-sensitive storage, could violate if:
-- User 1: username = "John"
-- User 2: username = "john"
-- Both try to insert, database sees lowercase collision
```

**Mitigation:** Database migration may be required to change constraints.

### 2. Cache Key Collisions

**Risk:** Medium  
**Description:** Old cache entries with lowercase keys won't match new case-sensitive lookups.

**Expected Behavior:**
- Cache misses on first lookup after Phase 1
- Database queries will succeed
- New cache entries created with correct case
- Old cache entries will expire naturally

**Action Required:** Consider cache invalidation after Phase 1 deployment.

### 3. Login Failures

**Risk:** Critical  
**Description:** Existing users stored as lowercase cannot login with mixed case.

**Example:**
```
Database has: username = "john.doe"
User tries: "John.Doe"
Lookup: SELECT * FROM user_entity WHERE username = 'John.Doe'
Result: NOT FOUND (database has 'john.doe')
```

**Mitigation Strategy:**
1. **Option A:** Require exact case match (breaking change for users)
2. **Option B:** Add case-insensitive fallback query
3. **Option C:** Data migration to preserve user preferences

### 4. Federation Provider Issues

**Risk:** High  
**Description:** External identity providers (LDAP, SAML, OIDC) may return usernames in different cases.

**Example:**
```
First login: LDAP returns "John.Doe" → user created as "John.Doe"
Second login: LDAP returns "john.doe" → lookup fails, tries to create duplicate
```

**Mitigation:** Phase 3 will address federation providers.

## Testing Strategy

### Immediate Actions (Manual Testing Required)

1. **Test User Registration:**
   ```bash
   # Create user with mixed case
   username: "TestUser"
   email: "Test.User@Example.com"
   
   # Verify stored with original case
   # Verify can login with exact case
   # Verify cannot login with different case
   ```

2. **Test Existing User Login:**
   ```bash
   # If database has lowercase users
   # Verify they can still login
   # Or document migration path
   ```

3. **Test User Search:**
   ```bash
   # Search for "test" should NOT find "Test"
   # Search must be case-sensitive now
   ```

### Phase 2 Prerequisites

Before starting Phase 2 (test fixes), verify:

- [ ] Database schema supports case-sensitive storage
- [ ] No unique constraint violations on real data
- [ ] Cache invalidation strategy defined
- [ ] Migration path for existing users documented

### Regression Testing Checklist

After Phase 2 (test updates):

- [ ] All RegisterTest cases pass
- [ ] WebAuthn tests pass
- [ ] Broker/SSO tests pass
- [ ] LDAP federation tests pass (may need Phase 3)
- [ ] User storage provider tests pass
- [ ] Authorization policy tests pass
- [ ] User profile tests pass

## Estimated Test Failure Count

Based on code analysis:

| Category | Test Files | Expected Failures | Severity |
|----------|------------|-------------------|----------|
| User Registration | 2 | ~10-15 assertions | High |
| WebAuthn | 1 | ~3 assertions | Medium |
| Broker/SSO | 1 | ~7 assertions | High |
| LDAP Federation | 3 | ~6 assertions | High |
| User Storage | 3 | ~12 instances | Medium |
| Authorization | 3 | ~3 assertions | Low |
| User Profile | 1 | ~2 assertions | Low |
| **Total** | **14** | **~43-48** | **Critical** |

## Recommendations

### Immediate (Before Phase 2)

1. **Document Migration Path:**
   - Create guide for existing deployments
   - Define data migration strategy
   - Plan rollback procedure

2. **Database Analysis:**
   - Check existing data for case variations
   - Verify constraint compatibility
   - Plan index updates if needed

3. **Cache Strategy:**
   - Document cache invalidation approach
   - Consider cluster-wide cache clear
   - Plan for gradual transition

### Phase 2 Scope

Focus on updating test assertions to match new behavior:

```java
// OLD:
assertEquals(email.toLowerCase(), user.getEmail());

// NEW:
assertEquals(email, user.getEmail());
```

Do NOT add back toLowerCase() to make tests pass - update tests to expect case-sensitive behavior.

### Phase 3 Considerations

Federation providers need careful handling:
- LDAP may have case-insensitive mode
- Kerberos has protocol-specific case rules  
- SAML/OIDC providers vary by implementation

---

**Next Steps:**

1. Review this analysis
2. Decide on data migration strategy
3. Plan Phase 2 test updates
4. Consider adding configuration option for case-sensitivity mode

**Note:** This analysis is based on static code review. Actual test failures may vary.
