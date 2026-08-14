# Oracle to PostgreSQL Migration — Consolidated Transformation Definition

## Objective

Transform application code, SQL queries, DDL schemas, configuration files, ORM mappings, and test infrastructure from Oracle Database to PostgreSQL. This consolidated transformation definition covers patterns applicable to Java/Kotlin applications using various ORM frameworks (Hibernate 3.x/4.x, Doma2, JPA), build tools (Gradle/Maven), and testing frameworks (JUnit 4/5, DBUnit, Testcontainers, AssertJ).

## Scope

This transformation applies to:
- DDL migration scripts (CREATE TABLE, CREATE SEQUENCE, CREATE INDEX)
- DML and query SQL files (SELECT, INSERT, UPDATE, DELETE)
- Java/Kotlin source files containing embedded SQL (StringBuilder, string concatenation, HQL/JPQL)
- SQL template files (Doma2 `.sql` files)
- XML-based SQL definition files
- Application configuration (datasource, driver, dialect)
- ORM framework dialect and type mapping customization
- Build dependency management (Gradle/Maven)
- Test infrastructure (Testcontainers, DBUnit, assertions)
- Application code handling date/time, numeric types, and boolean/integer mappings
- Entity/Model classes with JPA annotations
- Spring Boot application configuration (YAML/properties)

## Technology Stack Support

| Component | Supported Variants |
|---|---|
| Language | Java 8+, Kotlin (JVM) |
| ORM | Hibernate 3.x/4.x, Doma2, JPA |
| Framework | Spring Boot 2.x/3.x, Spring Batch |
| Build | Gradle, Maven |
| Test | JUnit 4/5, DBUnit, DbSetup, AssertJ, Testcontainers |
| Connection Pool | HikariCP |
| Migration Tool | Flyway |
| Compatibility | orafce extension (optional but recommended) |

## Applicability

This TD was derived from real-world migrations of multiple Java application repositories covering a range of technology combinations:

| Pattern | Technology |
|---|---|
| Legacy ORM | Java / Hibernate 3.x-4.x / Gradle |
| Native SQL heavy | Java / Hibernate / Native SQL / XML queries |
| JPA with orafce | Java / Hibernate / JPA / orafce extension |
| Spring Boot + Doma2 | Java / Spring Boot / Doma2 ORM / Gradle |
| Batch processing | Java / Spring Batch / Hibernate / Maven |

## CRITICAL Rules

1. **Never change business logic** — only change database-specific syntax
2. **Preserve query semantics** — the migrated query must return identical results
3. **Maintain existing code style** — do not reformat unrelated code
4. **Handle numeric type differences** — PostgreSQL JDBC returns different Java types than Oracle
5. **Column name references in Java code must be lowercase** — PostgreSQL returns unquoted identifiers in lowercase
6. **Use orafce extension** where Oracle functions are retained and compatible; replace with native PostgreSQL equivalents when orafce causes type mismatch errors
7. **Never turn a bound parameter into SQL text** — a value that arrives as `?`, `:named`, or a Doma2 `/* param */` binding must still be a binding after transformation. Never inline it as a literal and never concatenate it into the SQL string. Where a transformation needs a type that the driver will not infer, cast the binding (`LIMIT CAST(:maxRows AS INTEGER)`), do not interpolate the value. Rules 6, 10, 33, and 44 restructure SQL around parameters and are the ones most likely to violate this.
8. **Never weaken security posture while migrating** — do not remove TLS settings from a connection string (Rule 24), do not introduce a plaintext credential into source or configuration, and do not broaden database privileges to make a query work. If a migrated statement fails because of a missing grant, report it rather than widening the role.

---

## Transformation Rules

### 1. DDL Data Type Conversions

CRITICAL: Apply the following Oracle-to-PostgreSQL type mappings consistently across all DDL scripts.

| Oracle Type | PostgreSQL Type | Notes |
|---|---|---|
| `NUMBER(n,0)` where n ≤ 9 | `INTEGER` | Standard integers |
| `NUMBER(n,0)` where 10 ≤ n ≤ 18 | `BIGINT` | Large integers |
| `NUMBER(n,m)` where m > 0 | `NUMERIC(n,m)` | Keep precision/scale |
| `NUMBER(1,0)` (flag columns) | `INTEGER` | Do NOT use BOOLEAN for flag columns |
| `CHAR(n)` | `CHARACTER(n)` | Fixed-length strings |
| `VARCHAR2(n)` / `VARCHAR(n)` | `CHARACTER VARYING(n)` | Variable-length strings |
| `CLOB` | `TEXT` | Large text |
| `BLOB` / `RAW(n)` | `BYTEA` | Binary data |
| `DATE` | `TIMESTAMP(0) WITHOUT TIME ZONE` | Oracle DATE includes time component |
| `TIMESTAMP` | `TIMESTAMP WITHOUT TIME ZONE` | Timestamps |
| `FLOAT` | `DOUBLE PRECISION` | Floating point |

IMPORTANT: Oracle `DATE` type stores both date and time. Always convert to `TIMESTAMP(0) WITHOUT TIME ZONE` in PostgreSQL, not to PostgreSQL `DATE` (which is date-only).

IMPORTANT: Do NOT convert integer flag columns (e.g., `DELETE_FLG`, `VALID_FLG`, and any locale-specific flag-column naming convention used in the source schema) to PostgreSQL `BOOLEAN`. Keep them as `INTEGER` to maintain application compatibility.

### 2. Sequence Syntax Conversion

**Oracle:**
```sql
SELECT seq_name.NEXTVAL FROM DUAL
SELECT seq_name.CURRVAL FROM DUAL
```

**PostgreSQL:**
```sql
SELECT nextval('seq_name')
SELECT currval('seq_name')
```

Rules:
- Remove `FROM DUAL` — PostgreSQL does not need a dummy table for `SELECT` without a `FROM` clause
- Convert `seq_name.NEXTVAL` to `nextval('seq_name')`
- Convert `seq_name.CURRVAL` to `currval('seq_name')`
- `TO_CHAR(seq_name.nextval, 'format')` → `TO_CHAR(nextval('seq_name'), 'format')`
- Preserve the sequence name exactly (case-sensitive in PostgreSQL when quoted)
- Update Java result type: Oracle returns `BigDecimal`, PostgreSQL returns `BigInteger` or `Long`
- Use `Number` interface for cross-compatibility: `Number seq = (Number) query.uniqueResult();`

**Oracle CREATE SEQUENCE:**
```sql
CREATE SEQUENCE seq_name MINVALUE 1 MAXVALUE 999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOCYCLE;
```

**PostgreSQL CREATE SEQUENCE:**
```sql
CREATE SEQUENCE seq_name MINVALUE 1 MAXVALUE 999999 INCREMENT BY 1 START WITH 1 CACHE 20 NO CYCLE;
```

Note: `NOCYCLE` → `NO CYCLE` (two words in PostgreSQL).

### 3. FROM DUAL Removal

**Oracle:** `SELECT expression FROM DUAL`
**PostgreSQL:** `SELECT expression`

Rules:
- Remove `FROM DUAL` (case-insensitive) from queries where it is used solely to evaluate expressions or call functions
- Also in UNION queries: `SELECT 1 FROM DUAL UNION SELECT 2 FROM DUAL` → `SELECT 1 UNION SELECT 2`
- If the query framework requires a FROM clause for parsing, test after removal

### 4. Date/Time Function Conversions

| Oracle | PostgreSQL | Notes |
|---|---|---|
| `SYSDATE` | `CURRENT_TIMESTAMP`, `NOW()`, or `sysdate()` (orafce) | Current date/time |
| `TRUNC(date_col)` | `date_col::DATE` or `DATE_TRUNC('day', date_col)` | Truncate to day |
| `TRUNC(date_col, 'MM')` | `DATE_TRUNC('month', date_col::TIMESTAMP)` | Truncate to month |
| `TRUNC(SYSDATE)` in HQL | `CURRENT_DATE` | Date-only in HQL context |
| `ADD_MONTHS(date, n)` | `date + INTERVAL 'n months'` or `date + n * INTERVAL '1 month'` | Add months |
| `MONTHS_BETWEEN(d1, d2)` | `EXTRACT(YEAR FROM AGE(d1, d2)) * 12 + EXTRACT(MONTH FROM AGE(d1, d2))` | Month difference |
| `LAST_DAY(date)` | `(DATE_TRUNC('month', date) + INTERVAL '1 month' - INTERVAL '1 day')` | Last day of month |
| `TO_DATE('str', 'fmt')` | `TO_DATE('str', 'fmt')` or `TO_TIMESTAMP('str', 'fmt')` | Use `TO_TIMESTAMP` for TIMESTAMP columns |
| `TO_CHAR(date, 'fmt')` | `TO_CHAR(date, 'fmt')` | Compatible — verify format masks |

IMPORTANT: When using `DATE_TRUNC` with a parameter variable, explicitly cast the parameter to `TIMESTAMP`:
```sql
DATE_TRUNC('month', CAST(param_value AS TIMESTAMP))
```

IMPORTANT: In HQL context with orafce, use `sysdate()` (with parentheses) instead of bare `sysdate`. orafce registers it as a function, not a keyword.

### 5. Oracle-Specific Function Conversions

| Oracle | PostgreSQL | Notes |
|---|---|---|
| `NVL(expr, default)` | `COALESCE(expr, default)` | Null handling — or use orafce `NVL()` if types match |
| `NVL2(expr, val1, val2)` | `CASE WHEN expr IS NOT NULL THEN val1 ELSE val2 END` | Conditional null |
| `DECODE(expr, v1, r1, ...)` | `CASE expr WHEN v1 THEN r1 ... END` | Conditional logic |
| `ROWNUM` | `LIMIT` / `ROW_NUMBER() OVER()` | Row limiting (see Rule 6) |
| `(+)` outer join syntax | `LEFT/RIGHT OUTER JOIN` | ANSI join syntax (see Rule 7) |
| `SUBSTR(str, pos, len)` | `SUBSTRING(str FROM pos FOR len)` | Or keep `SUBSTR` with orafce |
| `INSTR(str, substr)` | `POSITION(substr IN str)` | String search |
| `LISTAGG(col, sep) WITHIN GROUP (ORDER BY x)` | `STRING_AGG(col, sep ORDER BY x)` | String aggregation |
| `REPLACE(col, 'str')` (2 args) | `REPLACE(col, 'str', '')` | PostgreSQL requires 3 args |
| `DBMS_LOB.INSTR(col, val, 1, 1) = 1` | `starts_with(col, val)` | String prefix check |
| `XMLAGG/XMLELEMENT/dbms_xmlgen` | `STRING_AGG(col, delimiter ORDER BY ...)` | String aggregation |
| `DENSE_RANK FIRST / KEEP` | `(ARRAY_AGG(col ORDER BY sort_cols))[1]` | First value by order |
| `Oracle Text CONTAINS()` | `to_tsvector('simple', col) @@ to_tsquery(...)` | Full-text search |
| `REGEXP_LIKE(col, 'pattern')` | `col ~ 'pattern'` | Regex match (`~*` for case-insensitive) |

CRITICAL: If orafce is installed and `NVL` works without type errors, it can be left as-is. Only convert to COALESCE when type incompatibility causes runtime errors. COALESCE requires both arguments to be the same type or implicitly castable.

### 6. ROWNUM to LIMIT/OFFSET or ROW_NUMBER()

**Pattern A — Simple top-N:**
- `WHERE ROWNUM <= N` → Remove ROWNUM clause, append `LIMIT N` after ORDER BY
- `WHERE ROWNUM = 1` → Append `LIMIT 1`
- `WHERE ROWNUM < N` → Append `LIMIT N-1` (strict less-than means one fewer row)

**Pattern B — Subquery unwrapping:**
```sql
-- Oracle: SELECT * FROM (SELECT ... ORDER BY x) WHERE ROWNUM = 1
-- PostgreSQL: SELECT ... ORDER BY x LIMIT 1
```

**Pattern C — Subquery retained (when outer query references subquery columns):**
```sql
-- PostgreSQL: SELECT * FROM (SELECT ... ORDER BY x LIMIT 1) sub
```
IMPORTANT: PostgreSQL requires subqueries in FROM clause to have an alias (e.g., `sub`, `temp`).

**Pattern D — ROWNUM in DELETE:**
```sql
-- Oracle: DELETE FROM table WHERE condition AND ROWNUM <= :limit
-- PostgreSQL: DELETE FROM table WHERE ctid IN (SELECT ctid FROM table WHERE condition LIMIT :limit)
```

**Pattern E — ROWNUM > 0 (always-true condition):**
```sql
-- Oracle: WHERE rownum > 0 AND ...
-- PostgreSQL: WHERE 1=1 AND ...
```

**Pattern F — ROWNUM as row counter (not for limiting):**
```sql
-- Oracle: SELECT ROWNUM AS ROWNUMBER, cols FROM (subquery ORDER BY ...)
-- PostgreSQL: SELECT row_number() OVER (ORDER BY ...) AS ROWNUMBER, cols FROM (subquery) inner_alias
```

IMPORTANT: ORDER BY must precede LIMIT. When converting, ensure LIMIT is placed inside the correct subquery.
IMPORTANT: In Oracle, `ROWNUM` is applied BEFORE `ORDER BY`. Ensure `LIMIT` is placed AFTER `ORDER BY`.

### 7. Oracle Outer Join (+) → ANSI JOIN

**Pattern:** Replace Oracle proprietary outer join syntax `table1.col = table2.col(+)` with ANSI SQL `LEFT OUTER JOIN`.

Rules:
- Identify all `(+)` markers in WHERE clause
- The table with `(+)` becomes the right side of the LEFT JOIN
- Convert comma-separated FROM tables to explicit JOIN syntax
- Move join conditions from WHERE to the ON clause
- Keep non-join filter conditions in WHERE
- Multiple `(+)` conditions on the same table pair become multiple ON conditions
- When multiple `(+)` conditions reference the same table, combine them in a single ON clause

### 8. DELETE Statement Syntax

**Oracle:** `DELETE table_name alias WHERE alias.col = ?`
**PostgreSQL:** `DELETE FROM table_name WHERE col = ?`

Rules:
- Add `FROM` keyword after `DELETE`
- Remove table aliases from single-table DELETE statements
- PostgreSQL does not support aliases in single-table DELETE

### 9. UPDATE Statement Alias Restrictions

**Oracle:** Allows `UPDATE table alias SET alias.column = value`
**PostgreSQL:** Does NOT allow alias prefix on columns in SET clause

Rules:
- Remove table alias prefix from column names in the SET clause
- `UPDATE tbl t1 SET t1.col = value` → `UPDATE tbl t1 SET col = value`
- The alias can still be used in WHERE clause and subqueries
- Also applies to tuple/multi-column SET: `SET (t1.col1, t1.col2) = (subquery)` → `SET (col1, col2) = (subquery)`

### 10. Date Arithmetic

```sql
-- Oracle: date + number (adds days)
-- PostgreSQL: For DATE type columns, integer addition works: date_col + 1
-- For TIMESTAMP columns, use INTERVAL: date_col + INTERVAL '1 day'
-- Or cast to DATE first: date_col::DATE + 1
```

**Dynamic INTERVAL with bind parameters:**
```sql
-- Oracle: date - number_param (subtracts days)
-- PostgreSQL: CURRENT_DATE - param * INTERVAL '1 day'
```

**Date subtraction for day difference:**
```sql
-- Oracle: date1 - date2 returns number of days
-- PostgreSQL: EXTRACT(DAYS FROM (date1 - date2)) or FLOOR(EXTRACT(EPOCH FROM (date1 - date2)) / 86400)
```

**Month arithmetic with dynamic parameter:**
```sql
-- Oracle: ADD_MONTHS(date, N)
-- PostgreSQL: date + CAST(N || ' months' AS INTERVAL)
```

**Date arithmetic in SQL with millisecond calculations — move to Java:**
```sql
-- Oracle: col BETWEEN :sysDate - :cacheTime/(24*60*60*1000) AND :sysDate
-- PostgreSQL: Compute in Java, pass as parameter
-- Java: Timestamp startTime = new Timestamp(sysDate.getTime() - cacheTime);
-- SQL: col BETWEEN :startTime AND :sysDate
```

### 11. Boolean/Integer Type Mismatch Handling

CRITICAL: PostgreSQL is strict about type matching. When a column is `INTEGER` but the application binds a `Boolean` parameter:

**Pattern A — CAST in SQL (preferred):**
```sql
column_name = CAST(/* paramName */0 AS INTEGER)
```

**Pattern B — CASE WHEN for Boolean-to-Integer:**
```sql
delete_flg = CASE WHEN /* isDeleted */false THEN 1 ELSE 0 END
```

IMPORTANT: Choose one pattern consistently across the project. Pattern A (CAST) is simpler and preferred.

### 12. Cross-Type Comparisons Require Explicit Casting

PostgreSQL does not implicitly convert between types in comparisons:

```sql
-- String column compared with integer: add quotes
WHERE char_col = '12345678'

-- Integer column compared with text: cast
AND text_column = integer_column::TEXT

-- TO_CHAR result used in numeric context: cast
CAST(TO_CHAR(date_col, 'D') AS INTEGER)

-- Mixed types in CASE WHEN branches: ensure same type
CASE WHEN condition THEN CAST(int_col AS VARCHAR) ELSE varchar_col END
```

### 13. Implicit Type Casting in Parameter Binding

PostgreSQL has stricter type checking than Oracle. Parameters bound as String may fail when the column is INTEGER.

Rules:
- `query.setString("param", value)` where column is INTEGER → `query.setInteger("param", Integer.parseInt(value))`
- `ps.setString(idx, value)` where column is INTEGER → `ps.setInt(idx, Integer.parseInt(value))`
- `query.setParameter("param", stringValue)` where column is numeric → use appropriate numeric type
- For IN clauses with numeric columns: `IN ('2','3')` → `IN (2,3)` when column is INTEGER
- For date parameters on TIMESTAMP columns: `query.setDate()` → `query.setTimestamp()`

### 14. Sequence/Aggregate Return Type Handling (BigDecimal → BigInteger)

**Oracle JDBC driver:** Returns `BigDecimal` for most numeric results
**PostgreSQL JDBC driver:** Returns different types based on column type:

| SQL Operation | Oracle Returns | PostgreSQL Returns |
|---|---|---|
| nextval (sequence) | BigDecimal | BigInteger / Long |
| COUNT(*) | BigDecimal | Long / BigInteger |
| SUM(integer_col) | BigDecimal | Long / BigInteger |
| SUM(numeric_col) | BigDecimal | BigDecimal |
| Integer column | BigDecimal | Integer |

**Recommended pattern — use Number interface:**
```java
Number result = (Number) query.uniqueResult();
return (result == null) ? 0 : result.intValue();
```

**When BigDecimal is specifically needed:**
```java
Number result = (Number) query.uniqueResult();
BigDecimal value = new BigDecimal(result.toString());
```

### 15. Hibernate addScalar() for Return Type Control

Use `addScalar()` to explicitly define return types and eliminate instanceof checks:

```java
query.addScalar("aliasName", Hibernate.BIG_DECIMAL);
BigDecimal result = (BigDecimal) query.uniqueResult();
```

Rules:
- Add explicit Hibernate type to `addScalar()` calls when column type is ambiguous
- The alias name in `addScalar()` must match the column alias in the SQL query
- Add column aliases for aggregate functions: `SELECT count(1) as count`
- Column names in `addScalar()` should be lowercase (PostgreSQL default)
- When `COUNT(*)` needs to be `BigDecimal`: use `SELECT CAST(count(*) AS numeric)`

### 16. Column Alias Case Sensitivity

**Oracle:** Returns column names in UPPERCASE by default
**PostgreSQL:** Returns unquoted identifiers in lowercase

Rules:
- In Java code accessing result maps: `map.get("COLUMN_NAME")` → `map.get("column_name")`
- Prefer changing Java code to lowercase over quoting aliases in SQL
- If uppercase is required: `SELECT col AS "UPPER_NAME"` (quoted preserves case)

### 17. executeQuery() → executeUpdate() for DML

PostgreSQL JDBC driver strictly enforces that DML statements must use `executeUpdate()`:
- `stmt.executeQuery()` on INSERT/UPDATE/DELETE → `stmt.executeUpdate()`
- `ps.executeQuery()` on DML → `ps.executeUpdate()`

### 18. HQL DELETE Syntax

Hibernate HQL in PostgreSQL mode requires `delete from EntityName` instead of `delete EntityName`:
```java
// Before: "delete OrderItemTrace where ..."
// After:  "delete from OrderItemTrace where ..."
```

### 19. Oracle Optimizer Hints — Remove or Retain as Comments

Oracle optimizer hints (`/*+ index(table INDEX_NAME) */`) are not recognized by PostgreSQL but are valid SQL comments.

Rules:
- Remove for cleanliness, OR keep as-is (PostgreSQL ignores them as comments)
- No functional impact either way

### 20. Oracle Synonym Removal

PostgreSQL does not support `CREATE PUBLIC SYNONYM`. Remove all synonym statements from migration scripts. Use schema search path (`SET search_path`) if cross-schema access is needed.

When Oracle used synonyms that pointed to underlying tables, SQL queries must reference the actual table names directly.

### 21. Numeric Precision and Scale Differences

PostgreSQL preserves trailing zeros in `NUMERIC` types differently than Oracle:
- Oracle `NUMBER(3,2)` storing `0.1` returns `0.1`
- PostgreSQL `NUMERIC(3,2)` storing `0.1` returns `0.10`

**Test code fix:** Replace direct `assertEquals` with scale-aware comparison:
```java
assertEquals(0, BigDecimal("0.1").compareTo(entity.taxRate));
```

### 22. DBUnit/Test Framework Type Casting

PostgreSQL returns different Java types from JDBC than Oracle:
- Oracle `NUMBER(n,0)` → Java `BigDecimal`
- PostgreSQL `INTEGER` → Java `Integer`
- PostgreSQL `BIGINT` → Java `Long`

**Fix:** Cast DBUnit `getValue()` results:
```java
assertEquals(12, (table.getValue(0, "INT_COLUMN") as Number).toInt())
assertEquals(100L, (table.getValue(0, "BIGINT_COLUMN") as Number).toLong())
```

### 23. CHARACTER Padding Behavior

PostgreSQL `CHARACTER(n)` pads with spaces to exactly `n` characters, and this padding is visible to the application (Oracle JDBC drivers often trim automatically).

**Solutions:**
1. Change column type from `CHARACTER(n)` to `CHARACTER VARYING` if padding is undesirable
2. Add `.trim()` in application code when reading fixed-length columns
3. Update test assertions to account for padded values

### 24. Configuration Changes

**JDBC Driver:**
```yaml
# Before: oracle.jdbc.OracleDriver
# After:  org.postgresql.Driver
```

**Connection URL:**
```yaml
# Before: jdbc:oracle:thin:@host:1521:SID
# After:  jdbc:postgresql://host:5432/dbname?sslmode=verify-full&sslrootcert=/path/to/rds-ca-bundle.pem
```

CRITICAL: The PostgreSQL JDBC driver does **not** negotiate TLS by default (`ssl=false`), while the Oracle deployment being migrated from is often configured for encryption in transit. Emitting a bare `jdbc:postgresql://host:5432/dbname` therefore silently downgrades transport security.

Rules:
- Always set `sslmode` explicitly on the migrated URL. Use `verify-full` with the Amazon RDS CA bundle for RDS and Aurora targets; `require` is the minimum acceptable value elsewhere.
- If the Oracle connection string or `sqlnet.ora` indicated encryption (for example `TCPS`, or `SQLNET.ENCRYPTION_CLIENT`), the PostgreSQL equivalent must be encrypted too. Do not drop the setting because the new URL syntax differs.
- Local Testcontainers connections are the one exception — plaintext to a container on loopback is acceptable. Do not copy that pattern into non-test configuration.

**Credentials:**

CRITICAL: Migration is a common moment for database passwords to be pasted into `application.yml` or a datasource bean. When editing datasource configuration:
- Never write a plaintext username/password into source or configuration files, and never move an existing one into a newly created file.
- Retrieve credentials from AWS Secrets Manager or SSM Parameter Store, or use IAM database authentication for Amazon RDS and Aurora.
- If the Oracle configuration already contained a hardcoded credential, do not carry it across. Leave a placeholder, and report it so it can be moved to a secret store.
- Do not log connection strings, credentials, or bound parameter values when adding diagnostics for type-mismatch debugging.

**Build Dependencies (Gradle):**
```groovy
// Remove: runtimeOnly("com.oracle.database.jdbc:ojdbc8:...")
// Remove: testRuntimeOnly("com.oracle.ojdbc:orai18n:...")
// Remove: testImplementation("org.testcontainers:oracle-xe")
// Add:    runtimeOnly("org.postgresql:postgresql:42.7.8")
// Add:    testImplementation("org.testcontainers:postgresql:1.21.4")
// Add:    testImplementation("org.flywaydb:flyway-core:11.10.3")
// Add:    testImplementation("org.flywaydb:flyway-database-postgresql:11.10.3")
```

### 25. ORM Dialect Configuration

Replace Oracle dialect with PostgreSQL dialect:

**Hibernate:**
```xml
<!-- Before -->
<property name="dialect">org.hibernate.dialect.Oracle10gDialect</property>
<!-- After -->
<property name="dialect">org.hibernate.dialect.PostgreSQLDialect</property>
```

**Doma2 (custom dialect):**
```java
public class CustomPostgresDialect extends PostgresDialect {
    public CustomPostgresDialect() {
        super(new CustomPostgresJdbcMappingVisitor(),
              new PostgresSqlLogFormattingVisitor(),
              new PostgresExpressionFunctions());
    }
}
```

### 26. Testcontainers Migration

Replace Oracle Testcontainer with PostgreSQL Testcontainer. If Oracle compatibility functions are still used in SQL, use an image that bundles the orafce extension.

CRITICAL: pin the container image to an explicit tag, and prefer a digest. An untagged reference resolves to `latest`, so the image can change between runs — builds stop being reproducible, and a compromised or simply incompatible image can be pulled without any change on your side.

```java
// Pin to an explicit version, not an implicit 'latest'
DockerImageName image = DockerImageName.parse("postgres:16.10-alpine");

// Stronger: pin by digest so the exact image is immutable
DockerImageName image = DockerImageName.parse(
    "postgres@sha256:<digest>").asCompatibleSubstituteFor("postgres");
```

Rules:
- Match the image's PostgreSQL major version to the target database version (see the Technology Stack table — 14+ supported, 16.x recommended)
- If using an orafce-bundled image, pin it the same way. Verify the publisher before adopting a third-party image, since orafce images are frequently community-built; building your own from the official `postgres` image plus `CREATE EXTENSION orafce` is the auditable option
- Never reference an image without a tag, and do not use `latest`
- Use singleton container pattern for performance
- Configure HikariCP: `minimumIdle=20, maximumPoolSize=20, connectionTimeout=60000`
- Flyway migration runs before tests

### 27. Search Path Configuration for orafce

When using orafce extension functions (NVL, DECODE, SYSDATE, TO_DATE, TO_CHAR), the `oracle` schema must be in the search path:
```java
pgDs.setCurrentSchema("oracle," + schema);
// Or connection property: options=-c search_path=oracle,schema_name,public
```

IMPORTANT: Include oracle schema BEFORE the application schema.

IMPORTANT — privilege requirement that comes with this ordering: PostgreSQL resolves unqualified names in `search_path` order, so any role able to create objects in an earlier schema can shadow a function the application calls. Putting `oracle` first is required for orafce to work, so the schema must be locked down to compensate:

- Install orafce as a superuser (`rds_superuser` on Amazon RDS and Aurora), not as the application user
- Keep the `oracle` and `public` schemas owned by an administrative role, with `CREATE` revoked from application roles: `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`
- Set `search_path` explicitly on any `SECURITY DEFINER` function rather than inheriting the session value

This is a database configuration task, not a code change — flag it for the platform team rather than attempting it from the application.

### 28. CURRVAL Alternative

PostgreSQL's `currval('seq')` requires `nextval()` to have been called in the same session. When this is not guaranteed:
```sql
-- Use direct sequence table access
SELECT last_value FROM sequence_name
```

### 29. Index Expression Differences

Oracle function-based indexes may need adjustment:
```sql
-- Oracle: CREATE INDEX idx ON table (TO_NUMBER(char_column));
-- PostgreSQL: CREATE INDEX idx ON table (CAST(char_column AS NUMERIC));
```

### 30. Entity/Model Type Changes

When database column types differ between Oracle and PostgreSQL:
- `String` field mapped to INTEGER column → change to `Integer`
- Update getter/setter signatures and all callers
- Add `@Embeddable` to composite PK classes if missing
- Add `@Temporal(TemporalType.TIMESTAMP)` to Date fields in PKs
- `java.sql.Blob` → `byte[]`; `rset.getBlob()` → `rset.getBytes()`

### 31. Empty String vs NULL

Oracle treats empty string as NULL. PostgreSQL correctly distinguishes them.

Rules:
- Do NOT add logic to convert empty strings to NULL in Java getters (anti-pattern)
- Handle at query level or data migration level
- Update test assertions: `assertNull(field)` → `assertThat(field, is(""))` where appropriate

### 32. Transaction Management

PostgreSQL aborts ALL subsequent statements in a transaction after any error (unlike Oracle which allows continued execution):
```java
// After setRollbackOnly(), add explicit rollback before new queries:
connection.getSQLConnection().rollback();
```

IMPORTANT: catch `SQLException`, not `Exception`, around the rollback. A bare `catch (Exception e)` here swallows connection-level failures and leaves the transaction in an unknown state, which is the failure mode this rule exists to prevent:
```java
try {
    connection.getSQLConnection().rollback();
} catch (SQLException e) {
    log.warn("Rollback failed; connection state is unreliable", e);
    throw new IllegalStateException("Rollback failed", e);
}
```
Do not log bound parameter values or connection strings when adding diagnostics here.

### 33. HQL to Native SQL Conversion

When HQL queries use Oracle-specific features that can't be replaced in HQL:
- Change `em.createQuery(hql)` → `em.createSQLQuery(sql)`
- Convert entity names to table names (CamelCase → snake_case with prefix)
- Convert property names to column names (camelCase → snake_case)
- Add `query.addEntity(EntityClass.class)` for result mapping
- Use `query.setMaxResults()` and `query.setFirstResult()` for pagination when possible

### 34. Timezone Handling

- Use IANA timezone IDs: `"Asia/Tokyo"` instead of `"JST"`
- Use `TIMESTAMP WITHOUT TIME ZONE` for columns that stored Oracle `DATE`
- In application code, explicitly specify timezone when required
- In test infrastructure: `TimeZone.setDefault(TimeZone.getTimeZone("Asia/Tokyo"))`

### 35. ORDER BY for Deterministic Query Results

PostgreSQL does not guarantee row order without an explicit ORDER BY. Add `ORDER BY` to queries where test assertions or application logic depend on result order. Alternatively, use order-independent assertions in tests (stream filtering by known identifiers).

### 36. Reserved Word Aliases

Rename table/column aliases that use PostgreSQL reserved words: `offset`, `limit`, `user`, `order`, `group`, `table`, `column`.

### 37. TO_DATE vs TO_TIMESTAMP for Column Type Matching

Oracle's `to_date()` can be compared with both DATE and TIMESTAMP columns. PostgreSQL is strict:
- Use `to_date(str, fmt)` when target column is DATE type
- Use `to_timestamp(str, fmt)` when target column is TIMESTAMP type
- Single-argument `TO_DATE(column)` (Oracle truncation) → `CAST(column AS DATE)` or `column::DATE`

### 38. concat_ws with Empty Separator

Replace `concat_ws('', col1, col2, ...)` with the `||` concatenation operator:
```sql
-- Before: concat_ws('', ' ', t1.col1, t1.col2)
-- After:  ' ' || t1.col1 || t1.col2
```

### 39. Date Column Cast Simplification

Complex Oracle-style date truncation can be simplified:
```sql
-- Before: to_date(to_char(col, 'YYYY/MM/DD'), 'YYYY/MM/DD')
-- After:  col::date
```

### 40. Numeric TRUNC vs Date TRUNC Distinction

IMPORTANT: Do NOT change `TRUNC()` used for numeric truncation (integer division). Only change date truncation.

```sql
-- Oracle numeric TRUNC (keep as-is or use FLOOR)
TRUNC(value * ratio, 0)   -- PostgreSQL has numeric TRUNC
FLOOR(value * ratio)       -- Alternative for positive numbers
```

Only convert `TRUNC()` to `DATE_TRUNC()` when the argument is a date/timestamp.

### 41. REGEXP_LIKE → PostgreSQL ~ Regex Operator

Oracle's `REGEXP_LIKE()` function is not available in PostgreSQL. Use the `~` regex operator:

```sql
-- Oracle: REGEXP_LIKE(column, 'pattern')
-- PostgreSQL: column ~ 'pattern'

-- Oracle: REGEXP_LIKE(column, 'pattern', 'i')  (case-insensitive)
-- PostgreSQL: column ~* 'pattern'
```

### 42. TIMESTAMP WITH TIME ZONE Casting for orafce Functions

When using orafce's `add_months()` with date parameters, it may require `TIMESTAMP WITH TIME ZONE`:

```sql
-- Oracle: add_months(:param, N)
-- PostgreSQL with orafce:
add_months(CAST(CAST(:param AS TIMESTAMP) AS TIMESTAMP WITH TIME ZONE), N)
```

Alternative: Use `setString("param", formattedDate)` instead of `setTimestamp("param", date)` for date parameters passed to orafce functions.

### 43. TO_CHAR Date Comparisons → Direct Date Comparison

When Oracle code uses `TO_CHAR(date, 'format')` solely for comparison (not display), simplify to direct date comparison in PostgreSQL:

```sql
-- Oracle: TO_CHAR(h.end_date, 'yyyy/mm/dd') <= :param
-- PostgreSQL: h.end_date <= CAST(:param AS DATE)
```

Keep `TO_CHAR` only when the result is used for display/output, not comparison.

### 44. Comma-Separated FROM → Explicit JOIN (Non-Outer-Join)

Convert Oracle-style implicit joins (comma-separated tables in FROM) to explicit ANSI JOIN syntax:

```sql
-- Oracle: FROM t1, t2, t3 WHERE t1.id = t2.id AND t2.id = t3.id
-- PostgreSQL: FROM t1 INNER JOIN t2 ON t1.id = t2.id INNER JOIN t3 ON t2.id = t3.id
```

Rules:
- Move all join predicates to ON clauses
- Keep filter predicates in WHERE
- This improves readability and is required when mixing with LEFT JOINs

### 45. ResultSet.getInt() → getLong() for Sequences

When reading sequence values from ResultSet, PostgreSQL returns BIGINT:

```java
// Before (Oracle): rset.getInt(1)
// After (PostgreSQL): rset.getLong(1)
```

IMPORTANT: prefer widening the consuming field to `long` over casting back to `int`. `(int) rset.getLong(1)` wraps silently past 2147483647 — sequences in this migration are commonly declared `MAXVALUE 9223372036854775807`, so the result is duplicate or negative primary keys appearing long after cutover, with no error at the point of truncation.

If the consuming signature genuinely cannot change, make the truncation explicit and fail loudly:
```java
long seqValue = rset.getLong(1);
int id = Math.toIntExact(seqValue);  // throws ArithmeticException instead of wrapping
```

### 46. Timestamp Casting Cleanup — Remove Unnecessary Casts

Remove unnecessary explicit timestamp casts that were Oracle-specific when the JDBC driver handles type conversion:

```java
// Before (unnecessary cast): " t1.SHIPPING_DATE >= ?::timestamp(0)"
// After (let driver handle):  " t1.SHIPPING_DATE >= ? "
```

Only keep explicit casts when there is a genuine type mismatch that the driver cannot resolve.

### 47. DECODE with orafce — Explicit Type Casting

When orafce's `decode` function is available but column types differ from Oracle:

```java
// Before (Oracle): decode(m1.PRODUCT_ID, ...)
// After (PostgreSQL with orafce): decode(m1.PRODUCT_ID::character varying, ...)
```

Add explicit `::character varying` cast when the column type changed during migration.

### 48. Doma2 Timestamp Literal Syntax

When using Doma2 SQL templates with timestamp literals:

```sql
-- Before (Oracle): WHERE start_date = /* startDate */timestamp'2020-01-02 00:00:00'
-- After (PostgreSQL): WHERE start_date = /* startDate */'2020-01-02 00:00:00'::timestamp
```

### 49. translate() Function Cast

PostgreSQL's `translate()` may require explicit `::text` cast on the input:

```sql
-- Oracle: translate(value, '0123456789', '          ')
-- PostgreSQL: translate(value::text, '0123456789', '          ')
```

### 50. Test Patterns — Date/Timestamp Object Comparison

PostgreSQL returns Timestamp objects with different precision. Direct `.equals()` comparison fails:

```java
// Before (Oracle — direct comparison works):
assertThat(result.getCreatedAt(), is(DateUtil.toDate("20140101")));

// After (PostgreSQL — compare millisecond values):
assertThat(result.getCreatedAt().getTime(), is(DateUtil.toDate("20140101").getTime()));
```

### 51. Test Patterns — Class Type Assertion (is → instanceOf)

When comparing Class objects, Oracle and PostgreSQL may return different proxy types:

```java
// Before (Oracle): assertThat(bean.getModelBean(), is(SomeClass.class))
// After (PostgreSQL): assertThat(bean.getModelBean(), instanceOf(SomeClass.class))
// Add import: import static org.hamcrest.CoreMatchers.instanceOf;
```

### 52. Test Patterns — Result Order Independence via Stream Filtering

When tests access results by index and PostgreSQL returns different order:

```java
// Before (fragile — depends on Oracle ordering):
Entity item = resultList.get(0);
assertThat(item.getId(), is("expected-id"));

// After (robust — order-independent):
Entity item = resultList.stream()
    .filter(m -> "expected-id".equals(m.getId()))
    .findFirst().get();
assertThat(item.getId(), is("expected-id"));
```

### 53. Test Patterns — Collation Differences for Multibyte Characters

PostgreSQL's default collation may sort strings (especially Japanese/multibyte characters) differently than Oracle:

- Use stream filtering by known identifiers instead of positional list access
- Add explicit `COLLATE` clause if exact ordering is required
- Use numeric/date ordering instead of string ordering where possible

### 54. Test Patterns — Data Split for Tables Without Primary Keys

Test frameworks using Excel-based data loading (DbUnit-style) rely on PKs for cleanup. Tables without PKs need special handling:

```java
private static TestData testData;       // Tables with PKs
private static TestData testData_noPK;  // Tables without PKs

@AfterClass
public static void cleanUp() {
    TestDB.removeData("tbl_no_pk_table", "condition_col IN ('val1','val2')");
    testData.cleanup();
}
```

IMPORTANT — applies to every rule that edits test data (this rule, plus Rules 66 and 67 and the numeric-value fixes in Rules 13 and 22): Oracle-era fixtures in long-lived enterprise applications are frequently extracts of production data. When editing a fixture, do not introduce or propagate real personal or customer data — names, addresses, phone numbers, email addresses, national identifiers, or account numbers. Substitute synthetic values.

If you encounter what appears to be real personal data in an existing fixture, do not copy it into a new or split fixture file, and report it. Widening a fixture's reach (splitting it, duplicating rows for a noPK table) increases the exposure of whatever is already in it.

### 55. Test Patterns — @Ignore for Unmigrated Features

Add `@Ignore` annotation with descriptive messages for tests that cannot run against PostgreSQL:

```java
@Ignore("PostgreSQL migration: table X does not exist")
@Test
public void testFeature() { ... }
```

Prefer method-level `@Ignore` over class-level to maximize test coverage.

IMPORTANT: every `@Ignore` added during migration is coverage that silently disappeared. Report them rather than leaving them to be discovered later. At the end of the transformation, emit a list of every test ignored, with the file, test name, and reason, so the team can triage it as follow-up work.

Flag explicitly, and do not ignore without sign-off, any test whose name or assertions relate to authorization, authentication, input validation, or constraint and uniqueness enforcement. Disabling one of those hides a security control regression behind a migration message.

### 56. Test Patterns — Resource Cleanup in Containerized Environments

PostgreSQL test containers may have stricter resource management. Explicitly clean up in `@After` methods:

```java
@After
public void after() throws IOException {
    if (action != null && action.getInputStream() != null) {
        action.getInputStream().close();
    }
}
```

### 57. Test Patterns — Transaction Commit for Visibility

PostgreSQL's stricter transaction isolation may require explicit commit in tests:

```java
updatedItem = dao.update(entity);
transaction.commit();
// Now assertions can see the committed data
```

### 58. Test Patterns — BigDecimal Scale in Assertions

PostgreSQL preserves declared scale. Use scale-aware comparisons:

```kotlin
// AssertJ recursive comparison with BigDecimal comparator
assertThat(actual).usingRecursiveComparison()
    .withComparatorForType({ a, b -> a.compareTo(b) }, BigDecimal::class.java)
    .isEqualTo(expected)
```

### 59. Import Cleanup After Transformations

After all transformations, update imports:
- Add `import java.math.BigInteger;` where BigInteger is used
- Add `import java.math.BigDecimal;` where BigDecimal is newly used
- Add `import java.sql.Timestamp;` where Timestamp is newly used
- Add `import static org.hamcrest.CoreMatchers.instanceOf;` for test assertions
- Remove unused imports (e.g., `import java.sql.Blob;` if no longer used)

---

### 60. setParameterList() with Integer Columns — Cross-Reference Entity Types

CRITICAL: When using Hibernate's `setParameterList()` with a `List<String>` but the database column is INTEGER/BIGINT, PostgreSQL throws `operator does not exist: integer = character varying`.

**Detection Strategy:**
1. Find `setParameterList("paramName", listVariable)` calls
2. Locate the SQL column in the `IN (:paramName)` clause
3. Find the `@Entity` class mapped to that table
4. Check the Java field type for that column — if numeric (`int`, `long`, `Integer`, `Long`, `BigDecimal`), the parameter list must match

```java
// BEFORE (fails — List<String> against INTEGER column):
query.setParameterList("idList", stringList);

// AFTER (convert to matching type):
List<Integer> intList = stringList.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
query.setParameterList("idList", intList);
```

Also for single parameters:
```java
// BEFORE: query.setString("id", value) where column is INTEGER
// AFTER:  query.setInteger("id", Integer.parseInt(value))
```

This single issue cascades to 100+ failures via "current transaction is aborted, commands ignored until end of transaction block".

### 61. BigInteger to BigDecimal Cast for Sequence Results in Entity assignNewPk()

PostgreSQL's `nextval()` returns `BigInteger`, not `BigDecimal`. Direct cast fails with `ClassCastException`.

```java
// BEFORE (fails):
BigDecimal newpk = (BigDecimal) em.createSQLQuery(sql).uniqueResult();

// AFTER (handles both types):
Object result = em.createSQLQuery(sql).uniqueResult();
BigDecimal newpk;
if (result instanceof BigInteger) {
    newpk = new BigDecimal((BigInteger) result);
} else if (result instanceof Number) {
    newpk = new BigDecimal(((Number) result).toString());
} else {
    newpk = (BigDecimal) result;
}
```

### 62. REPLACE() Function — PostgreSQL Requires Three Arguments

Oracle's `REPLACE(string, search)` with two args removes occurrences. PostgreSQL requires three args. Error: `function replace(character, unknown) does not exist`.

```sql
-- BEFORE: TRIM(REPLACE(column, '-'))
-- AFTER:  TRIM(REPLACE(column::TEXT, '-', ''))
```

If column is `CHARACTER(n)` type, cast to TEXT first.

### 63. Reserved Words as Column Names — Quote with Double Quotes

PostgreSQL reserved words (`limit`, `offset`, `user`, `order`, `group`, `table`, `column`, `end`) used as column names must be double-quoted in all SQL.

```java
// In entity: @Column(name = "\"limit\"")
// In SQL:    SELECT "limit" FROM table
```

### 64. Entity Field Type vs Database Column Type Mismatch

When entity has String field but DB column is INTEGER, Hibernate throws: `column "col_name" is of type integer but expression is of type character varying`.

```java
// BEFORE: private String regionId;  // mapped to INTEGER column
// AFTER:  private Integer regionId;
```

Update getters, setters, and all callers.

### 65. PostgreSQL Transaction Abort Cascading

After ANY SQL error in a transaction, ALL subsequent statements fail with: `current transaction is aborted, commands ignored until end of transaction block`. Fix the root cause error first. In tests, use independent transactions or rollback after errors.

### 66. DBUnit NoSuchTableException — Schema Completeness

All tables in test data files must exist in PostgreSQL schema. Compile a list from all `.xls` test files and ensure DDL covers them all.

### 67. NOT NULL Constraint Differences — Oracle Empty String = NULL

Oracle treats `''` as NULL. PostgreSQL does not. Test data with empty strings may violate NOT NULL constraints.

### 68. sysdate Arithmetic to CURRENT_TIMESTAMP with INTERVAL

```sql
-- BEFORE: order_date > sysdate - 30
-- AFTER:  order_date > CURRENT_TIMESTAMP - interval '30 days'
```

### 69. LIMIT Placement in Subqueries

When converting `ROWNUM` in correlated subqueries, LIMIT goes inside the subquery. For existence checks, use `EXISTS` pattern instead.

### 70. TRUNC(date, 'month') in HQL

```java
// BEFORE: "trunc(t1.date, 'month') = trunc(t2.date, 'month')"
// AFTER:  "year(t1.date) = year(t2.date) and month(t1.date) = month(t2.date)"
```

### 71. executeUpdate() Must Be Called on PreparedStatement, Not Wrapper Objects

Custom DB wrapper classes may only expose `execute()` and `executeQuery()`, not `executeUpdate()`. DML must call `executeUpdate()` on the underlying `PreparedStatement`.

```java
// BEFORE (wrapper doesn't have executeUpdate):
stmt.executeUpdate();  // COMPILATION ERROR

// AFTER (call on PreparedStatement, closed via try-with-resources):
try (PreparedStatement ps = connection.getSQLConnection().prepareStatement(sql)) {
    ps.setString(1, param);
    ps.executeUpdate();
}
```

IMPORTANT: use try-with-resources rather than a trailing `ps.close()`. If `executeUpdate()` throws — and during migration it frequently does, on the type mismatches described in Rules 11-14 — a trailing `close()` never runs. Under the pool sizing in Rule 26 the leaked statements exhaust the pool, turning a type error into an availability failure that looks unrelated.

---

## Execution Instructions

1. Scan all `.java` files in `src/main` and `src/test` directories
2. Scan all `.sql` files (Doma2 templates)
3. Scan all `.xml` files containing SQL queries
4. Apply transformations in priority order (CRITICAL → HIGH → MEDIUM → LOW)
5. For each file, apply ALL applicable rules — do not partially transform
6. After transformation, verify the build compiles successfully
7. Run existing unit tests to validate behavior
8. Fix compilation errors related to type mismatches
9. Fix test failures using patterns from the references

## File Patterns to Transform

- `**/*.sql` — DDL and DML query files
- `**/*.java` — Java source files with embedded SQL
- `**/*.kt` — Kotlin source files with embedded SQL
- `**/*.xml` — XML-based SQL definition files (query-define-*.xml or the project's equivalent naming, applicationContext*.xml, hibernate.cfg.xml)
- `**/application*.yml` — Spring Boot configuration
- `**/build.gradle` / `**/pom.xml` — Build dependencies
- `**/testcontainers.properties` — Test container configuration
- `**/*Configuration*` — Database configuration classes
- `**/*Dialect*` — ORM dialect classes
- `**/*Entity*` / `**/*Model*` — Entity/Model classes
- `**/*Dao*` / `**/*Repository*` — Data access classes
- `**/*Test*` — Test files with type assertions

## Files to Skip

- `**/build/**` — Build output
- `**/.gradle/**` — Gradle cache
- `**/gradle/wrapper/**` — Gradle wrapper
- `**/.git/**` — Git internals

## Validation Commands

```bash
# Gradle
./gradlew clean build

# Maven
mvn clean compile

# Test execution
./gradlew test

# With database type property
./gradlew build -Pdb.type=postgresql
```
