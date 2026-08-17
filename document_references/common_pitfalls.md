# Common Pitfalls & Anti-Patterns — Consolidated

This document captures anti-patterns and mistakes that were encountered and reverted during multiple Oracle-to-PostgreSQL migrations. Avoid these patterns.

---

## Pitfall 1: Converting Oracle DATE to PostgreSQL DATE

### What happened
Initial migration attempted to use PostgreSQL `DATE` type for Oracle `DATE` columns.

### Why it was wrong
Oracle `DATE` stores date AND time (to seconds precision). PostgreSQL `DATE` is date-only. This causes silent loss of time component, broken time-of-day filters, and wrong timestamp comparisons.

### Correct approach
Always convert Oracle `DATE` to `TIMESTAMP(0) WITHOUT TIME ZONE` in PostgreSQL.

---

## Pitfall 2: Strict Type System — String Parameters on Integer Columns

### What happened
Kept `query.setString("param", value)` for columns that are INTEGER in PostgreSQL.

### Why it fails
PostgreSQL does not perform implicit type coercion. Binding a String parameter to an INTEGER column throws `PSQLException: invalid input syntax for type integer`.

### Correct approach
Change to `query.setInteger("param", Integer.parseInt(value))` or `ps.setInt(idx, Integer.parseInt(value))`.

---

## Pitfall 3: Hard-Casting to BigInteger Without Fallback

### What happened
Changed all `(BigDecimal) query.uniqueResult()` to `(BigInteger) query.uniqueResult()`.

### Why it fails
Not all PostgreSQL numeric returns are BigInteger. NUMERIC(p,s) columns still return BigDecimal, INTEGER columns return Integer. Causes ClassCastException for non-sequence queries.

### Correct approach
Use `Number` interface: `Number result = (Number) query.uniqueResult()` then call `.intValue()` or `.longValue()`.

---

## Pitfall 4: Assuming BigDecimal Equality Across Databases

### What happened
Tests using `assertEquals(BigDecimal("0.1"), entity.field)` failed because PostgreSQL returns `BigDecimal("0.10")` for a `NUMERIC(3,2)` column.

### Why it fails
`BigDecimal("0.1").equals(BigDecimal("0.10"))` returns `false` in Java because `equals()` checks both value AND scale.

### Correct approach
Use `compareTo()` instead of `equals()`: `assertEquals(0, expected.compareTo(actual))`.

---

## Pitfall 5: Forgetting to Cast Parameters in DATE_TRUNC()

### What happened
Converting Oracle `TRUNC(param, 'MM')` to PostgreSQL `DATE_TRUNC('month', param)` without casting caused: `"function date_trunc(unknown, unknown) is ambiguous"`

### Correct approach
Always cast the parameter: `DATE_TRUNC('month', CAST(param AS TIMESTAMP))`

---

## Pitfall 6: Mixing Boolean Parameters with Integer Columns

### What happened
Application passed `Boolean` values to SQL queries where the column type was `INTEGER`. PostgreSQL throws type mismatch error.

### Correct approach
Choose ONE consistent pattern: `CAST(/* param */0 AS INTEGER)` or change application to pass `Int` instead of `Boolean`.

---

## Pitfall 7: Using -1 as a Boolean/Flag Value

### What happened
Oracle allowed storing `-1` in integer columns mapped to boolean fields. PostgreSQL JDBC maps: `0 = false`, `1 = true`. Any other value throws: `"Cannot cast to boolean: -1"`

### Correct approach
Update test data to use only `0` and `1` for flag columns. If `-1` is meaningful, change entity field type from `Boolean` to `Int`.

---

## Pitfall 8: Not Accounting for CHARACTER Padding in Assertions

### What happened
Column `CHARACTER(8)` stored "Tokyo" but returned "Tokyo   " (padded). Test assertions expecting "Tokyo" failed.

### Correct approach
1. Change column type to `CHARACTER VARYING` if padding is not required
2. Or update assertions to expect padded values
3. Or add `.trim()` in entity accessor methods

---

## Pitfall 9: ROWNUM < N Converted to LIMIT N (Off-by-One Error)

### What happened
`WHERE ROWNUM < :maxRow` was converted to `LIMIT :maxRow` without adjusting the value.

### Why it was wrong
Oracle's `ROWNUM < N` returns N-1 rows. PostgreSQL's `LIMIT N` returns N rows.

### Correct approach
`ROWNUM < N` → `LIMIT N-1`. `ROWNUM <= N` → `LIMIT N`.

---

## Pitfall 10: Placing LIMIT Outside the Subquery

### What happened
LIMIT was placed where ROWNUM was (outside the subquery), but PostgreSQL applies LIMIT to the immediately preceding SELECT.

### Correct approach
LIMIT must be inside the subquery it restricts.

---

## Pitfall 11: Missing Subquery Alias After Adding LIMIT

### What happened
PostgreSQL requires all subqueries in FROM clause to have an alias. `SELECT * FROM (SELECT ... LIMIT 1)` fails.

### Correct approach
Add alias: `SELECT * FROM (SELECT ... LIMIT 1) sub`

---

## Pitfall 12: Placing ORDER BY After LIMIT

### What happened
ORDER BY was placed after LIMIT, which is syntactically invalid in PostgreSQL.

### Correct approach
Always: `... ORDER BY x LIMIT n` (ORDER BY precedes LIMIT).

---

## Pitfall 13: Removing WHERE rownum > 0 Without Replacement

### What happened
`WHERE rownum > 0` was removed, but subsequent conditions used `AND` without a preceding WHERE clause.

### Correct approach
Replace with `WHERE 1=1` to maintain the query anchor for dynamic condition appending.

---

## Pitfall 14: UNNEST/ARRAY Pattern for IN Clauses (Overly Complex)

### What happened
Used `UNNEST(ARRAY[...])` with CAST for integer IN clauses instead of simply fixing parameter binding.

### Correct approach
Simply change `setString` to `setInt` at the binding point. Fix the parameter type, not the SQL structure.

---

## Pitfall 15: Quoting Column Aliases for Uppercase Map Access

### What happened
Added quoted aliases (`"COLUMN_NAME"`) to force uppercase in result set.

### Why it was reverted
Quoted identifiers are fragile, error-prone, and inconsistent with PostgreSQL conventions.

### Correct approach
Change Java code to use lowercase keys: `map.get("column_name")`.

---

## Pitfall 16: Keeping NVL When Types Don't Match (orafce limitation)

### What happened
Left NVL as-is relying on orafce, but orafce's `nvl` requires both arguments to be the exact same type.

### Correct approach
Use `COALESCE` which handles implicit type promotion. Test each NVL usage with orafce, especially with mixed types.

---

## Pitfall 17: Blindly Replacing NVL with COALESCE When Orafce Is Installed

### What happened
NVL was replaced with COALESCE across a large codebase. The change was later reverted because orafce was installed and NVL was already functional.

### Correct approach
Check if orafce is installed first. If available, NVL can remain unchanged in existing code. Only replace when orafce causes type mismatch errors.

---

## Pitfall 18: Not Handling Sequence Return Type Change

### What happened
Kept Oracle-style `(BigDecimal)` cast after changing SQL to `nextval()`. PostgreSQL returns `BigInteger`, causing ClassCastException.

### Correct approach
Always update Java type handling when changing SQL that affects return types.

---

## Pitfall 19: Using executeQuery() for DML Statements

### What happened
Oracle driver allowed `executeQuery()` for INSERT/UPDATE/DELETE. PostgreSQL throws: "No results were returned by the query"

### Correct approach
Use `executeUpdate()` for all DML statements.

---

## Pitfall 20: Assuming DBUnit Returns BigDecimal for All Numeric Columns

### What happened
Tests using `assertEquals(BigDecimal("12"), table.getValue(0, "INT_COLUMN"))` failed because PostgreSQL returns `Integer` for INTEGER columns.

### Correct approach
Cast to appropriate type: `(table.getValue(0, "INT_COLUMN") as Number).toInt()`

---

## Pitfall 21: Converting Empty Strings to NULL in Getters

### What happened
Added logic to return null for empty strings in entity getters (Oracle compatibility shim).

### Why it fails
PostgreSQL correctly distinguishes empty string from NULL. This breaks downstream logic and changes the contract of the getter method unpredictably.

### Correct approach
Return the field as-is. Handle at query/data level, not in Java getters.

---

## Pitfall 22: Assuming Result Set Order Without ORDER BY

### What happened
Tests asserted specific elements by index without ORDER BY. PostgreSQL makes no ordering guarantee.

### Correct approach
Add ORDER BY to queries, or use order-independent assertions (stream filtering, Map-based lookups).

---

## Pitfall 23: Removing Connection Cleanup Before New Queries After Rollback

### What happened
Removed rollback handling between error and subsequent queries.

### Why it fails
PostgreSQL: after transaction error, ALL subsequent statements fail with "current transaction is aborted" until explicit ROLLBACK.

### Correct approach
Add explicit `connection.getSQLConnection().rollback()` before new operations.

---

## Pitfall 24: Using currval() Without Prior nextval() in Same Session

### What happened
PostgreSQL's `currval()` requires `nextval()` to have been called on the same sequence in the current session.

### Correct approach
Use `select last_value from sequence_name` as a safer alternative.

---

## Pitfall 25: Complex to_date/to_char Round-Trip for Date Truncation

### What happened
Converted Oracle's implicit date truncation to verbose `to_date(to_char(col, 'YYYY/MM/DD'), 'YYYY/MM/DD')`.

### Correct approach
Use PostgreSQL's native `col::date` cast for truncation. Simpler, more readable, more performant.

---

## Pitfall 26: Incorrect search_path Configuration for orafce

### What happened
Set schema without `oracle` in search path. orafce functions not found.

### Correct approach
Include oracle schema BEFORE the application schema: `setCurrentSchema("oracle," + schema)`

### Privilege requirement that comes with it
Because PostgreSQL resolves unqualified names in `search_path` order, a role that can create objects in `oracle` (or `public`) can shadow a function the application calls. Install orafce as a superuser / `rds_superuser`, keep both schemas owned by an administrative role, and `REVOKE CREATE ON SCHEMA public FROM PUBLIC`.

---

## Pitfall 27: Ignoring Foreign Key Constraints in Test Cleanup

### What happened
Simple cleanup that doesn't respect FK order fails with constraint violations.

### Correct approach
Delete in reverse dependency order (children before parents). Use explicit `removeData()` calls with WHERE clauses.

---

## Pitfall 28: TimeZone "JST" Works Differently in PostgreSQL

### What happened
Tests using `TimeZone.getTimeZone("JST")` produced incorrect timestamps.

### Correct approach
Use IANA timezone IDs: `"Asia/Tokyo"` instead of `"JST"`.

---

## Pitfall 29: Changing ZoneId.systemDefault() to Explicit Timezone Prematurely

### What happened
Timezone test failures led to hardcoding `ZoneId.of("Asia/Tokyo")` in production code. Later reverted.

### Correct approach
Fix the test environment timezone configuration. Only hardcode timezone if business requirement demands it.

---

## Pitfall 30: Applying Multiple Incompatible Fix Patterns

### What happened
Different team members applied different patterns for the same problem (e.g., CASE WHEN vs CAST for boolean conversion).

### Correct approach
Agree on ONE pattern per problem type before starting. Document and apply consistently.

---

## Pitfall 31: Changing Business Logic While Migrating SQL

### What happened
Changed comparison operators or logic during migration, introducing bugs.

### Correct approach
CRITICAL: Never change business logic during database migration. Syntax changes only. Logic changes should be separate commits.

---

## Pitfall 32: Not Using orafce When Oracle Functions Are Widespread

### What happened
Attempted to convert every NVL(), TRUNC(), DECODE() to native equivalents — massive changeset with high regression risk.

### Correct approach
Install orafce as a bridge. Gradually migrate to native PostgreSQL functions in subsequent iterations.

---

## Pitfall 33: Using max(id) + 1 Instead of Sequences

### What happened
Replaced sequence calls with `select max(id) from table` + 1 in Java.

### Why it fails
Race condition: concurrent requests get the same max value. Performance: full table scan vs lightweight sequence.

### Correct approach
Always use `select nextval('seq_name')`.

---

## Pitfall 34: Adding CAST Without Updating Java Consumers

### What happened
Added `CAST(col AS VARCHAR)` to SQL but Java code still cast result to BigDecimal.

### Correct approach
When adding CAST, update ALL Java code consuming that column simultaneously.

---

## Pitfall 35: String Prefixes in Numeric Column Test Data

### What happened
Used string IDs like `"junit000001"` for columns that are NUMERIC in PostgreSQL.

### Why it fails
PostgreSQL throws: `ERROR: invalid input syntax for type numeric`.

### Correct approach
Use purely numeric test IDs.

---

## Pitfall 36: Sorting Collections in Production Code to Fix Test Order

### What happened
Added `itemIdList.sort(Comparator.naturalOrder())` to make tests pass.

### Why it was reverted
Changes application behavior and may have performance implications. Root cause is tests assumed Oracle's implicit ordering.

### Correct approach
Add explicit `ORDER BY` to SQL if ordering matters for business logic. Use order-independent test assertions.

---

## Pitfall 37: Using ::timestamp(0) Casts on Parameter Placeholders

### What happened
Added `WHERE t1.SHIP_DATE >= ?::timestamp(0)` to SQL.

### Why it was problematic
When using PreparedStatement with `setTimestamp()`, JDBC handles type conversion. Explicit casts on `?` placeholders can cause type mismatch errors.

### Correct approach
Let JDBC handle the type: `WHERE t1.SHIP_DATE >= ?` with `stmt.setTimestamp(index, timestamp)`.

---

## Pitfall 38: Using DATE_TRUNC When orafce TRUNC is Available

### What happened
Used `DATE_TRUNC('month', CAST(date AS timestamp))` when orafce was installed.

### Why it was reverted
`DATE_TRUNC` returns TIMESTAMP, not DATE — causes type mismatch in comparisons. orafce's `TRUNC` behaves identically to Oracle's.

### Correct approach (with orafce)
`TRUNC(date_param::DATE, 'MM')`

---

## Pitfall 39: Inconsistent Alias Handling in UPDATE Statements

### What happened
Aliases removed from SET clause AND WHERE clause, breaking references.

### Correct approach
Remove alias ONLY from columns in the SET clause. Keep alias references in WHERE, subqueries, and JOINs.

---

## Pitfall 40: Changing IN Clause Types Without Checking Column Definition

### What happened
`IN ('2','3')` changed to `IN (2,3)` assuming INTEGER column, but some tables had VARCHAR.

### Correct approach
Always verify the actual column type in the database schema before changing IN clause types.

---

## Pitfall 41: Integer Arithmetic on TIMESTAMP Columns

### What happened
Oracle allows `date + integer` for day addition. PostgreSQL requires interval for TIMESTAMP.

### Correct approach
- If column is `TIMESTAMP`: use interval arithmetic (`+ INTERVAL '1 day'`)
- If column is `DATE`: integer addition works natively (`+ 1`)
- Be explicit about the type to avoid ambiguity

---

## Pitfall 42: Changing Parameter Types Without Verifying Column DDL

### What happened
Changed `query.setParameter("groupId", new BigDecimal(groupId))` without checking column type.

### Correct approach
Always verify the actual PostgreSQL column type before changing parameter binding. If column is `INTEGER`, use `Integer`; if `NUMERIC`/`DECIMAL`, use `BigDecimal`.

---

## Pitfall 43: Removing Null Checks When PostgreSQL Returns NULL

### What happened
```java
Number result = (Number) query.uniqueResult();
return result.intValue();  // NullPointerException when no rows match
```

### Correct approach
```java
Number result = (Number) query.uniqueResult();
return result == null ? 0 : result.intValue();
```

---

## Pitfall 44: Premature Conversion Without Stakeholder Confirmation

### What happened
Data type changes (date handling, numeric precision) were applied then reverted because business impact wasn't confirmed.

### Correct approach
Create issue/ticket for each ambiguous conversion. Get explicit confirmation from application team. Apply only after confirmation.

---

## Pitfall 44b: Narrowing a Sequence Value Back to int

### What happened
`rset.getInt(1)` was changed to `(int) rset.getLong(1)` to keep the existing method signature compiling.

### Why it fails
Sequences in these migrations are declared `MAXVALUE 9223372036854775807`. The cast wraps silently past 2147483647 — no exception, just duplicate or negative primary keys surfacing months after cutover.

### Correct approach
Keep the value as `long` and widen the consuming field. If the signature genuinely cannot change, use `Math.toIntExact(seqValue)` so the truncation throws instead of wrapping.

---

## General Anti-Pattern: Changing Production Code to Fix Test Failures

**Principle:** If a test fails after migration, first determine whether:
1. The test expectation is wrong (Oracle-specific assumption) → Fix the test
2. The production code has a genuine bug exposed by PostgreSQL → Fix the production code
3. The data setup is wrong for PostgreSQL → Fix the test data

Never change production code behavior solely to make a test pass without understanding the root cause.

---

## Summary Table

| # | Anti-Pattern | Correct Pattern |
|---|---|---|
| 1 | Oracle DATE → PostgreSQL DATE | Oracle DATE → TIMESTAMP(0) WITHOUT TIME ZONE |
| 2 | setString for INTEGER columns | setInteger / setInt with parseInt |
| 3 | Hard-cast to BigInteger | Use Number interface |
| 4 | BigDecimal.equals() in tests | Use compareTo() == 0 |
| 5 | DATE_TRUNC without CAST on param | CAST(param AS TIMESTAMP) |
| 6 | Boolean param on INTEGER column | CAST or change app to pass Int |
| 7 | -1 as boolean flag value | Use only 0 and 1 |
| 8 | Ignore CHAR padding | Change to VARCHAR or assert padded value |
| 9 | ROWNUM < N → LIMIT N | LIMIT N-1 |
| 10 | LIMIT outside subquery | LIMIT inside subquery |
| 11 | Missing subquery alias | Add alias after ) |
| 12 | ORDER BY after LIMIT | ORDER BY before LIMIT |
| 13 | Remove rownum > 0 entirely | Replace with WHERE 1=1 |
| 14 | UNNEST/ARRAY for IN clauses | Fix parameter binding type |
| 15 | Quote aliases for uppercase | Use lowercase in Java |
| 16 | NVL with mixed types (orafce) | Use COALESCE |
| 17 | Replace NVL when orafce works | Keep NVL if orafce handles it |
| 18 | Keep BigDecimal cast after nextval | Update to Number interface |
| 19 | executeQuery() for DML | executeUpdate() |
| 20 | BigDecimal for INTEGER DBUnit | Cast to Number.toInt() |
| 21 | Empty-string-to-null in getters | Return field as-is |
| 22 | Rely on implicit row ordering | Use ORDER BY or stream filter |
| 23 | Skip rollback before new queries | Explicit rollback() |
| 24 | currval without prior nextval | SELECT last_value FROM seq |
| 25 | to_date(to_char()) round-trip | Use col::date |
| 26 | Missing oracle in search_path | setCurrentSchema("oracle," + schema) |
| 27 | Ignore FK order in cleanup | Delete children before parents |
| 28 | TimeZone "JST" | Use "Asia/Tokyo" |
| 29 | Hardcode timezone in prod code | Fix test environment instead |
| 30 | Multiple patterns for same problem | Agree on ONE pattern |
| 31 | Change logic during migration | Syntax changes only |
| 32 | Convert all Oracle functions at once | Use orafce as bridge |
| 33 | max(id) + 1 instead of sequence | nextval('seq') |
| 34 | CAST without updating Java | Update SQL and Java together |
| 35 | String values for numeric columns | Numeric-only values |
| 36 | Sort in prod code for test order | ORDER BY in SQL or fix test |
| 37 | ::timestamp(0) on ? placeholder | Let JDBC handle type |
| 38 | DATE_TRUNC when orafce available | Use orafce TRUNC |
| 39 | Remove alias from WHERE in UPDATE | Only remove from SET clause |
| 40 | Change IN types without checking | Verify column type first |
| 41 | Integer arithmetic on TIMESTAMP | Use INTERVAL for TIMESTAMP |
| 42 | Change param type without DDL check | Verify column type first |
| 43 | Skip null check on query result | Always check for null |
| 44 | Convert without stakeholder OK | Get confirmation first |


---

## Pitfall 45: Using setParameterList() with String List on Integer Column

### What happened
`query.setParameterList("idList", stringList)` used with a `List<String>` on a column that is INTEGER in PostgreSQL. Oracle's implicit coercion handled this silently.

### Why it fails
PostgreSQL throws: `operator does not exist: integer = character varying`. This single failure cascades to 100+ subsequent errors via "current transaction is aborted, commands ignored until end of transaction block".

### Correct approach
Convert the list to the appropriate type before binding:
```java
List<Integer> intList = stringList.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
query.setParameterList("idList", intList);
```
Cross-reference the `@Entity` class field type to determine the correct parameter type.

---

## Pitfall 46: Direct BigDecimal Cast on nextval() Result in assignNewPk()

### What happened
`(BigDecimal) em.createSQLQuery("SELECT nextval(...)").uniqueResult()` throws `ClassCastException` because PostgreSQL returns `BigInteger`.

### Why it fails
PostgreSQL's `nextval()` returns `BigInteger`, not `BigDecimal` like Oracle. A direct cast fails at runtime.

### Correct approach
Handle multiple possible return types:
```java
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

---

## Pitfall 47: Using Two-Argument REPLACE() on CHARACTER Columns

### What happened
Oracle's `REPLACE(column, '-')` (two args) works to remove all dashes. PostgreSQL requires three arguments and also needs TEXT type input for `CHARACTER(n)` columns.

### Why it fails
PostgreSQL throws: `function replace(character, unknown) does not exist`.

### Correct approach
Use three arguments and cast CHARACTER to TEXT:
```sql
REPLACE(column::TEXT, '-', '')
```

---

## Pitfall 48: Using PostgreSQL Reserved Words as Column/Alias Names Without Quoting

### What happened
Columns or aliases named `limit`, `offset`, `user`, `order`, `end` used without quoting in SQL statements.

### Why it fails
PostgreSQL treats these as reserved keywords and throws syntax errors.

### Correct approach
Double-quote reserved word column names in all SQL references:
```sql
SELECT "limit", "offset" FROM config_table
```
In JPA annotations: `@Column(name = "\"limit\"")`

---

## Pitfall 49: Entity String Field Mapped to INTEGER Column Without Type Change

### What happened
Entity has `private String regionId` but the PostgreSQL column is `INTEGER`. Hibernate throws: `column "region_id" is of type integer but expression is of type character varying`.

### Why it fails
Oracle's implicit type coercion allowed String→NUMBER mapping. PostgreSQL requires exact type matching between entity field and column.

### Correct approach
Change the entity field type to match the database column:
```java
// BEFORE: private String regionId;
// AFTER:  private Integer regionId;
```
Update all getters, setters, and callers.

---

## Pitfall 50: Ignoring Transaction Abort Cascading Effects

### What happened
A single SQL error causes all subsequent statements in the same transaction to fail with "current transaction is aborted, commands ignored until end of transaction block". Tests report 100+ failures from a single root cause.

### Why it fails
PostgreSQL aborts the entire transaction after ANY error, unlike Oracle which allows continued execution.

### Correct approach
1. Fix the root cause error first (usually a type mismatch from pitfalls 45-49)
2. In test infrastructure, use independent transactions or rollback after errors
3. Debug by running failing tests individually to find the initial error

---

## Pitfall 51: Missing Tables in PostgreSQL DDL Referenced by Test Data

### What happened
Test data files (Excel/XML) reference tables that exist in Oracle schema but were not included in PostgreSQL migration DDL. DBUnit throws `NoSuchTableException`.

### Correct approach
1. Compile a complete list of tables referenced in ALL test data files
2. Cross-reference with PostgreSQL DDL scripts
3. Add missing `CREATE TABLE` statements to migration scripts
4. Ensure table structure matches what test data expects

---

## Pitfall 52: Oracle Empty String = NULL vs PostgreSQL NOT NULL Constraint

### What happened
Test data with empty string values (`''`) violates NOT NULL constraints in PostgreSQL. Oracle treated `''` as NULL, so those columns accepted it.

### Why it fails
PostgreSQL distinguishes empty string from NULL. Empty string passes NOT NULL checks but may fail CHECK constraints or application validation that expects non-empty.

### Correct approach
1. Change test data empty strings to explicit NULL where the intent was NULL
2. Or change to valid non-empty values
3. Review column constraints: if column should allow empty, ensure no CHECK constraint prevents it

---

## Pitfall 53: sysdate Arithmetic Without INTERVAL Keyword

### What happened
Converted `sysdate - 30` to `CURRENT_TIMESTAMP - 30` without INTERVAL.

### Why it fails
PostgreSQL does not support subtracting bare integers from TIMESTAMP. Throws: `operator does not exist: timestamp without time zone - integer`.

### Correct approach
Always use INTERVAL for arithmetic with TIMESTAMP:
```sql
CURRENT_TIMESTAMP - interval '30 days'
```

---

## Pitfall 54: Placing LIMIT Outside Correlated Subquery

### What happened
Converting `ROWNUM <= 1` in correlated subquery placed LIMIT on the outer query instead of inside the subquery.

### Why it fails
LIMIT on outer query restricts overall results, not the subquery condition. Different semantics.

### Correct approach
Place LIMIT inside the subquery:
```sql
WHERE EXISTS (SELECT 1 FROM tbl WHERE condition LIMIT 1)
```

---

## Pitfall 55: Using TRUNC(date, 'month') in HQL Queries

### What happened
Oracle's `TRUNC(date, 'month')` works in Hibernate Oracle dialect HQL. PostgreSQL dialect does not support this function in HQL.

### Why it fails
Hibernate's PostgreSQL dialect does not register TRUNC for date operations in HQL. Throws: `unexpected token: trunc`.

### Correct approach
Use HQL-compatible functions:
```java
"year(e.startDate) = year(e.endDate) and month(e.startDate) = month(e.endDate)"
```
Or convert to native SQL with `DATE_TRUNC`.

---

## Pitfall 56: Calling executeUpdate() on Custom DB Wrapper Class

### What happened
Application uses a custom database wrapper class that only exposes `execute()` and `executeQuery()` methods. After migration, `executeUpdate()` is needed but doesn't exist on the wrapper.

### Why it fails
Compilation error: `method executeUpdate() not found in CustomStatement`.

### Correct approach
Call `executeUpdate()` on the underlying `PreparedStatement`, with try-with-resources so a thrown exception cannot leak it (see Pitfall 60):
```java
try (PreparedStatement ps = connection.getSQLConnection().prepareStatement(sql)) {
    ps.executeUpdate();
}
```
Or update the wrapper class to expose `executeUpdate()`.

---

## Summary Table (Continued)

| # | Anti-Pattern | Correct Pattern |
|---|---|---|
| 45 | setParameterList with String on INT col | Convert list to matching Integer type |
| 46 | (BigDecimal) cast on nextval() result | Use instanceof checks, handle BigInteger |
| 47 | Two-arg REPLACE on CHARACTER column | Three-arg REPLACE with ::TEXT cast |
| 48 | Reserved words unquoted as columns | Double-quote reserved word columns |
| 49 | String entity field for INTEGER column | Change field type to Integer |
| 50 | Ignore transaction abort cascade | Fix root cause first, use rollback |
| 51 | Missing tables in PostgreSQL DDL | Add all test-referenced tables to DDL |
| 52 | Empty string in NOT NULL columns | Use explicit NULL or valid values |
| 53 | sysdate - 30 without INTERVAL | CURRENT_TIMESTAMP - interval '30 days' |
| 54 | LIMIT outside correlated subquery | LIMIT inside the subquery |
| 55 | TRUNC(date, 'month') in HQL | Use year()/month() HQL functions |
| 56 | executeUpdate() on wrapper class | Call on underlying PreparedStatement |

---

## Pitfall 57: Dropping Encryption in Transit When Rewriting the Connection URL

### What happened
`jdbc:oracle:thin:@host:1521:SID` was rewritten to `jdbc:postgresql://host:5432/dbname`. The application connected and tests passed, so the change looked complete.

### Why it fails
The PostgreSQL JDBC driver defaults to `ssl=false`. The Oracle connection had been using an encrypted transport (`TCPS`, or `SQLNET.ENCRYPTION_CLIENT` in `sqlnet.ora`), so the migration silently downgraded a compliant connection to plaintext. Nothing errors — this only surfaces in an audit.

### Correct approach
Always set `sslmode` explicitly on the migrated URL:
```
jdbc:postgresql://host:5432/dbname?sslmode=verify-full&sslrootcert=/path/to/rds-ca-bundle.pem
```
Use `verify-full` with the Amazon RDS CA bundle for RDS and Aurora; `require` is the floor elsewhere. Plaintext is acceptable only for a Testcontainers instance on loopback, and that pattern must not be copied into non-test configuration.

---

## Pitfall 58: Inlining a Bound Parameter While Restructuring SQL

### What happened
While converting `WHERE rownum <= :searchMax` to a `LIMIT` clause, the parameter was inlined because `LIMIT :searchMax` initially failed a type check:
```java
// Bound parameter replaced with concatenated value
sql.append("LIMIT " + searchMax);
```
The same shortcut appears when moving date arithmetic into Java (Rule 10) and when converting HQL to native SQL (Rule 33), where the query string is already being rebuilt.

### Why it fails
It converts a parameterised query into string-built SQL. If the value is request-derived at any point in its call chain, that is a SQL injection vector introduced by the migration itself — a security regression in code that was previously safe. It also loses statement-cache reuse.

### Correct approach
Keep the binding and cast it if the driver will not infer the type:
```java
sql.append("LIMIT CAST(:searchMax AS INTEGER) ");
query.setInteger("searchMax", searchMax);
```
Never concatenate a value into SQL text to resolve a type mismatch. Fix the binding type instead (see Pitfalls 2, 45, and 14 — the same wrong instinct produces all four).

---

## Summary Table (Continued)

| # | Anti-Pattern | Correct Pattern |
|---|---|---|
| 57 | Rewrite URL without `sslmode` | Set `sslmode=verify-full` (+ RDS CA bundle) |
| 58 | Inline a bound parameter to fix a type error | Keep the bind, `CAST(:param AS type)` |

---

## Pitfall 59: Bare `catch (Exception)` Around Rollback Handling

### What happened
The rollback added for PostgreSQL's transaction-abort semantics (Pitfalls 23 and 50) was wrapped in a bare catch that logged and continued:
```java
} catch (Exception re) { Logger.warn(this, re); }
```

### Why it fails
It swallows the one failure that matters. If the rollback itself fails, the connection is still in an aborted state and every following statement fails with "current transaction is aborted" — the exact condition the rollback was added to clear. The bare catch also hides `RuntimeException`s unrelated to SQL.

### Correct approach
```java
try {
    connection.getSQLConnection().rollback();
} catch (SQLException re) {
    Logger.warn(this, "Rollback failed; connection state is unreliable", re);
    throw new IllegalStateException("Rollback failed", re);
}
```
Catch `SQLException`. Do not log connection strings, credentials, or bound parameter values when adding diagnostics.

---

## Pitfall 60: Trailing `close()` Instead of try-with-resources

### What happened
Converting DML to `executeUpdate()` (Pitfalls 19 and 56) produced:
```java
PreparedStatement ps = connection.getSQLConnection().prepareStatement(sql);
ps.executeUpdate();
ps.close();
```

### Why it fails
Migration is precisely when `executeUpdate()` throws — on the type mismatches in Pitfalls 2, 45, and 49. When it does, `close()` never runs. With the HikariCP sizing used for Testcontainers (`maximumPoolSize=20`), leaked statements exhaust the pool and the run fails with connection timeouts that look unrelated to the original type error.

### Correct approach
```java
try (PreparedStatement ps = connection.getSQLConnection().prepareStatement(sql)) {
    ps.setString(1, param);
    ps.executeUpdate();
}
```

---

## Pitfall 61: Silently Disabling Tests With `@Ignore`

### What happened
Tests that failed against PostgreSQL were annotated `@Ignore("PostgreSQL migration: ...")` to get a green build. Nobody tracked how many.

### Why it fails
`@Ignore` is coverage deletion that reads as progress. At migration scale it reaches dozens of tests, and nothing distinguishes a test skipped for a missing lookup table from one covering authorization or constraint enforcement — so a security control regression merges behind a migration message.

### Correct approach
`@Ignore` is acceptable as a deliberate, tracked decision. Emit a list of every test ignored during the transformation (file, test name, reason) as part of the output. Flag any test related to authorization, authentication, input validation, or constraint and uniqueness enforcement for explicit sign-off rather than ignoring it.

---

## Pitfall 62: Carrying Production Data Into Migrated Test Fixtures

### What happened
While fixing fixture data for PostgreSQL (numeric-only IDs, empty string vs NULL, `-1` flag values), rows containing real customer names and addresses were copied into new and split fixture files.

### Why it fails
Oracle-era fixtures in long-lived enterprise applications are frequently production extracts. Editing them is not neutral: splitting a fixture for noPK tables or duplicating rows widens the reach of whatever personal data is already in there, and those files often end up in a repository with broader access than the database ever had.

### Correct approach
Substitute synthetic values for names, addresses, phone numbers, email addresses, national identifiers, and account numbers. Do not propagate real data into a new or split fixture. Report anything real found in an existing fixture rather than quietly carrying it forward.

---

## Summary Table (Continued)

| # | Anti-Pattern | Correct Pattern |
|---|---|---|
| 44b | `(int) rset.getLong(1)` on a sequence | Keep `long`, or `Math.toIntExact()` |
| 59 | Bare `catch (Exception)` around rollback | Catch `SQLException`, rethrow on failure |
| 60 | Trailing `ps.close()` | try-with-resources |
| 61 | Untracked `@Ignore` during migration | List every ignored test; sign off on security-relevant ones |
| 62 | Real data copied into migrated fixtures | Synthetic values; report real data found |
