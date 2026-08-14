# Oracle to PostgreSQL Migration — Consolidated Summary & Key Learnings

## Project Context

This consolidated transformation definition captures patterns learned from migrating 5 Java/Kotlin enterprise applications from Oracle Database to PostgreSQL. Combined migration scope: ~500+ files across all repositories.

- **Languages:** Java 8+, Kotlin (JVM)
- **Frameworks:** Spring Boot 2.x/3.x, Spring Batch, Hibernate 3.x/4.x, JPA, Doma2
- **Build Tools:** Gradle, Maven
- **Testing:** JUnit 4/5, DBUnit, DbSetup, AssertJ, Testcontainers
- **Migration Tools:** Flyway
- **SQL Patterns:** Embedded SQL via StringBuilder, string concatenation, HQL/JPQL, XML-based SQL definitions, Doma2 SQL templates
- **PostgreSQL Extensions:** orafce (for Oracle compatibility functions)

---

## Key Learnings (Ranked by Impact)

### 1. Type Strictness is the #1 Challenge

PostgreSQL is significantly stricter about type coercion than Oracle. The single most frequent change across all codebases was adding explicit type casts. This manifests as runtime errors like:
- "Cannot cast to boolean: -1"
- "operator does not exist: integer = boolean"
- "function date_trunc(unknown, unknown) is ambiguous"
- "invalid input syntax for type integer"
- PSQLException for string parameters on numeric columns

Every parameter binding must match the column type exactly, or an explicit `CAST` must be added. This is the most labor-intensive part of the migration.

- **Doma2 apps**: ~60% of SQL file changes were `CAST(/* param */0 AS INTEGER)` additions
- **Hibernate apps**: Nearly every DAO and test file required type-related fixes

### 2. Oracle DATE ≠ PostgreSQL DATE

The single most impactful difference: Oracle's `DATE` type includes both date and time components (precision to seconds). PostgreSQL's `DATE` type is date-only. The correct PostgreSQL equivalent is `TIMESTAMP(0) WITHOUT TIME ZONE`. Failing to account for this causes silent data truncation.

### 3. Result Set Ordering is Non-Deterministic

Oracle and PostgreSQL use different default collation and storage ordering. Tests that relied on positional access (`list.get(0)`) broke because rows came back in different order.

- **Fix**: Stream-filter lookup instead of index-based access
- **Impact**: 50+ test files per application required this fix
- **This was the single most common test fix pattern**

### 4. JDBC Return Type Differences Are Pervasive

The PostgreSQL JDBC driver returns different Java types than Oracle's:

| SQL Operation | Oracle Returns | PostgreSQL Returns |
|---|---|---|
| nextval (sequence) | BigDecimal | BigInteger / Long |
| COUNT(*) | BigDecimal | Long / BigInteger |
| SUM(integer_col) | BigDecimal | Long / BigInteger |
| SUM(numeric_col) | BigDecimal | BigDecimal |
| Integer column | BigDecimal | Integer |

The safest pattern is casting to `Number` interface and calling `.intValue()` or `.longValue()`.

### 5. orafce Extension Reduces Migration Effort Dramatically

By installing orafce, Oracle functions like `NVL`, `TRUNC`, `TO_CHAR`, `DECODE`, `sysdate()` continue to work with minimal changes.

- **Trade-off**: Creates dependency on orafce but reduces SQL changes by 30-50%
- **Setup**: Add `oracle` to PostgreSQL `search_path`, ahead of the application schema
- **Limitations**: NVL with mixed types fails; DECODE requires explicit casting; search path must include `oracle` schema
- **Privilege requirement**: because `search_path` order determines name resolution, `oracle` and `public` must not be writable by application roles — install orafce as a superuser and `REVOKE CREATE ON SCHEMA public FROM PUBLIC`

### 6. Column Alias Case Sensitivity is a Silent Bug Source

Oracle returns aliases in UPPERCASE. PostgreSQL folds unquoted to lowercase. This causes `NullPointerException` at runtime.

- **Fix**: Quote aliases in SQL or change Java to use lowercase keys
- **Impact**: Silent failures that only appear at runtime

### 7. DUAL Table Does Not Exist

Oracle's `DUAL` dummy table is not available in PostgreSQL. Queries like `SELECT seq.NEXTVAL FROM DUAL` must be rewritten to `SELECT nextval('seq')` without a `FROM` clause.

### 8. Numeric Scale Preservation

PostgreSQL preserves the declared scale of `NUMERIC` columns. A value stored as `0.1` in a `NUMERIC(3,2)` column is returned as `0.10`. This breaks direct `assertEquals` comparisons in tests. Use `compareTo()` or scale-aware assertion methods.

### 9. Sequence Syntax is Fundamentally Different

Oracle: `sequence_name.NEXTVAL` (dot notation, used as expression)
PostgreSQL: `nextval('sequence_name')` (function call with string argument)

### 10. ROWNUM Conversion is Context-Dependent

ROWNUM replacement is not a simple find-and-replace. The correct PostgreSQL equivalent depends on:
- Whether it's used for top-N queries (→ LIMIT)
- Whether it's used in DELETE statements (→ ctid subquery)
- Whether it's used as an always-true condition (→ WHERE 1=1)
- Whether the comparison is `<` vs `<=` (affects LIMIT value by 1)
- Whether it's used as a row counter in expressions (→ ROW_NUMBER() OVER)

### 11. Outer Join (+) Syntax Requires Full Restructuring

Converting Oracle's `(+)` syntax requires restructuring FROM and WHERE clauses. This is the most complex transformation and most error-prone. Join conditions move from WHERE to ON, and the table list becomes explicit JOIN syntax.

### 12. DELETE and UPDATE Syntax Differences

- Oracle allows `DELETE table_alias WHERE alias.col = ?`. PostgreSQL requires `DELETE FROM table_name WHERE col = ?`
- Oracle allows `UPDATE tbl t1 SET t1.col = value`. PostgreSQL does not allow alias prefix in SET clause

### 13. DML via executeQuery() Fails

PostgreSQL JDBC driver strictly enforces that INSERT/UPDATE/DELETE must use `executeUpdate()`. Oracle's driver was lenient about this.

### 14. HQL DELETE Requires FROM Keyword

Hibernate HQL in PostgreSQL mode requires `delete from EntityName` instead of `delete EntityName`.

### 15. CURRVAL Requires Same-Session NEXTVAL

PostgreSQL's `currval('seq')` only works if `nextval('seq')` was called in the same session. Use `select last_value from seq_name` as a safer alternative.

### 16. PostgreSQL Transaction Semantics

PostgreSQL aborts ALL subsequent statements in a transaction after any error (unlike Oracle which allows continued execution). Code must explicitly `ROLLBACK` before executing new queries after an error.

### 17. CHARACTER Padding is Visible

PostgreSQL `CHARACTER(n)` pads strings with spaces to exactly `n` characters, and this padding is visible to the application. Oracle JDBC drivers often trim automatically. Solutions: change to `CHARACTER VARYING`, add `.trim()` in code, or update assertions.

### 18. Timezone Sensitivity

- Use IANA timezone IDs (`"Asia/Tokyo"`) instead of abbreviations (`"JST"`)
- PostgreSQL with `TIMESTAMP WITHOUT TIME ZONE` stores values as-is without timezone conversion
- Fix test environment timezone configuration rather than hardcoding in production code

### 19. NULL vs Empty String Semantics

Oracle treats `''` as NULL. PostgreSQL distinguishes them.
- **Anti-pattern**: Do NOT replicate Oracle behavior in Java getters
- Handle at query level or data migration level

### 20. Connection Management Requires Attention

PostgreSQL with Testcontainers exhausts connections faster. Transaction caching needs `isActive()` checks.
- **Fix**: HikariCP pool sizing + transaction state verification

### 21. REGEXP_LIKE Not Available in PostgreSQL

Oracle's `REGEXP_LIKE()` must be replaced with PostgreSQL's `~` regex operator (or `~*` for case-insensitive).

### 22. Numeric TRUNC vs Date TRUNC

PostgreSQL has both numeric `TRUNC(number, places)` and date `DATE_TRUNC('unit', date)`. Do NOT convert numeric TRUNC to DATE_TRUNC — only convert date-related TRUNC calls.

### 23. orafce add_months May Require TIMESTAMP WITH TIME ZONE

When using orafce's `add_months()` function, the input may need to be cast to `TIMESTAMP WITH TIME ZONE` for proper operation.

---

## Incremental Migration Strategy

The most successful approach observed:
1. First convert DDL schemas (Flyway migrations)
2. Then update configuration (driver, dialect, datasource)
3. Then fix SQL queries (functions, sequences, type casts)
4. Then fix Java/Kotlin type handling (BigDecimal → BigInteger, parameter binding)
5. Then fix test assertions (type mismatches, scale differences, ordering)
6. Finally address edge cases (padding, timezone, boolean mapping, transaction semantics)

---

## Migration Statistics (Across All Projects)

| Pattern Category | Frequency | Complexity |
|---|---|---|
| Type coercion (CAST additions) | Very High (60%+ of SQL changes) | Medium |
| Result ordering fixes (stream filter) | Very High (50+ test files) | Low |
| BigDecimal → Number type casts | High (30-50 per app) | Medium |
| NVL → COALESCE (where types mismatch) | High (20-30 per app) | Low |
| Sequence syntax conversion | Medium (15-20 per app) | Low |
| ROWNUM → LIMIT | Medium (10-15 per app) | Medium |
| Outer join (+) → ANSI JOIN | Medium (5-10 per app) | High |
| DECODE → CASE | Medium (10-15 per app) | Low |
| LISTAGG/XMLAGG → STRING_AGG | Medium (10+ XML files) | Low |
| TO_DATE → TO_TIMESTAMP | Medium (10+ files) | Low |
| DELETE/UPDATE alias removal | Medium (5-10 per app) | Low |
| Date arithmetic changes | Medium (10-15 per app) | Medium |
| addScalar type hints | Medium (10+ per app) | Low |
| FROM DUAL removal | Medium | Low |
| Column alias case | Medium | Low |
| ORDER BY for determinism | Medium | Low |
| SYSDATE → sysdate()/NOW() | Medium | Low |
| HQL DELETE (add FROM) | Low-Medium (5-10 per app) | Low |
| Reserved word (offset) rename | Low (3-5 per app) | Low |
| Blob → byte[] | Low (5-10 per app) | Medium |
| Entity type changes | Medium | High |
| HQL to native SQL conversion | Low | High |
| Timezone handling | Low | Medium |
| CHAR padding | Low | Low |
| Transaction rollback handling | Low | Medium |
| Connection pool tuning | Low (1 file) | Low |

---

## Technology Stack Compatibility

| Component | Oracle | PostgreSQL |
|---|---|---|
| Database | 11g / 19c | 14+ / 16.x (recommended) |
| JDBC Driver | ojdbc8 19.x-21.x | postgresql 42.7.x |
| ORM | Hibernate 3.x/4.x, Doma2 | Same + PostgreSQL dialect |
| Migration Tool | — | Flyway |
| Test Container | oracle-xe | postgres with orafce (pin by explicit tag or digest) |
| Connection Pool | HikariCP | HikariCP (unchanged) |
| Spring Boot | 2.x / 3.x | Same (unchanged) |
| Compatibility Ext | — | orafce |

---

## Risk Areas

1. **Implicit type conversions** — Most common source of runtime failures
2. **Date/time precision loss** — Silent data corruption if DATE→DATE instead of DATE→TIMESTAMP
3. **Boolean/Integer mismatch** — Application-level errors when ORM maps integers to booleans
4. **Numeric scale in tests** — Test failures that don't indicate real bugs
5. **Timezone differences** — Subtle bugs that only appear in certain environments
6. **CHARACTER padding** — String comparison failures in tests and application logic
7. **Transaction abort semantics** — Cascading failures after first error in transaction
8. **Result set ordering** — Flaky tests without explicit ORDER BY
9. **ROWNUM off-by-one** — `< N` vs `<= N` semantics difference with LIMIT
10. **Column alias case bugs** — Silent NullPointerException at runtime
11. **Connection exhaustion** — HikariCP pool sizing with Testcontainers
12. **Silent encryption downgrade** — the PostgreSQL JDBC driver does not encrypt by default, so a rewritten connection URL without `sslmode` turns a previously encrypted Oracle connection into plaintext with no error
13. **Parameter inlining during SQL restructuring** — flattening a bound parameter into the SQL string to resolve a type mismatch introduces SQL injection into code that was previously safe
