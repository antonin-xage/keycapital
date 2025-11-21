# Migration Guide: Case-Sensitive Usernames and Emails

**Goal:** Modify Keycloak to support case-sensitive usernames and email addresses.

**Current State:** Usernames and emails are normalized to lowercase throughout the system for case-insensitive matching.

**Impact:** Requires modifications across **48 files** with approximately **100+ code locations**.

---

## Executive Summary

To support case-sensitive usernames and emails, the following system layers must be modified:

1. **Core Persistence Layer** (4 files) - JPA entities, cache adapters
2. **Query & Search Layer** (2 files) - Database queries, search predicates  
3. **User Storage SPI** (3 files) - Storage manager, query methods
4. **Identity Federation** (10 files) - LDAP, Kerberos, SSSD, broker mappers
5. **Authentication & Security** (5 files) - Password policies, token exchange
6. **Admin API** (2 files) - User management endpoints
7. **Test Infrastructure** (22 files) - Unit tests, integration tests

---

## Critical Files Requiring Modification

### Layer 1: Core Persistence (CRITICAL - Must be done first)

#### 1.1 JPA User Provider
**File:** `model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java`

**Lines to modify:**
- Line 118: `entity.setUsername(username.toLowerCase())` → `entity.setUsername(username)`
- Line 144: `username.toLowerCase()` → `username`
- Line 522: `query.setParameter("username", username.toLowerCase())` → `username`
- Line 532: `query.setParameter("email", email.toLowerCase())` → `email`
- Lines 1049, 1051, 1057, 1059, 1094, 1098: Remove `value.toLowerCase()` in predicates

**Impact:** This controls how usernames/emails are stored and queried in the database.

**Critical Note:** Database schema expects lowercase values. You'll need:
- Database migration to preserve case OR convert existing data
- Update named queries: `getRealmUserByUsername`, `getRealmUserByEmail`

#### 1.2 User Cache Session
**File:** `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserCacheSession.java`

**Lines to modify:**
- Line 266: `String username = rawUsername.toLowerCase()` → `String username = rawUsername`
- Line 435: `String email = rawEmail.toLowerCase()` → `String email = rawEmail`
- Line 564: `username = username.toLowerCase()` → (remove normalization)

**Impact:** Cache key generation. Case-sensitive keys needed to avoid collisions.

**Warning:** Existing cache entries will become invalid. Plan cache invalidation strategy.

#### 1.3 User Adapter (Cache)
**File:** `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserAdapter.java`

**Lines to modify:**
- Line 98: `email = email == null ? null : email.toLowerCase()` → remove toLowerCase()
- Line 150: `username = username==null ? null : username.toLowerCase()` → remove toLowerCase()

**Impact:** Cached user objects will preserve original case.

#### 1.4 User Storage Manager
**File:** `model/storage-private/src/main/java/org/keycloak/storage/UserStorageManager.java`

**Lines to modify:**
- Line 487: `.orElseGet(() -> localStorage().addUser(realm, username.toLowerCase()))` → `username`
- Line 591-594: Remove `.toLowerCase()` from search filter predicates (4 locations)
- Line 775: `username.toLowerCase()` → `username`

**Additional lines:** Lines 314, 538 (equalsIgnoreCase comparisons may need review)

---

### Layer 2: Query & Search

#### 2.1 User Query Methods Provider
**File:** `server-spi/src/main/java/org/keycloak/storage/user/UserQueryMethodsProvider.java`

**Lines to modify:**
- Lines 187-188: Remove `.toLowerCase()` from search filters for username and email

**Impact:** Search will become case-sensitive. Consider UX implications.

---

### Layer 3: Identity Federation & External Storage

#### 3.1 LDAP Storage Provider
**File:** `federation/ldap/src/main/java/org/keycloak/storage/ldap/LDAPStorageProviderFactory.java`

**Lines to modify:**
- Line 255: Username normalization for LDAP sync

**Critical:** LDAP directories may have different case rules. Careful testing needed.

#### 3.2 LDAP Attribute Mapper
**File:** `federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/UserAttributeLDAPStorageMapper.java`

**Lines to modify:**
- Lines involving username/email attribute mapping (4 instances)

#### 3.3 LDAP Mappers Comparator
**File:** `federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/LDAPMappersComparator.java`

**Lines to modify:**
- Lines 41, 44, 46, 47: Comparisons for mapper ordering

#### 3.4 Kerberos Federation
**File:** `federation/kerberos/src/main/java/org/keycloak/federation/kerberos/KerberosFederationProvider.java`

**Lines to modify:**
- Lines 245, 282: Username extraction and email generation

**Note:** Kerberos principals may have specific case requirements per RFC.

#### 3.5 SSSD Federation
**File:** `federation/sssd/src/main/java/org/keycloak/federation/sssd/api/Sssd.java`

**Lines to modify:**
- Line 165: Username handling

---

### Layer 4: Authentication & Security Policies

#### 4.1 Password Policies
**Files:**
- `server-spi-private/src/main/java/org/keycloak/policy/NotContainsUsernamePasswordPolicyProvider.java` (Line 27)
- `server-spi-private/src/main/java/org/keycloak/policy/NotEmailPasswordPolicyProvider.java` (Line 27)
- `server-spi-private/src/main/java/org/keycloak/policy/NotUsernamePasswordPolicyProvider.java` (Line 25)

**Impact:** Password validation uses case-insensitive checks. Decide if this should change.

#### 4.2 Identity Brokering
**Files:**
- `server-spi-private/src/main/java/org/keycloak/broker/provider/AbstractIdentityProvider.java` (Line 65)
- `server-spi-private/src/main/java/org/keycloak/broker/provider/BrokeredIdentityContext.java` (Line 197)
- `services/src/main/java/org/keycloak/authentication/authenticators/broker/IdpCreateUserIfUniqueAuthenticator.java` (Line 161)

**Impact:** User linking and account matching during SSO flows.

#### 4.3 Broker Attribute Mappers
**Files:**
- `services/src/main/java/org/keycloak/broker/oidc/mappers/UserAttributeMapper.java` (Lines 89, 110)
- `services/src/main/java/org/keycloak/broker/saml/mappers/UserAttributeMapper.java` (Lines 88, 106)

---

### Layer 5: Admin API

#### 5.1 Realm Admin Resource
**File:** `services/src/main/java/org/keycloak/services/resources/admin/RealmAdminResource.java`

**Lines to modify:**
- Lines 1171, 1182: User lookup in admin operations

---

### Layer 6: Database Migration Considerations

**Named Queries to Review:**
- `getRealmUserByUsername` - Used by JpaUserProvider line 521
- `getRealmUserByEmail` - Used by JpaUserProvider line 531
- Any other queries with username/email parameters

**Database Indexes:**
- Current indexes likely use lowercase or case-insensitive collation
- May need new indexes for case-sensitive lookups
- Performance implications for large user bases

**Data Migration Options:**

1. **Preserve existing lowercase (safest):**
   - Keep existing users lowercase
   - Only new users can have mixed case
   - No data migration needed

2. **Allow retroactive case changes:**
   - Add migration to update existing usernames
   - Risky - could break external integrations
   - Need careful user communication

---

## Test Files Requiring Updates

### Critical Test Files (Must update assertions)

1. **User Registration Tests:**
   - `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterTest.java` (9 instances)
   - `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterWithUserProfileTest.java` (2 instances)

2. **User Storage Tests:**
   - Multiple test storage implementations expect lowercase (6 files)

3. **Federation Tests:**
   - LDAP tests: 3 files
   - Kerberos/SSSD tests may need updates

4. **Broker/SSO Tests:**
   - `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/KcOidcFirstBrokerLoginTest.java` (7 instances)

**Test Update Pattern:**
```java
// Before:
assertEquals(email.toLowerCase(), user.getEmail());

// After:
assertEquals(email, user.getEmail());
```

---

## Implementation Strategy

### Phase 1: Core Changes (Breaking Changes)
1. Update JpaUserProvider (storage layer)
2. Update UserCacheSession (cache layer)
3. Update UserAdapter (cache wrapper)
4. Update UserStorageManager (storage facade)
5. **Run full test suite** - expect failures

### Phase 2: Test Fixes
1. Update all test assertions
2. Update test storage providers
3. Update test data generators
4. Verify test coverage

### Phase 3: Federation & External Storage
1. LDAP integration
2. Kerberos integration
3. SSSD integration
4. Test each federation type

### Phase 4: Security & Policies
1. Review password policies (decide on case-sensitive comparison)
2. Update identity broker flows
3. Review token exchange
4. Security testing

### Phase 5: API & Admin
1. Update admin API operations
2. Test admin console
3. Update REST API tests

---

## Risks & Considerations

### High Risk Areas

1. **Database Performance:**
   - Case-sensitive queries may be slower
   - Index strategy needs review
   - Consider collation changes at DB level

2. **Backward Compatibility:**
   - Existing integrations expect lowercase
   - LDAP sync may create duplicates
   - SSO flows could break

3. **Security:**
   - Username enumeration via case variations
   - Cache poisoning if keys not handled correctly
   - Session fixation risks

4. **User Experience:**
   - Users may create confusing similar usernames (Admin vs admin)
   - Password reset flows
   - Username hints in error messages

### Medium Risk Areas

1. **Federation:**
   - LDAP case rules vary by directory type
   - Kerberos has RFC-defined case handling
   - Active Directory considerations

2. **Caching:**
   - Cache invalidation complexity
   - Potential cache misses
   - Memory usage if more entries

---

## Testing Checklist

- [ ] User creation with mixed case username
- [ ] User creation with mixed case email
- [ ] Login with exact case match
- [ ] Login with wrong case (should fail)
- [ ] Password reset with mixed case email
- [ ] Admin user search (case-sensitive)
- [ ] LDAP user import with mixed case
- [ ] LDAP user sync (prevent duplicates)
- [ ] Kerberos authentication
- [ ] SAML/OIDC broker login
- [ ] Token exchange flows
- [ ] User impersonation
- [ ] Realm import/export
- [ ] Cache invalidation
- [ ] Performance testing (large user base)

---

## Complete File List (48 files)

### Production Code (26 files)

**Model/Persistence (4 files):**
1. model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java
2. model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserCacheSession.java
3. model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserAdapter.java
4. model/storage-private/src/main/java/org/keycloak/storage/UserStorageManager.java

**SPI (4 files):**
5. server-spi/src/main/java/org/keycloak/storage/user/UserQueryMethodsProvider.java
6. server-spi-private/src/main/java/org/keycloak/models/UserModelDefaultMethods.java
7. server-spi-private/src/main/java/org/keycloak/storage/adapter/AbstractInMemoryUserAdapter.java
8. server-spi-private/src/main/java/org/keycloak/broker/provider/AbstractIdentityProvider.java

**Federation (6 files):**
9. federation/ldap/src/main/java/org/keycloak/storage/ldap/LDAPStorageProviderFactory.java
10. federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/UserAttributeLDAPStorageMapper.java
11. federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/LDAPMappersComparator.java
12. federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/HardcodedAttributeMapperFactory.java
13. federation/kerberos/src/main/java/org/keycloak/federation/kerberos/KerberosFederationProvider.java
14. federation/sssd/src/main/java/org/keycloak/federation/sssd/api/Sssd.java

**Services (5 files):**
15. services/src/main/java/org/keycloak/authentication/authenticators/broker/IdpCreateUserIfUniqueAuthenticator.java
16. services/src/main/java/org/keycloak/broker/oidc/mappers/UserAttributeMapper.java
17. services/src/main/java/org/keycloak/broker/saml/mappers/UserAttributeMapper.java
18. services/src/main/java/org/keycloak/protocol/oidc/tokenexchange/AbstractTokenExchangeProvider.java
19. services/src/main/java/org/keycloak/services/resources/admin/RealmAdminResource.java

**Security Policies (4 files):**
20. server-spi-private/src/main/java/org/keycloak/policy/NotContainsUsernamePasswordPolicyProvider.java
21. server-spi-private/src/main/java/org/keycloak/policy/NotEmailPasswordPolicyProvider.java
22. server-spi-private/src/main/java/org/keycloak/policy/NotUsernamePasswordPolicyProvider.java
23. server-spi-private/src/main/java/org/keycloak/broker/provider/BrokeredIdentityContext.java

**Migration (1 file):**
24. model/storage-private/src/main/java/org/keycloak/migration/migrators/MigrateTo1_3_0.java

**Tests/Utils (2 files):**
25. tests/custom-providers/src/main/java/org/keycloak/testsuite/federation/UserMapStorage.java
26. tests/utils/src/main/java/org/keycloak/tests/utils/admin/AdminApiUtil.java

### Test Code (22 files)

**Test Providers (3 files):**
27. testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/BackwardsCompatibilityUserStorage.java
28. testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/UserMapStorage.java
29. testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/UserPropertyFileStorage.java

**Test Utilities (2 files):**
30. testsuite/integration-arquillian/tests/base/src/main/java/org/keycloak/testsuite/admin/ApiUtil.java
31. testsuite/integration-arquillian/tests/base/src/main/java/org/keycloak/testsuite/updaters/UserAttributeUpdater.java

**Integration Tests (12 files):**
32. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/AggregatePolicyManagementTest.java
33. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/ClientPolicyManagementTest.java
34. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/authz/admin/UserPolicyManagementTest.java
35. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/AbstractFirstBrokerLoginTest.java
36. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/KcOidcBrokerLoginHintTest.java
37. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/KcOidcFirstBrokerLoginTest.java
38. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/UserStorageGracefulDegradationTest.java
39. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPMSADFullNameTest.java
40. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPSpecialCharsTest.java
41. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/federation/ldap/LDAPUserProfileTest.java
42. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterTest.java
43. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterWithUserProfileTest.java

**Additional Tests (5 files):**
44. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/user/profile/UserProfileTest.java
45. testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/webauthn/WebAuthnRegisterAndLoginTest.java
46. js/libs/keycloak-admin-client/test/authenticationManagement.spec.ts
47. js/libs/keycloak-admin-client/test/crossRealm.spec.ts
48. js/libs/keycloak-admin-client/test/realms.spec.ts

---

## Recommended Approach

1. **Start with a feature flag** to enable/disable case-sensitive mode
2. **Implement in phases** (core → tests → federation → security)
3. **Extensive testing** at each phase
4. **Document migration path** for existing deployments
5. **Consider making it configurable** per realm

---

## Additional Resources

- See `docs/case-conversion-analysis.md` for complete analysis
- See `docs/case-conversion-instances.csv` for line-by-line details
- Consult Keycloak database schema documentation
- Review LDAP/Kerberos RFCs for case handling requirements

---

**Last Updated:** 2025-11-21  
**Analysis Source:** Automated scan of 8,147 files
