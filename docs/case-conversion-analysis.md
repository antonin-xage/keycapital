# Case Conversion Functions Analysis Report

**Repository:** keycapital (Keycloak fork)  
**Date:** 2025-11-21  
**Analysis Scope:** toLowerCase(), toUpperCase(), equalsIgnoreCase(), builder.lower()

## Executive Summary

This report provides a comprehensive categorization of all **606 instances** of case conversion functions found in the codebase. These functions have been analyzed and grouped into 11 distinct categories based on their purpose and usage context.

### Total Instances by Function Type

| Function | Count | Percentage |
|----------|-------|------------|
| `toLowerCase()` | 284 | 46.9% |
| `equalsIgnoreCase()` | 198 | 32.7% |
| `toUpperCase()` | 95 | 15.7% |
| `builder.lower()` | 29 | 4.8% |
| **Total** | **606** | **100%** |

### Distribution by Category

| Rank | Category | Instances | % of Total |
|------|----------|-----------|------------|
| 1 | String Comparison | 235 | 38.8% |
| 2 | Other/Miscellaneous | 196 | 32.3% |
| 3 | Database Query (Case-Insensitive Search) | 44 | 7.3% |
| 4 | Database Name/Domain Normalization | 29 | 4.8% |
| 5 | Configuration/Enum Parsing | 29 | 4.8% |
| 6 | Logging/Display | 23 | 3.8% |
| 7 | Kerberos/LDAP | 21 | 3.5% |
| 8 | User/Email Normalization | 16 | 2.6% |
| 9 | HTTP Headers | 6 | 1.0% |
| 10 | Constant/Key Generation | 4 | 0.7% |
| 11 | File/OS Detection | 3 | 0.5% |

---

## Detailed Category Analysis

### 1. String Comparison (235 instances)

**Description:** Case-insensitive string comparisons using `equalsIgnoreCase()` or `contains()` with `toLowerCase()`

**Purpose:** These instances handle case-insensitive string matching, primarily for:
- Comparing user inputs
- Matching configuration values
- Validating SAML/OIDC attributes
- Testing framework assertions

**Function Distribution:**
- `equalsIgnoreCase()`: 198 instances (84.3%)
- `toLowerCase()`: 37 instances (15.7%)

**Top Files:**
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/UserAttributeLDAPStorageMapper.java`: 14 instances
- `services/src/main/java/org/keycloak/broker/saml/mappers/UserAttributeMapper.java`: 8 instances
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/idm/store/ldap/LDAPIdentityStore.java`: 8 instances
- `services/src/main/java/org/keycloak/broker/oidc/mappers/UserAttributeMapper.java`: 6 instances
- `testsuite/integration-arquillian/tests/base/src/main/java/org/keycloak/testsuite/arquillian/ContainerInfo.java`: 6 instances

**Example Usage:**
```java
// File: integration/admin-client/src/main/java/org/keycloak/admin/client/CreatedResponseUtil.java
return a.getType().equalsIgnoreCase(b.getType()) && a.getSubtype().equalsIgnoreCase(b.getSubtype());

// File: model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserCacheSession.java
.filter((u) -> username.equalsIgnoreCase(u.getUsername()))
```

---

### 2. Other/Miscellaneous (196 instances)

**Description:** Various case conversion operations that don't fit into specific categories

**Purpose:** This category includes:
- String transformations for display
- Property key generation
- Filename/path manipulation
- Generic string processing
- Test data generation

**Function Distribution:**
- `toLowerCase()`: 147 instances (75.0%)
- `toUpperCase()`: 49 instances (25.0%)

**Top Files:**
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/forms/RegisterTest.java`: 7 instances
- `testsuite/integration-arquillian/tests/base/src/test/java/org/keycloak/testsuite/broker/KcOidcBrokerIdpLinkActionTest.java`: 6 instances
- `themes/src/main/resources/theme/keycloak.v2/login/resources/js/password-policy.js`: 4 instances
- `testsuite/integration-arquillian/servers/auth-server/services/testsuite-providers/src/main/java/org/keycloak/testsuite/federation/UserMapStorage.java`: 4 instances

**Example Usage:**
```java
// File: common/src/main/java/org/keycloak/common/Profile.java
this.key = name().toLowerCase().replaceAll("_", "-");

// File: common/src/main/java/org/keycloak/common/Version.java
Version.RESOURCES_VERSION = Version.VERSION.toLowerCase();
```

---

### 3. Database Query (Case-Insensitive Search) (44 instances)

**Description:** JPA CriteriaBuilder queries using `builder.lower()` or `toLowerCase()` for case-insensitive database searches

**Purpose:** Enable case-insensitive database queries for:
- User searches by username/email
- Resource lookups by name
- Policy searches
- Organization/domain searches

**Function Distribution:**
- `builder.lower()`: 29 instances (65.9%)
- `toLowerCase()`: 15 instances (34.1%)

**Top Files:**
- `model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java`: 11 instances
- `model/jpa/src/main/java/org/keycloak/organization/jpa/JpaOrganizationProvider.java`: 3 instances
- `model/jpa/src/main/java/org/keycloak/authorization/jpa/store/JPAResourceStore.java`: 3 instances
- `model/jpa/src/main/java/org/keycloak/authorization/jpa/store/JPAPolicyStore.java`: 1 instance

**Example Usage:**
```java
// File: model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java
predicates.add(builder.like(builder.lower(root.get(key)), "%" + value.toLowerCase() + "%"));

// File: model/jpa/src/main/java/org/keycloak/organization/jpa/JpaOrganizationProvider.java
namePredicate = builder.like(builder.lower(org.get("name")), "%" + search.toLowerCase() + "%");
```

---

### 4. Database Name/Domain Normalization (29 instances)

**Description:** Normalizing database entity names like domains, realms, organizations, and role names to lowercase

**Purpose:** Ensure consistency in:
- Organization domain names
- Realm names
- Role names
- Identity provider aliases

**Function Distribution:**
- `toLowerCase()`: 29 instances (100%)

**Top Files:**
- `model/jpa/src/main/java/org/keycloak/organization/jpa/JpaOrganizationProvider.java`: 2 instances
- `model/jpa/src/main/java/org/keycloak/connections/jpa/updater/liquibase/custom/JpaUpdate13_0_0_MigrateDefaultRoles.java`: 2 instances
- `services/src/main/java/org/keycloak/organization/OrganizationProvider.java`: 1 instance

**Example Usage:**
```java
// File: model/jpa/src/main/java/org/keycloak/organization/jpa/JpaOrganizationProvider.java
query.setParameter("name", domain.toLowerCase());

// File: model/jpa/src/main/java/org/keycloak/connections/jpa/updater/liquibase/custom/JpaUpdate13_0_0_MigrateDefaultRoles.java
String roleName = Constants.DEFAULT_ROLES_ROLE_PREFIX + "-" + realmName.toLowerCase();
```

---

### 5. Configuration/Enum Parsing (29 instances)

**Description:** Converting string values to uppercase for enum parsing using `Enum.valueOf()`

**Purpose:** Parse configuration strings into Java enum types for:
- Event types
- Migration strategies
- SSL requirements
- Logger levels
- Theme types
- Key usage types

**Function Distribution:**
- `toUpperCase()`: 29 instances (100%)

**Top Files:**
- `services/src/main/java/org/keycloak/events/email/EmailEventListenerProviderFactory.java`: 2 instances
- `services/src/main/java/org/keycloak/events/log/JBossLoggingEventListenerProviderFactory.java`: 2 instances
- `quarkus/runtime/src/test/java/org/keycloak/quarkus/runtime/configuration/TracingConfigurationTest.java`: 4 instances

**Example Usage:**
```java
// File: model/storage-private/src/main/java/org/keycloak/storage/datastore/DefaultExportImportManager.java
realm.setSslRequired(SslRequired.valueOf(rep.getSslRequired().toUpperCase()));

// File: services/src/main/java/org/keycloak/events/email/EmailEventListenerProviderFactory.java
includedEvents.add(EventType.valueOf(i.toUpperCase()));
```

---

### 6. Logging/Display (23 instances)

**Description:** Case conversion for logging, error messages, and display purposes

**Purpose:** Format output for:
- Error messages
- Log entries
- Exception messages
- User-facing display text

**Function Distribution:**
- `toLowerCase()`: 17 instances (73.9%)
- `toUpperCase()`: 6 instances (26.1%)

**Top Files:**
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/mappers/msad/MSADUserAccountControlStorageMapper.java`: 1 instance
- `federation/kerberos/src/main/java/org/keycloak/federation/kerberos/impl/KerberosUsernamePasswordAuthenticator.java`: 1 instance
- `model/infinispan/src/main/java/org/keycloak/spi/infinispan/impl/embedded/CacheConfigurator.java`: 1 instance

**Example Usage:**
```java
// File: federation/kerberos/src/main/java/org/keycloak/federation/kerberos/impl/KerberosUsernamePasswordAuthenticator.java
String message = le.getMessage().toUpperCase();

// File: model/infinispan/src/main/java/org/keycloak/spi/infinispan/impl/embedded/CacheConfigurator.java
throw new RuntimeException("Unable to start Keycloak. '%s' cache must be replicated but is %s".formatted(WORK_CACHE_NAME, cacheMode.friendlyCacheModeString().toLowerCase()));
```

---

### 7. Kerberos/LDAP (21 instances)

**Description:** Case conversions specific to Kerberos and LDAP operations

**Purpose:** Handle case sensitivity in:
- LDAP attribute names
- Kerberos realm names
- LDAP object properties
- Directory service queries

**Function Distribution:**
- `toLowerCase()`: 15 instances (71.4%)
- `toUpperCase()`: 6 instances (28.6%)

**Top Files:**
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/idm/model/LDAPObject.java`: 4 instances
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/idm/store/ldap/LDAPIdentityStore.java`: 3 instances
- `federation/ldap/src/main/java/org/keycloak/storage/ldap/idm/store/ldap/LDAPOperationManager.java`: 2 instances

**Example Usage:**
```java
// File: federation/kerberos/src/main/java/org/keycloak/federation/kerberos/KerberosPrincipal.java
this.realm = parts[1].toUpperCase();

// File: federation/ldap/src/main/java/org/keycloak/storage/ldap/idm/model/LDAPObject.java
name = name.toLowerCase();
```

---

### 8. User/Email Normalization (16 instances)

**Description:** Normalizing usernames and email addresses to lowercase for consistency

**Purpose:** Ensure case-insensitive user identity management:
- Storing usernames in lowercase
- Normalizing email addresses
- User lookups and queries
- Cache key generation

**Function Distribution:**
- `toLowerCase()`: 16 instances (100%)

**Top Files:**
- `model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java`: 4 instances
- `model/storage-private/src/main/java/org/keycloak/storage/UserStorageManager.java`: 2 instances
- `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserAdapter.java`: 2 instances
- `model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserCacheSession.java`: 3 instances

**Example Usage:**
```java
// File: model/jpa/src/main/java/org/keycloak/models/jpa/JpaUserProvider.java
entity.setUsername(username.toLowerCase());

// File: model/infinispan/src/main/java/org/keycloak/models/cache/infinispan/UserAdapter.java
email = email == null ? null : email.toLowerCase();
```

---

### 9. HTTP Headers (6 instances)

**Description:** Normalizing HTTP header names to lowercase

**Purpose:** HTTP header handling following HTTP/2 specification:
- Header storage in maps
- Header lookups
- Header comparison

**Function Distribution:**
- `toLowerCase()`: 6 instances (100%)

**Top Files:**
- `integration/client-cli/admin-cli/src/main/java/org/keycloak/client/cli/util/Headers.java`: 5 instances

**Example Usage:**
```java
// File: integration/client-cli/admin-cli/src/main/java/org/keycloak/client/cli/util/Headers.java
headers.put(header.toLowerCase(), new Header(header, value));

// File: integration/client-cli/admin-cli/src/main/java/org/keycloak/client/cli/util/Headers.java
return headers.get(header.toLowerCase());
```

---

### 10. Constant/Key Generation (4 instances)

**Description:** Generating constants, map keys, or OID mappings in uppercase

**Purpose:** Create standardized identifiers:
- X509 certificate OID mappings
- Environment variable names
- Configuration keys

**Function Distribution:**
- `toUpperCase()`: 4 instances (100%)

**Top Files:**
- `services/src/main/java/org/keycloak/authentication/authenticators/client/X509ClientAuthenticator.java`: 4 instances

**Example Usage:**
```java
// File: services/src/main/java/org/keycloak/authentication/authenticators/client/X509ClientAuthenticator.java
CUSTOM_OIDS.put("2.5.4.5", "serialNumber".toUpperCase());
CUSTOM_OIDS.put("2.5.4.15", "businessCategory".toUpperCase());
```

---

### 11. File/OS Detection (3 instances)

**Description:** Operating system and file system detection

**Purpose:** Platform-specific behavior:
- OS name detection
- System property checks

**Function Distribution:**
- `toLowerCase()`: 3 instances (100%)

**Top Files:**
- `integration/client-cli/admin-cli/src/main/java/org/keycloak/client/cli/util/OsUtil.java`: 1 instance

**Example Usage:**
```java
// File: integration/client-cli/admin-cli/src/main/java/org/keycloak/client/cli/util/OsUtil.java
String os = System.getProperty("os.name").toLowerCase();
```

---

## Key Findings and Observations

### 1. Dominant Usage Patterns

- **String Comparison dominates** with 38.8% of all instances, primarily using `equalsIgnoreCase()`
- **Database operations** (categories 3, 4, 8) account for 14.7% of usage, critical for case-insensitive queries
- **Configuration parsing** consistently uses `toUpperCase()` for enum conversion

### 2. Security-Sensitive Areas

The following categories require careful attention for security:
- **User/Email Normalization**: Ensures consistent user identity
- **Database Query**: Prevents case-sensitivity bypasses
- **HTTP Headers**: Follows security best practices for header handling

### 3. Technology-Specific Patterns

- **JPA Queries**: Predominantly use `builder.lower()` for database portability
- **LDAP/Kerberos**: Have specific case-handling requirements per protocol specs
- **Configuration**: Consistently normalizes to uppercase for enum parsing

### 4. Code Quality Observations

- **Consistent patterns** within each category suggest good architectural decisions
- **Localized usage** in specific packages indicates good separation of concerns
- **Test code** represents a significant portion (many instances in testsuite directories)

---

## Recommendations

### 1. Documentation
- Document the rationale for case normalization in user/email handling
- Add JavaDoc comments to helper methods that perform case conversion
- Create coding guidelines for when to use each type of case conversion

### 2. Potential Improvements
- Consider creating utility methods for common patterns (e.g., `normalizeUsername()`, `normalizeEmail()`)
- Review "Other" category (196 instances) to identify additional patterns
- Ensure consistent locale handling (currently using default locale)

### 3. Security Considerations
- Audit case-insensitive comparisons for potential security bypasses
- Consider using `Locale.ROOT` for predictable case conversion
- Review database collation settings to align with code behavior

---

## Methodology

This analysis was performed using automated code scanning with context-based categorization. Each instance was classified based on:
- Surrounding code context
- Variable names and method calls
- File path and module structure
- Pattern of usage

The categorization algorithm examined:
- Function names and parameters
- Control flow (if/while/for statements)
- Database query builders
- Framework-specific patterns (JPA, LDAP, etc.)

---

## Appendix: Search Patterns Used

```regex
\.toLowerCase\(\)
\.toUpperCase\(\)
\.equalsIgnoreCase\(
builder\.lower\(
```

File types scanned: `.java`, `.js`, `.ts`, `.jsx`, `.tsx`

Total files scanned: 8,147 files
