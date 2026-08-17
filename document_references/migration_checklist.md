# Migration Checklist — Consolidated Oracle to PostgreSQL

Use this checklist for each file or module being migrated. Verify each item applies and is addressed.

---

## Pre-Migration Setup

- [ ] Confirm PostgreSQL version 14+ (16.x recommended)
- [ ] Confirm PostgreSQL extensions installed (orafce, etc.)
- [ ] **Container images pinned:** Testcontainers image referenced by explicit tag or digest, never untagged or `latest`; publisher verified for any third-party orafce-bundled image
- [ ] **Container image pinned:** Testcontainers image uses an explicit tag (digest preferred), never untagged or `latest`; third-party orafce images publisher-verified
- [ ] Flyway configured for PostgreSQL migration directory
- [ ] PostgreSQL JDBC driver added to dependencies
- [ ] Oracle JDBC driver removed from runtime dependencies
- [ ] Testcontainers configured with PostgreSQL image (with orafce)
- [ ] `search_path` includes `oracle` schema for orafce functions
- [ ] CI pipeline updated to use PostgreSQL
- [ ] Identify file type: DDL, DAO, Entity/Model, Service, Batch, Test, XML config
- [ ] Check if file has corresponding test class
- [ ] Review database schema for tables/columns referenced in the file

---

## DDL Migration Scripts (CREATE TABLE/INDEX/SEQUENCE)

- [ ] **Data types converted:**
  - [ ] `NUMBER(n,0)` → `INTEGER` or `BIGINT` based on precision
  - [ ] `NUMBER(n,m)` where m > 0 → `NUMERIC(n,m)` (unchanged)
  - [ ] `CHAR(n)` → `CHARACTER(n)`
  - [ ] `VARCHAR2(n)` / `VARCHAR(n)` → `CHARACTER VARYING(n)`
  - [ ] `DATE` → `TIMESTAMP(0) WITHOUT TIME ZONE`
  - [ ] `CLOB` → `TEXT`
  - [ ] `BLOB` / `RAW` → `BYTEA`
  - [ ] `FLOAT` → `DOUBLE PRECISION`
- [ ] **Flag columns remain INTEGER:** `DELETE_FLG`, `VALID_FLG`, and any locale-specific flag-column naming used in the source schema are `INTEGER`, not `BOOLEAN`
- [ ] **Sequences use PostgreSQL syntax:** `NO CYCLE` (not `NOCYCLE`)
- [ ] **PUBLIC SYNONYM removed:** PostgreSQL does not support synonyms
- [ ] **Function-based indexes updated:** `TO_NUMBER()` → `CAST(... AS NUMERIC)`, etc.
- [ ] **Table/column names are lowercase or quoted**

---

## SQL Syntax Transformations

### Sequence Access
- [ ] `sequence_name.NEXTVAL FROM DUAL` → `nextval('sequence_name')`
- [ ] `sequence_name.CURRVAL FROM DUAL` → `currval('sequence_name')` or `select last_value from sequence_name`
- [ ] `TO_CHAR(seq.nextval, 'fmt')` → `TO_CHAR(nextval('seq'), 'fmt')`
- [ ] Remove all `FROM DUAL` clauses (also in UNION queries)

### Row Limiting (ROWNUM)
- [ ] `WHERE ROWNUM <= N` → `LIMIT N` (after ORDER BY)
- [ ] `WHERE ROWNUM < N` → `LIMIT N-1` (off-by-one adjustment)
- [ ] `WHERE ROWNUM = 1` → `LIMIT 1`
- [ ] `WHERE ROWNUM > 0` → `WHERE 1=1`
- [ ] `ROWNUM AS alias` in subquery → `ROW_NUMBER() OVER(ORDER BY ...) AS alias`
- [ ] `DELETE ... AND ROWNUM <= N` → ctid subquery pattern
- [ ] Verify LIMIT is placed after ORDER BY
- [ ] Verify subqueries have aliases

### Join Syntax
- [ ] `table1.col = table2.col(+)` → `LEFT OUTER JOIN table2 ON table1.col = table2.col`
- [ ] Comma-separated FROM tables → explicit INNER JOIN / LEFT JOIN
- [ ] Join conditions moved from WHERE to ON clause
- [ ] Non-join filter conditions remain in WHERE
- [ ] Multiple (+) on same table → combine in single ON clause

### DELETE / UPDATE Syntax
- [ ] `DELETE table_name alias WHERE` → `DELETE FROM table_name WHERE`
- [ ] Remove table aliases from DELETE statements
- [ ] Remove alias prefix from SET clause in UPDATE statements
- [ ] Keep alias in WHERE clause and subqueries of UPDATE

### Oracle Functions
- [ ] `NVL(a, b)` → `COALESCE(a, b)` (or keep if orafce handles it without type errors)
- [ ] `NVL2(a, b, c)` → `CASE WHEN a IS NOT NULL THEN b ELSE c END`
- [ ] `DECODE(...)` → `CASE WHEN ... END` (or keep with orafce + explicit `::character varying` cast)
- [ ] `SYSDATE` → `sysdate()` (orafce) / `NOW()` / `CURRENT_TIMESTAMP`
- [ ] `SYSDATE` in HQL → `sysdate()` (with parentheses) or `CURRENT_DATE`
- [ ] `TRUNC(SYSDATE)` in HQL → `CURRENT_DATE` or named parameter `:today`
- [ ] `TRUNC(date)` → `date::DATE` or `DATE_TRUNC('day', date)` (date context only)
- [ ] `TRUNC(numeric, n)` → keep as-is (PostgreSQL has numeric TRUNC) or use `FLOOR()`
- [ ] `TRUNC(date, 'MM')` → `DATE_TRUNC('month', date::TIMESTAMP)` or `TRUNC(date::DATE, 'MM')` with orafce
- [ ] `ADD_MONTHS(date, n)` → `date + n * INTERVAL '1 month'`
- [ ] `ADD_MONTHS` with orafce → may need `CAST(CAST(:param AS TIMESTAMP) AS TIMESTAMP WITH TIME ZONE)`
- [ ] `MONTHS_BETWEEN(d1, d2)` → `EXTRACT(YEAR FROM AGE(d1,d2))*12 + EXTRACT(MONTH FROM AGE(d1,d2))`
- [ ] `LAST_DAY(date)` → `DATE_TRUNC('month', date) + INTERVAL '1 month' - INTERVAL '1 day'`
- [ ] `TO_DATE(col)` (single arg, truncation) → `CAST(col AS DATE)` or `col::date`
- [ ] `TO_DATE(str, fmt)` → `TO_TIMESTAMP(str, fmt)` when comparing with TIMESTAMP columns
- [ ] `to_date(to_char(col, fmt), fmt)` → `col::date` (simplified)
- [ ] `TO_CHAR(date, fmt)` for comparison → direct date comparison where possible
- [ ] `LISTAGG(col, sep) WITHIN GROUP (ORDER BY x)` → `STRING_AGG(col, sep ORDER BY x)`
- [ ] `XMLAGG/XMLELEMENT/dbms_xmlgen` → `STRING_AGG(...)`
- [ ] `DBMS_LOB.INSTR(col, val, 1, 1) = 1` → `starts_with(col, val)`
- [ ] `dbms_lob.substr(col, len, start)` → `substring(col from start for len)`
- [ ] `REPLACE(col, 'str')` (2 args) → `REPLACE(col, 'str', '')`
- [ ] `DENSE_RANK FIRST / KEEP` → `(ARRAY_AGG(col ORDER BY ...))[1]`
- [ ] `INSTR(str, substr)` → `POSITION(substr IN str)`
- [ ] `REGEXP_LIKE(col, 'pattern')` → `col ~ 'pattern'` (or `~*` for case-insensitive)
- [ ] `concat_ws('', ...)` → `||` operator
- [ ] Oracle Text `CONTAINS()` → `to_tsvector @@ to_tsquery`

### Oracle Hints
- [ ] Remove all `/*+ hint_text */` optimizer hints (or leave as harmless comments)

### Type Casting (CRITICAL — most common change)
- [ ] Boolean parameters compared to INTEGER columns → explicit `CAST` or `CASE WHEN`
- [ ] All Doma2 `/* param */0` or `/* param */1` → `CAST(/* param */0 AS INTEGER)`
- [ ] Date parameters in `DATE_TRUNC` → cast to `TIMESTAMP`
- [ ] String literals for CHARACTER column comparisons (not unquoted numbers)
- [ ] Cross-type comparisons → explicit `::TEXT` or `::INTEGER` casts
- [ ] `TO_CHAR(date, 'D')` in numeric context → `CAST(TO_CHAR(date, 'D') AS INTEGER)`
- [ ] Mixed types in CASE WHEN branches → ensure same type in all branches
- [ ] String-to-integer comparisons → explicit `::integer` cast
- [ ] `to_char()` results compared to numeric → add `::integer` cast

### Date Arithmetic
- [ ] `date + n` (days) → `date::DATE + n` or `date + INTERVAL 'n days'`
- [ ] `TRUNC(date + n - 1)` → `(date + n - INTERVAL '1 day')::DATE`
- [ ] `date1 - date2` (for day count) → `EXTRACT(DAYS FROM (date1 - date2))`
- [ ] `TRUNC(numeric_expr)` → `FLOOR(numeric_expr)` (numeric context only)
- [ ] Dynamic intervals: `SYSDATE - :param` → `CURRENT_DATE - param * INTERVAL '1 day'`
- [ ] Millisecond calculations → move to Java, pass Timestamp parameters

### Subquery Requirements
- [ ] All subqueries in FROM clause have aliases
- [ ] Reserved word aliases renamed (`offset`, `limit`, `user`, `order`)
- [ ] `SELECT count(1)` has alias when used with `addScalar("count", ...)`

---

## Java/Kotlin Code Transformations

### Return Type Changes
- [ ] `(BigDecimal) query.uniqueResult()` for sequences → `(Number)` with `.longValue()`
- [ ] `(BigDecimal) query.uniqueResult()` for COUNT/SUM → `(Number)` with `.intValue()`
- [ ] `rset.getInt(1)` for sequences → `rset.getLong(1)`, widening the consuming field
- [ ] **No silent narrowing:** no `(int)` cast on a sequence value — use `Math.toIntExact()` if the signature cannot change
- [ ] Handle null: `(result == null) ? 0 : ((Number) result).intValue()`
- [ ] Hibernate `ALIAS_TO_ENTITY_MAP`: integer columns return `Integer` not `BigDecimal`

### Parameter Binding
- [ ] `query.setString("param", val)` for INTEGER columns → `query.setInteger("param", Integer.parseInt(val))`
- [ ] `ps.setString(idx, val)` for INTEGER columns → `ps.setInt(idx, Integer.parseInt(val))`
- [ ] `query.setDate("param", date)` for TIMESTAMP → `query.setTimestamp("param", timestamp)`
- [ ] IN clauses: `IN ('2','3')` → `IN (2,3)` when column is INTEGER
- [ ] Null handling: add null checks before `Integer.parseInt()`
- [ ] NUMERIC/DECIMAL columns → `new BigDecimal(value)`

### Statement Execution
- [ ] `stmt.executeQuery()` on INSERT/UPDATE/DELETE → `stmt.executeUpdate()`
- [ ] Statements and result sets closed with try-with-resources, not a trailing `close()` that a thrown exception will skip

### Hibernate addScalar
- [ ] Add explicit type: `query.addScalar("col", Hibernate.BIG_DECIMAL)`
- [ ] Column names in addScalar should be lowercase
- [ ] Alias in addScalar must match column alias in SQL
- [ ] When `COUNT(*)` needs to be `BigDecimal`: use `SELECT CAST(count(*) AS numeric)`

### HQL Syntax
- [ ] `"delete EntityName where ..."` → `"delete from EntityName where ..."`
- [ ] HQL with ROWNUM → convert to native SQL with LIMIT
- [ ] NVL in HQL → always convert to COALESCE

### Entity/Model Changes
- [ ] String fields mapped to INTEGER columns → change to Integer
- [ ] Update getter/setter signatures and all callers
- [ ] Add `@Embeddable` to composite PK classes if missing
- [ ] Add `@Temporal(TemporalType.TIMESTAMP)` to Date fields in PKs
- [ ] Remove empty-string-to-null conversion in getters
- [ ] `java.sql.Blob` → `byte[]`; `rset.getBlob()` → `rset.getBytes()`

### Transaction Management
- [ ] After `setRollbackOnly()`: add explicit `rollback()` before new queries
- [ ] Wrap native SQL queries in transaction with proper commit/rollback
- [ ] Add `.isActive()` check before reusing cached transactions
- [ ] Rollback handlers catch `SQLException`, not bare `Exception`, and do not swallow the failure
- [ ] No credentials, connection strings, or bound parameter values written to logs by new diagnostics

### Timezone
- [ ] `TimeZone.getTimeZone("JST")` → `TimeZone.getTimeZone("Asia/Tokyo")`
- [ ] Use IANA timezone IDs instead of abbreviations

---

## Test File Transformations

### Type Assertions
- [ ] `assertThat(bd, is(new BigDecimal("10")))` → `.compareTo(new BigDecimal("10")), is(0)`
- [ ] `assertEquals(BigDecimal("12"), dbunitValue)` → `assertEquals(12, ((Number) value).intValue())`
- [ ] `BigDecimal.ZERO` → `BigDecimal("0.00")` matching column scale
- [ ] `BigDecimal.ONE` → `BigDecimal("1.00")` matching column scale
- [ ] `assertThat(obj, is(SomeClass.class))` → `assertThat(obj, instanceOf(SomeClass.class))`

### Column Alias Case
- [ ] `result.get("UPPER_CASE")` → `result.get("lower_case")`

### Date/Timestamp Assertions
- [ ] `assertThat(date, is(expected))` → `assertThat(date.getTime(), is(expected.getTime()))`
- [ ] Compare using `.getTime()` (milliseconds) instead of object equality

### Result Ordering
- [ ] Add ORDER BY to queries where assertions depend on order
- [ ] Or use stream filtering: `list.stream().filter(m -> "id".equals(m.getId())).findFirst()`
- [ ] For multibyte/Japanese characters: use stream filter to avoid collation differences

### Test Data
- [ ] No `-1` values for boolean-mapped columns (use `0` or `1` only)
- [ ] Decimal values match column scale (e.g., `0.10` not `0.1`)
- [ ] No non-numeric characters in numeric column test data
- [ ] String literals for CHARACTER column values
- [ ] **No real personal or customer data** introduced or propagated into fixtures — names, addresses, phone numbers, emails, national identifiers, account numbers. Substitute synthetic values and report anything real found in existing fixtures

### Test Infrastructure
- [ ] Testcontainers with PostgreSQL + orafce image
- [ ] Singleton container pattern for performance
- [ ] HikariCP pool: `minimumIdle=20, maximumPoolSize=20, connectionTimeout=60000`
- [ ] Flyway migration runs before tests
- [ ] Split test data for tables without PKs into separate files
- [ ] Add explicit `TestDB.removeData()` cleanup for noPK tables
- [ ] Respect FK constraint order in cleanup (children before parents)
- [ ] Close InputStreams and delete temp files in `@After` methods
- [ ] Add `@Ignore` with descriptive message for tests depending on unmigrated features
- [ ] Prefer method-level `@Ignore` over class-level to maximize coverage
- [ ] Every `@Ignore` added during migration is listed in the output for follow-up triage
- [ ] No test covering authorization, authentication, input validation, or constraint enforcement was ignored without sign-off
- [ ] Add `transaction.commit()` when subsequent assertions need committed data
- [ ] Add `import static org.hamcrest.CoreMatchers.instanceOf;` for class assertions
- [ ] Add `import java.math.BigInteger;` where BigInteger is used

---

## Configuration Files

- [ ] **JDBC driver:** `oracle.jdbc.OracleDriver` → `org.postgresql.Driver`
- [ ] **Connection URL:** `jdbc:oracle:thin:@...` → `jdbc:postgresql://...`
- [ ] **TLS explicitly set:** `sslmode` present on the new URL (`verify-full` + RDS CA bundle for RDS/Aurora) — the driver does NOT encrypt by default
- [ ] **Encryption not downgraded:** if the Oracle side used `TCPS` / `SQLNET.ENCRYPTION_CLIENT`, the PostgreSQL connection is encrypted too
- [ ] **No plaintext credentials:** username/password come from Secrets Manager, Parameter Store, or IAM database authentication — none introduced or carried over into config files
- [ ] **Dialect:** Oracle dialect → PostgreSQL dialect (Hibernate or Doma2)
- [ ] **Search path:** Include `oracle` schema for orafce: `setCurrentSchema("oracle," + schema)`
- [ ] **Search path locked down:** orafce installed by a superuser; `CREATE` revoked from application roles on `oracle` and `public` (earlier schemas in `search_path` can shadow functions)
- [ ] **Testcontainers:** Oracle image → PostgreSQL image (with orafce if needed)

---

## Build Files (Gradle/Maven)

- [ ] Remove: Oracle JDBC (`ojdbc8`, `orai18n`)
- [ ] Remove: Oracle Testcontainer (`testcontainers:oracle-xe`)
- [ ] Add: PostgreSQL JDBC (`org.postgresql:postgresql:42.7.x`)
- [ ] Add: PostgreSQL Testcontainer, pinned (`org.testcontainers:postgresql:1.21.4`)
- [ ] Add: Flyway PostgreSQL module if using Flyway
- [ ] Add: `db.type` system property in test config if needed

---

## Post-Migration Validation

- [ ] **Compile:** `./gradlew compileJava` or `mvn compile` passes
- [ ] **Tests pass:** `./gradlew test` with PostgreSQL Testcontainer
- [ ] **No Oracle syntax remains** (run search patterns below)
- [ ] **No Oracle dependencies remain** in build files
- [ ] **Application starts** with PostgreSQL connection
- [ ] **Flyway migrations run cleanly** on fresh database
- [ ] **No security regression in the diff:** TLS still configured on every non-test connection, no plaintext credential added, no bound parameter turned into a literal or concatenated string, no test silently disabled, no exception handler widened to bare `Exception`

---

## Search Patterns for Remaining Oracle Syntax

```bash
# Sequence dot notation
grep -rn "\.NEXTVAL\|\.CURRVAL\|\.nextval\|\.currval" --include="*.java" --include="*.kt" --include="*.sql"

# DUAL table
grep -rn "FROM.*DUAL\|from.*dual" --include="*.java" --include="*.kt" --include="*.sql"

# Oracle outer join
grep -rn "(+)" --include="*.java" --include="*.kt" --include="*.sql"

# SYSDATE without parentheses
grep -rn "SYSDATE[^(]" --include="*.java" --include="*.kt"

# ROWNUM
grep -rn "ROWNUM\|rownum" --include="*.java" --include="*.kt" --include="*.sql"

# NVL (verify orafce handles or needs conversion)
grep -rn "NVL(" --include="*.java" --include="*.kt" --include="*.sql"

# DECODE
grep -rn "DECODE(" --include="*.java" --include="*.kt" --include="*.sql"

# Oracle JDBC driver
grep -rn "oracle.jdbc\|ojdbc" --include="*.yml" --include="*.yaml" --include="*.properties" --include="*.gradle" --include="*.xml"

# DBMS_ packages
grep -rn "DBMS_\|dbms_" --include="*.java" --include="*.kt"

# XMLAGG / LISTAGG
grep -rn "XMLAGG\|XMLELEMENT\|LISTAGG" --include="*.java" --include="*.kt" --include="*.sql"

# BigDecimal cast on sequence/count results
grep -rn "(BigDecimal).*uniqueResult\|(BigDecimal).*list().get" --include="*.java"

# executeQuery on DML
grep -rn "executeQuery()" --include="*.java" --include="*.kt"

# REGEXP_LIKE
grep -rn "REGEXP_LIKE" --include="*.java" --include="*.kt" --include="*.sql"

# HQL delete without FROM
grep -rn "\"delete [A-Z]" --include="*.java" --include="*.kt"

# TimeZone with 3-letter abbreviation
grep -rn "getTimeZone(\"[A-Z][A-Z][A-Z]\")" --include="*.java" --include="*.kt"

# DELETE without FROM
grep -rn "DELETE [A-Za-z_]+ WHERE" --include="*.java" --include="*.kt"

# java.sql.Blob
grep -rn "java\.sql\.Blob\|getBlob(" --include="*.java" --include="*.kt"

# FROM DUAL
grep -rn "FROM\s+DUAL\|from\s+dual" --include="*.java" --include="*.kt" --include="*.sql"
```

---

## Decision Trees

### ROWNUM Conversion
```
Is ROWNUM used in a DELETE statement?
├── YES → Use ctid subquery pattern
└── NO
    ├── Is it WHERE rownum > 0 (always-true)?
    │   └── Replace with WHERE 1=1
    ├── Is it ROWNUM in an expression (MOD, counter)?
    │   └── Use ROW_NUMBER() OVER (ORDER BY ...)
    ├── Is it WHERE rownum = 1 or <= 1?
    │   ├── Can outer subquery be removed?
    │   │   ├── YES → Remove wrapper, add LIMIT 1
    │   │   └── NO → Move LIMIT 1 inside, add alias
    ├── Is it WHERE rownum <= :N?
    │   └── LIMIT :N
    └── Is it WHERE rownum < :N?
        └── LIMIT :N - 1
```

### SYSDATE Conversion
```
Is the query HQL/JPQL (createQuery)?
├── YES
│   ├── trunc(sysdate)? → CURRENT_DATE
│   └── bare sysdate? → sysdate() or CURRENT_TIMESTAMP
└── NO (native SQL with createSQLQuery)
    ├── orafce available? → sysdate()
    └── No orafce → NOW() or CURRENT_TIMESTAMP
```

### NVL Conversion
```
Is orafce installed?
├── YES
│   ├── Does NVL work without type errors? → Keep as-is
│   └── Type mismatch error? → Convert to COALESCE
└── NO → Convert to COALESCE
    └── Ensure default value type matches column type
```

---

## File Type Priority Order

Process files in this order to minimize cascading issues:

1. **DDL scripts** — Schema conversion first
2. **Configuration files** — Driver, dialect, datasource
3. **Utility/Helper classes** — Date utilities, type converters
4. **DAO/Repository classes** — Primary SQL transformation target
5. **Entity/Model classes** — Type mappings, annotations
6. **Service/Batch classes** — May contain inline SQL
7. **XML SQL definitions** — Externalized queries
8. **Test classes** — Assertion updates (do last, after source is stable)

---

## Quick Reference: SQL Function Mapping

| Oracle | PostgreSQL |
|--------|-----------|
| `NVL(a, b)` | `COALESCE(a, b)` |
| `NVL2(a, b, c)` | `CASE WHEN a IS NOT NULL THEN b ELSE c END` |
| `DECODE(col, v1, r1, default)` | `CASE col WHEN v1 THEN r1 ELSE default END` |
| `SYSDATE` | `sysdate()` (orafce) / `NOW()` / `CURRENT_TIMESTAMP` |
| `TRUNC(date)` | `date::DATE` / `DATE_TRUNC('day', date)` |
| `TRUNC(date, 'MM')` | `DATE_TRUNC('month', date::TIMESTAMP)` |
| `ADD_MONTHS(date, n)` | `date + n * INTERVAL '1 month'` |
| `MONTHS_BETWEEN(d1, d2)` | `EXTRACT(YEAR FROM AGE(d1,d2))*12 + EXTRACT(MONTH FROM AGE(d1,d2))` |
| `LAST_DAY(date)` | `DATE_TRUNC('month', date) + INTERVAL '1 month - 1 day'` |
| `TO_DATE(s, fmt)` | `TO_TIMESTAMP(s, fmt)` (for TIMESTAMP cols) |
| `seq.NEXTVAL FROM DUAL` | `nextval('seq')` |
| `seq.CURRVAL FROM DUAL` | `currval('seq')` / `SELECT last_value FROM seq` |
| `ROWNUM <= n` | `LIMIT n` |
| `ROWNUM < n` | `LIMIT n-1` |
| `REPLACE(s, search)` | `REPLACE(s, search, '')` |
| `table.col = other.col(+)` | `LEFT JOIN other ON table.col = other.col` |
| `LISTAGG(col, sep) WITHIN GROUP (ORDER BY x)` | `STRING_AGG(col, sep ORDER BY x)` |
| `XMLAGG/XMLELEMENT/dbms_xmlgen` | `STRING_AGG(...)` |
| `DBMS_LOB.INSTR(col, val, 1, 1) = 1` | `starts_with(col, val)` |
| `INSTR(str, substr)` | `POSITION(substr IN str)` |
| `REGEXP_LIKE(col, 'pattern')` | `col ~ 'pattern'` |
| `MAX(col) KEEP(DENSE_RANK FIRST ...)` | `(ARRAY_AGG(col ORDER BY ...))[1]` |
| `DELETE table WHERE` | `DELETE FROM table WHERE` |
| `CONTAINS(col, expr)` | `to_tsvector(col) @@ to_tsquery(expr)` |
| `concat_ws('', ...)` | `col1 \|\| col2 \|\| ...` |

---

## Quick Reference: JDBC Return Type Mapping

| Oracle Returns | PostgreSQL Returns | Safe Java Type |
|---|---|---|
| BigDecimal (sequence) | BigInteger / Long | Number |
| BigDecimal (COUNT) | Long / BigInteger | Number |
| BigDecimal (SUM int) | Long / BigInteger | Number |
| BigDecimal (SUM numeric) | BigDecimal | BigDecimal |
| BigDecimal (INTEGER col) | Integer | Number |
| BigDecimal (BIGINT col) | Long | Number |
| String (CHAR padded) | String (CHAR padded with spaces) | String |
| null (empty string) | "" (empty string) | String |
| Blob | byte[] | byte[] |
| DATE (with time) | TIMESTAMP | Timestamp |


---

## Additional Java/Kotlin Code Transformations (Rules 60-71)

### setParameterList Type Matching
- [ ] `setParameterList("param", List<String>)` on INTEGER columns → convert to `List<Integer>` first
- [ ] Cross-reference `@Entity` class field type to determine correct list element type
- [ ] For BIGINT columns: convert to `List<Long>`
- [ ] Add null/empty checks before stream conversion

### Sequence assignNewPk() Pattern
- [ ] `(BigDecimal) em.createSQLQuery("SELECT nextval(...)").uniqueResult()` → handle BigInteger return
- [ ] Use instanceof check: `if (result instanceof BigInteger) { new BigDecimal((BigInteger) result); }`
- [ ] Fallback: use `Number` interface with `toString()` conversion
- [ ] Apply to all `assignNewPk()` / `getNextSequence()` utility methods

### REPLACE() Function Arguments
- [ ] `REPLACE(col, 'search')` (2 args) → `REPLACE(col::TEXT, 'search', '')` (3 args)
- [ ] Cast `CHARACTER(n)` columns to `::TEXT` before passing to REPLACE
- [ ] Verify all REPLACE calls have three arguments

### Reserved Words as Column Names
- [ ] Identify columns named: `limit`, `offset`, `user`, `order`, `group`, `table`, `column`, `end`
- [ ] Double-quote in SQL: `SELECT "limit" FROM table`
- [ ] Update JPA annotations: `@Column(name = "\"limit\"")`
- [ ] Update `addScalar()` calls with quoted names

### Entity Field Type vs Column Type
- [ ] String fields mapped to INTEGER columns → change to `Integer`
- [ ] String fields mapped to BIGINT columns → change to `Long`
- [ ] Update getters/setters and all callers across codebase
- [ ] Verify HQL/JPQL queries using changed fields still compile

### Transaction Abort Cascading
- [ ] After any SQL error, add explicit `rollback()` before new statements
- [ ] In tests: use independent transactions or catch and rollback
- [ ] Debug cascading failures by running tests individually to find root cause
- [ ] First fix type mismatch errors (most common root cause)

### Schema Completeness for Tests
- [ ] Compile list of ALL tables referenced in test data files (Excel/XML)
- [ ] Verify ALL tables exist in PostgreSQL migration DDL
- [ ] Add missing `CREATE TABLE` statements for test-only tables
- [ ] Verify column definitions match test data expectations

### Empty String vs NULL Constraints
- [ ] Review test data for empty string values in NOT NULL columns
- [ ] Change `''` to explicit `NULL` where Oracle semantics intended NULL
- [ ] Change `''` to valid non-empty values where actual string was intended
- [ ] Update CHECK constraints if they assume Oracle empty-string-is-NULL behavior

### sysdate Arithmetic with INTERVAL
- [ ] `sysdate - N` → `CURRENT_TIMESTAMP - interval 'N days'`
- [ ] `sysdate + N` → `CURRENT_TIMESTAMP + interval 'N days'`
- [ ] Do NOT use bare integer arithmetic with TIMESTAMP
- [ ] For variable N: `CURRENT_TIMESTAMP - :param * interval '1 day'`

### LIMIT in Correlated Subqueries
- [ ] `ROWNUM <= 1` in correlated subquery → `LIMIT 1` inside the subquery
- [ ] For EXISTS checks: `WHERE EXISTS (SELECT 1 FROM t WHERE condition LIMIT 1)`
- [ ] Verify LIMIT is not accidentally placed on outer query
- [ ] Check that LIMIT is after WHERE conditions in subquery

### TRUNC(date, 'month') in HQL
- [ ] `trunc(date, 'month')` in HQL → `year(date)` and `month(date)` comparison
- [ ] Or convert HQL to native SQL using `DATE_TRUNC('month', date)`
- [ ] Verify Hibernate dialect supports the replacement functions
- [ ] Test with actual date boundary values (end of month, leap year)

### executeUpdate() on Wrapper Objects
- [ ] Identify custom DB wrapper classes that lack `executeUpdate()`
- [ ] For DML: get underlying `PreparedStatement` and call `executeUpdate()` directly
- [ ] Or update wrapper class to expose `executeUpdate()` method
- [ ] Verify all DML (INSERT/UPDATE/DELETE) uses `executeUpdate()`, not `executeQuery()`

---

## Search Patterns for Rules 60-71

```bash
# setParameterList with potential type mismatch
grep -rn "setParameterList(" --include="*.java" --include="*.kt"

# BigDecimal cast on sequence queries
grep -rn "(BigDecimal).*createSQLQuery.*nextval\|(BigDecimal).*uniqueResult" --include="*.java"

# Two-argument REPLACE
grep -rn "REPLACE([^,]*,[^,]*)" --include="*.java" --include="*.kt" --include="*.sql"

# Reserved words as columns (potential issues)
grep -rn "\"limit\"\|\"offset\"\|\"user\"\|\"order\"\|\"end\"" --include="*.java" --include="*.kt" --include="*.sql"

# Entity String fields that might map to numeric columns
grep -rn "private String.*[Ii]d\|private String.*[Cc]ode\|private String.*[Nn]um" --include="*.java" --include="*.kt"

# sysdate arithmetic without interval
grep -rn "sysdate.*[-+].*[0-9]\|SYSDATE.*[-+].*[0-9]" --include="*.java" --include="*.kt" --include="*.sql"

# TRUNC in HQL context
grep -rn "trunc(.*'month'\|TRUNC(.*'MONTH'" --include="*.java" --include="*.kt"

# Custom wrapper executeUpdate issues
grep -rn "executeUpdate()" --include="*.java" --include="*.kt"
```
