# Before/After Code Examples — Consolidated Oracle to PostgreSQL Migration

This document provides concrete before/after examples for all major transformation patterns.

---

## 1. DDL Schema Conversion

### Before (Oracle)
```sql
CREATE TABLE example_table (
    id CHAR(8) NOT NULL,
    sequence_num NUMERIC(4,0) NOT NULL,
    name VARCHAR2(200),
    amount NUMERIC(10,0),
    rate NUMERIC(5,2),
    flag NUMERIC(1,0) DEFAULT 0 NOT NULL,
    record_date DATE,
    created_at DATE,
    PRIMARY KEY (id, sequence_num)
);
```

### After (PostgreSQL)
```sql
CREATE TABLE example_table (
    id CHARACTER(8) NOT NULL,
    sequence_num INTEGER NOT NULL,
    name CHARACTER VARYING(200),
    amount BIGINT,
    rate NUMERIC(5,2),
    flag INTEGER DEFAULT 0 NOT NULL,
    record_date TIMESTAMP(0) WITHOUT TIME ZONE,
    created_at TIMESTAMP(0) WITHOUT TIME ZONE,
    PRIMARY KEY (id, sequence_num)
);
```

---

## 2. Sequence Access (SELECT FROM DUAL)

### Before (Oracle)
```sql
SELECT seq_order_id.NEXTVAL FROM DUAL
```

### After (PostgreSQL)
```sql
SELECT nextval('seq_order_id')
```

---

## 3. Sequence with TO_CHAR Formatting

### Before (Oracle)
```java
em.createSQLQuery("select TO_CHAR(seq_request_id.nextval, 'FM00000000') from dual")
    .uniqueResult().toString();
```

### After (PostgreSQL)
```java
em.createSQLQuery("select TO_CHAR(nextval('seq_request_id'), 'FM00000000')")
    .uniqueResult().toString();
```

---

## 4. Sequence Return Type Handling

### Before (Oracle — returns BigDecimal)
```java
BigDecimal seq = (BigDecimal) query.uniqueResult();
String idSuffix = String.format("%05d", seq.longValue());
```

### After (PostgreSQL — use Number interface)
```java
Number seq = (Number) query.uniqueResult();
String idSuffix = String.format("%05d", seq.longValue());
```

---

## 5. Sequence with ResultSet

### Before (Oracle)
```java
ResultSet rset = stmt.executeQuery();
if (rset.next()) {
    String id = rset.getString(1);
}
```

### After (PostgreSQL)
```java
ResultSet rset = stmt.executeQuery();
if (rset.next()) {
    String id = Long.toString(rset.getLong(1));
}
```

Keep the value as `long`. Do not narrow with `(int) rset.getLong(1)` — sequences here are declared `MAXVALUE 9223372036854775807` and the cast wraps silently. If a caller signature forces `int`, use `Math.toIntExact()` so it throws instead.

---

## 6. CREATE SEQUENCE

### Before (Oracle)
```sql
CREATE SEQUENCE seq_order_id MINVALUE 1 MAXVALUE 9223372036854775807 INCREMENT BY 1 START WITH 1 CACHE 100 NOCYCLE;
```

### After (PostgreSQL)
```sql
CREATE SEQUENCE seq_order_id MINVALUE 1 MAXVALUE 9223372036854775807 INCREMENT BY 1 START WITH 1 CACHE 100 NO CYCLE;
```

---

## 7. SYSDATE Conversion

### In native SQL (with orafce)
```java
// Before: "SET updated_at = sysdate, ..."
// After:  "SET updated_at = sysdate(), ..."
```

### In native SQL (without orafce)
```java
// Before: "AND NEXT_SHIPPING_DATE >= sysdate"
// After:  "AND NEXT_SHIPPING_DATE >= NOW()"
```

### In HQL context
```java
// Before: "and t1.endDate >= sysdate "
// After:  "and t1.endDate >= sysdate() "
```

### TRUNC(SYSDATE) in HQL
```java
// Before: "(d.validEndDate is null or d.validEndDate >= trunc(sysdate)) "
// After:  "(d.validEndDate is null or d.validEndDate >= CURRENT_DATE) "
```

---

## 8. TRUNC Date Function

### Before (Oracle)
```sql
WHERE start_date <= TRUNC(/*targetDate*/'20221201', 'MM')
```

### After — Option A (PostgreSQL native)
```sql
WHERE start_date <= DATE_TRUNC('month', CAST(/*targetDate*/'20221201' AS TIMESTAMP))
```

### After — Option B (with orafce)
```sql
WHERE start_date <= TRUNC(/*targetDate*/'20221201'::DATE, 'MM')
```

---

## 9. NVL → COALESCE

### Before (Oracle)
```java
sql.append("NVL(T2.QUANTITY, 0) as qty ");
```

### After (PostgreSQL)
```java
sql.append("COALESCE(T2.QUANTITY, 0) as qty ");
```

---

## 10. DECODE → CASE

### Before (Oracle)
```sql
DECODE(item_id, 'OLD001', 'NEW001', 'OLD002', 'NEW002', item_id) AS mapped_item
```

### After (PostgreSQL)
```sql
(CASE item_id WHEN 'OLD001' THEN 'NEW001' WHEN 'OLD002' THEN 'NEW002' ELSE item_id END) AS mapped_item
```

---

## 11. ROWNUM → LIMIT (Simple top-N)

### Before (Oracle)
```java
sql.append(" WHERE ORDER_ID = :orderId ");
sql.append("   AND ROWNUM = 1");
sql.append(" ORDER BY BROWSE_ID DESC");
```

### After (PostgreSQL)
```java
sql.append(" WHERE ORDER_ID = :orderId ");
sql.append(" ORDER BY BROWSE_ID DESC");
sql.append(" LIMIT 1");
```

---

## 12. ROWNUM → LIMIT (Subquery unwrapping)

### Before (Oracle)
```java
sb.append("SELECT * FROM (")
    .append("SELECT * FROM m_table ")
    .append("ORDER BY updated_at DESC ")
    .append(") where rownum=1");
```

### After (PostgreSQL)
```java
sb.append("SELECT * FROM m_table ")
    .append("ORDER BY updated_at DESC ")
    .append("LIMIT 1");
```

---

## 13. ROWNUM < N (Off-by-one adjustment)

### Before (Oracle)
```java
sql.append(" WHERE ROWNUM < :NUM ");
query.setInteger("NUM", num);
```

### After (PostgreSQL)
```java
sql.append(" LIMIT :NUM ");
query.setInteger("NUM", num - 1);
```

---

## 14. ROWNUM > 0 (Always-true anchor)

### Before (Oracle)
```java
hql.append("FROM Entity AS s WHERE rownum > 0 ");
```

### After (PostgreSQL)
```java
hql.append("FROM Entity AS s WHERE 1=1 ");
```

---

## 15. ROWNUM in DELETE (ctid pattern)

### Before (Oracle)
```java
sql.append("DELETE FROM M_TABLE WHERE IND = :ind AND ROWNUM <= :limit ");
```

### After (PostgreSQL)
```java
sql.append("DELETE FROM M_TABLE WHERE CTID IN ( ");
sql.append("  SELECT CTID FROM M_TABLE WHERE IND = :ind LIMIT :limit ");
sql.append(")");
```

---

## 16. ROWNUM → ROW_NUMBER() (Pagination)

### Before (Oracle)
```java
String sql = "SELECT ROWNUMBER, ITEM_ID FROM ("
    + "SELECT ROWNUM AS ROWNUMBER, ITEM_ID FROM ("
    + "  SELECT ... ORDER BY DELIVERY_DATE DESC"
    + ")) WHERE ROWNUMBER BETWEEN :start AND :end";
```

### After (PostgreSQL)
```java
String sql = "SELECT ROWNUMBER, ITEM_ID FROM ("
    + "SELECT row_number() OVER (ORDER BY DELIVERY_DATE DESC) AS ROWNUMBER, ITEM_ID FROM ("
    + "  SELECT ..."
    + ") inner_q"
    + ") numbered WHERE ROWNUMBER BETWEEN :start AND :end";
```

---

## 17. Oracle Outer Join (+) → ANSI LEFT JOIN (Simple)

### Before (Oracle)
```java
String sql = "FROM TBL_ORDER t1, TBL_INVOICE t2 "
    + "WHERE t1.customer_id = ? "
    + "AND t2.invoice_id(+) = t1.invoice_id ";
```

### After (PostgreSQL)
```java
String sql = "FROM TBL_ORDER t1 "
    + "LEFT OUTER JOIN TBL_INVOICE t2 ON t2.invoice_id = t1.invoice_id "
    + "WHERE t1.customer_id = ? ";
```

---

## 18. Oracle Outer Join (+) → ANSI LEFT JOIN (Multiple ON conditions)

### Before (Oracle)
```java
String sql = "FROM TBL_PAYMENT t1, TBL_RECEIPT t2, TBL_INVOICE t3 "
    + "WHERE t1.DELETE_FLG = 0 AND t2.INVOICE_ID(+) = t1.INVOICE_ID "
    + "AND t2.SEQ_NO(+) = t1.SEQ_NO AND t1.INVOICE_ID = t3.INVOICE_ID ";
```

### After (PostgreSQL)
```java
String sql = "FROM TBL_PAYMENT t1 "
    + "LEFT OUTER JOIN TBL_RECEIPT t2 ON t2.INVOICE_ID = t1.INVOICE_ID AND t2.SEQ_NO = t1.SEQ_NO "
    + "INNER JOIN TBL_INVOICE t3 ON t1.INVOICE_ID = t3.INVOICE_ID "
    + "WHERE t1.DELETE_FLG = 0 ";
```

---

## 19. Oracle Outer Join (+) with Subquery

### Before (Oracle)
```java
String sql = "SELECT t1.col1, t2.col2 "
    + "FROM main_table t1 "
    + ",(SELECT sub_id, SUM(amount) as total FROM detail_table WHERE customer_id = ? GROUP BY sub_id) t2 "
    + "WHERE t1.customer_id = ? "
    + "  AND t1.sub_id = t2.sub_id(+) ";
```

### After (PostgreSQL)
```java
String sql = "SELECT t1.col1, t2.col2 "
    + "FROM main_table t1 "
    + "LEFT OUTER JOIN (SELECT sub_id, SUM(amount) as total FROM detail_table WHERE customer_id = ? GROUP BY sub_id) t2 "
    + "  ON t1.sub_id = t2.sub_id "
    + "WHERE t1.customer_id = ? ";
```

---

## 20. DELETE Statement Syntax

### Before (Oracle)
```java
private String deleteSql =
    "DELETE TBL_PRODUCT t1 WHERE "
        + " t1.customer_id = ? AND t1.product_id = ? ";
```

### After (PostgreSQL)
```java
private String deleteSql =
    "DELETE FROM TBL_PRODUCT WHERE "
        + " customer_id = ? AND product_id = ? ";
```

---

## 21. UPDATE Alias in SET Clause

### Before (Oracle)
```java
sql = "update tbl_entity t1 set "
    + "t1.status=1, t1.updated_date=:now "
    + "where t1.status=0";
```

### After (PostgreSQL)
```java
sql = "update tbl_entity t1 set "
    + "status=1, updated_date=:now "
    + "where t1.status=0";
```

---

## 22. Parameter Type Binding (String → Integer)

### Before (Oracle — implicit coercion)
```java
query.setString("categoryId", categoryId);
```

### After (PostgreSQL — explicit type)
```java
query.setInteger("categoryId", Integer.parseInt(categoryId));
```

---

## 23. Implicit Type Coercion (Doma2 CAST pattern)

### Before (Oracle — implicit conversion works)
```sql
WHERE delete_flg = /* isDeleted */0
AND active_flg = /* isActive */1
```

### After (PostgreSQL — explicit CAST required)
```sql
WHERE delete_flg = CAST(/* isDeleted */0 AS INTEGER)
AND active_flg = CAST(/* isActive */1 AS INTEGER)
```

---

## 24. COUNT/Aggregate Result Handling

### Before (Oracle)
```java
BigDecimal cnt = (BigDecimal) query.uniqueResult();
return cnt.intValue();
```

### After (PostgreSQL)
```java
Number cnt = (Number) query.uniqueResult();
return (cnt == null) ? 0 : cnt.intValue();
```

---

## 25. executeQuery() → executeUpdate() for DML

### Before (Oracle — lenient driver)
```java
stmt.executeQuery();  // Used for INSERT/UPDATE/DELETE
```

### After (PostgreSQL — strict driver)
```java
stmt.executeUpdate();  // Required for INSERT/UPDATE/DELETE
```

---

## 26. HQL DELETE Syntax

### Before (Oracle — HQL without FROM)
```java
String hql = "delete OrderItemTrace where id.orderId = :orderId ";
```

### After (PostgreSQL — HQL requires FROM)
```java
String hql = "delete from OrderItemTrace where id.orderId = :orderId ";
```

---

## 27. Oracle Hints Removal

### Before (Oracle)
```java
sql.append("select /*+ index(tbl_cart IDX_CART_01) */ product_id, sum(count) ");
```

### After (PostgreSQL)
```java
sql.append("select product_id, sum(count) ");
```

---

## 28. TO_DATE → CAST AS DATE (Single-argument truncation)

### Before (Oracle)
```java
sb.append("TO_DATE(SUBSCRIPTION.REGISTRATION_DATE) = TO_DATE(INFO.CREATED_AT) ");
```

### After (PostgreSQL)
```java
sb.append("CAST(SUBSCRIPTION.REGISTRATION_DATE as DATE) = CAST(INFO.CREATED_AT as DATE) ");
```

---

## 29. to_date → to_timestamp for TIMESTAMP Columns

### Before (Oracle — to_date works with TIMESTAMP)
```java
String sql = "WHERE CREATED_AT = to_date('2012/02/01 12:00:00', 'YYYY/MM/DD HH24:MI:SS') ";
```

### After (PostgreSQL — to_timestamp required)
```java
String sql = "WHERE CREATED_AT = to_timestamp('2012/02/01 12:00:00', 'YYYY/MM/DD HH24:MI:SS') ";
```

---

## 30. Date Arithmetic with INTERVAL

### Before (Oracle)
```java
sql.append("AND created_date >= trunc(:currentDate - 365) ");
```

### After (PostgreSQL)
```java
sql.append("AND created_date >= CAST(:currentDate AS DATE) - INTERVAL '365 days' ");
```

---

## 31. Date Subtraction for Week Calculation

### Before (Oracle — date subtraction returns number)
```java
sb.append("TRUNC((date1 - date2) / 7) AS WEEKAGE ");
```

### After (PostgreSQL — use EXTRACT)
```java
sb.append("FLOOR(EXTRACT(DAYS FROM (date1 - date2)) / 7) AS WEEKAGE ");
```

---

## 32. ADD_MONTHS → INTERVAL

### Before (Oracle)
```java
sql.append("ELSE ADD_MONTHS(TRUNC(J.shipping_date, 'MONTH'), 1) ");
```

### After (PostgreSQL)
```java
sql.append("ELSE (DATE_TRUNC('month', J.shipping_date) + INTERVAL '1 month') ");
```

---

## 33. LISTAGG → STRING_AGG

### Before (Oracle)
```java
sql.append("LISTAGG(doc.ID, ', ') WITHIN GROUP (ORDER BY doc.FLAG, doc.ID) as DOC_IDS ");
```

### After (PostgreSQL)
```java
sql.append("STRING_AGG(doc.ID, ', ' ORDER BY doc.FLAG, doc.ID) as DOC_IDS ");
```

---

## 34. XMLAGG/DBMS_XMLGEN → STRING_AGG

### Before (Oracle)
```java
sql.append("RTRIM(dbms_xmlgen.convert(XMLAGG(XMLELEMENT(e, LABEL, ',').EXTRACT('//text()')");
sql.append("  ORDER BY LABEL).GetClobVal(), 1), ',') AS ANSWER");
```

### After (PostgreSQL)
```java
sql.append("STRING_AGG(LABEL, ',' ORDER BY LABEL) AS ANSWER");
```

---

## 35. DENSE_RANK FIRST → ARRAY_AGG

### Before (Oracle)
```java
sb.append("SELECT MAX(T1.ITEM_ID) KEEP(DENSE_RANK FIRST ORDER BY T3.LIMIT ASC NULLS LAST, T1.ITEM_ID ASC) ITEM_ID ");
```

### After (PostgreSQL)
```java
sb.append("SELECT (ARRAY_AGG(T1.ITEM_ID ORDER BY T3.LIMIT ASC NULLS LAST, T1.ITEM_ID ASC))[1] AS ITEM_ID ");
```

---

## 36. DBMS_LOB.INSTR → starts_with

### Before (Oracle)
```java
sql.append("where DBMS_LOB.INSTR(registration_id,:value,1,1) = 1 ");
```

### After (PostgreSQL)
```java
sql.append("where starts_with(registration_id,:value) ");
```

---

## 37. REPLACE Two-Argument → Three-Argument

### Before (Oracle)
```java
sb.append("AND TRIM(REPLACE(addr.tel, '-')) = :tel ");
```

### After (PostgreSQL)
```java
sb.append("AND TRIM(REPLACE(addr.tel, '-', '')) = :tel ");
```

---

## 38. Blob → byte[]

### Before (Oracle)
```java
import java.sql.Blob;
private Blob uploadImage;
public Blob getUploadImage() { return this.uploadImage; }
```

### After (PostgreSQL)
```java
private byte[] uploadImage;
public byte[] getUploadImage() { return this.uploadImage; }
```

---

## 39. Column Alias Case in Result Map Access

### Before (Oracle — returns UPPERCASE)
```java
str = resultMap.get("DISPLAY_NAME").toString();
```

### After (PostgreSQL — returns lowercase)
```java
str = resultMap.get("display_name").toString();
```

---

## 40. Hibernate addScalar() for Type Control

### Before (Oracle — implicit type inference)
```java
query.addScalar("amount");
```

### After (PostgreSQL — explicit type required)
```java
query.addScalar("amount", Hibernate.BIG_DECIMAL);
```

---

## 41. CURRVAL → SELECT last_value

### Before (Oracle/PostgreSQL — currval may fail)
```java
String sql = "select currval('seq_order_trace')";
```

### After (PostgreSQL — direct sequence table access)
```java
String sql = "select last_value from seq_order_trace";
```

---

## 42. TimeZone Abbreviation → IANA ID

### Before (Oracle-compatible)
```java
TimeZone.setDefault(TimeZone.getTimeZone("JST"));
```

### After (PostgreSQL-compatible)
```java
TimeZone.setDefault(TimeZone.getTimeZone("Asia/Tokyo"));
```

---

## 43. Transaction Rollback Before New Queries

### Before (Oracle — continued queries after error)
```java
DBConnection.getConnection().setRollbackOnly();
Entity entity = EntityDao.getEntity(id, timestamp, true);
```

### After (PostgreSQL — explicit rollback required)
```java
DBConnection.getConnection().setRollbackOnly();
try {
    DBConnection.getConnection().getSQLConnection().rollback();
} catch (SQLException re) {
    Logger.warn(this, "Rollback failed; connection state is unreliable", re);
    throw new IllegalStateException("Rollback failed", re);
}
Entity entity = EntityDao.getEntity(id, timestamp, true);
```

Catch `SQLException`, not `Exception`. A bare catch here swallows connection-level failures and leaves the transaction in exactly the aborted state this fix exists to clear.

---

## 44. HQL to Native SQL Conversion (with LIMIT)

### Before (Oracle — HQL with ROWNUM)
```java
sql.append("FROM MessageDelivery t ");
sql.append("WHERE t.batchProcessId = :batchProcessId ");
sql.append(" AND rownum <= :searchMax");
sql.append(" ORDER BY t.sendDate DESC ");
Query query = em.createQuery(sql.toString());
```

### After (PostgreSQL — Native SQL with LIMIT)
```java
sb.append("SELECT t.* FROM TBL_MESSAGE_DELIVERY t ");
sb.append("WHERE t.batch_process_id = :batchProcessId ");
sb.append("ORDER BY t.send_date DESC ");
sb.append("LIMIT :searchMax ");
SQLQuery query = em.createSQLQuery(sb.toString());
query.addEntity(MessageDelivery.class);
```

---

## 45. Test Result Order Independence

### Before (Oracle — deterministic order without ORDER BY)
```java
Entity item = resultList.get(0);
assertThat(item.getId(), is("expected-id"));
```

### After (PostgreSQL — use stream filtering)
```java
Entity item = resultList.stream()
    .filter(m -> "expected-id".equals(m.getId()))
    .findFirst().get();
assertThat(item.getId(), is("expected-id"));
```

---

## 46. Search Path Configuration for orafce

### Before (orafce functions not found)
```java
pgDs.setCurrentSchema(schema);
```

### After (include oracle schema)
```java
pgDs.setCurrentSchema("oracle," + schema);
```

---

## 47. Date Column Cast Simplification

### Before (Oracle-style verbose truncation)
```java
sql.append("to_date(to_char(t1.billing_date, 'YYYY/MM/DD'), 'YYYY/MM/DD') ");
```

### After (PostgreSQL — direct ::date cast)
```java
sql.append("t1.billing_date::date ");
```

---

## 48. concat_ws('', ...) → || Operator

### Before (concat_ws with empty separator)
```sql
concat_ws('', ' ', t1.col1, t1.col2, t1.col3)
```

### After (PostgreSQL — || operator)
```sql
' ' || t1.col1 || t1.col2 || t1.col3
```

---

## 49. Reserved Word as Table Alias

### Before (Oracle — 'offset' as alias)
```java
sql.append("SELECT offset.entity_id FROM tbl_entity offset ");
```

### After (PostgreSQL — renamed alias)
```java
sql.append("SELECT cancel_ent.entity_id FROM tbl_entity cancel_ent ");
```

---

## 50. Empty String vs NULL in Entity Getters

### Before (Oracle workaround)
```java
public String getErrorCode() {
    return (this.errorCode != null && this.errorCode.trim().isEmpty()) ? null : this.errorCode;
}
```

### After (PostgreSQL — return as-is)
```java
public String getErrorCode() {
    return this.errorCode;
}
```

---

## 51. BigDecimal Scale Comparison in Tests

### Before (Oracle — returns exact scale as stored)
```java
assertEquals(BigDecimal("0.1"), entity.taxRate);
```

### After (PostgreSQL — use compareTo)
```java
assertEquals(0, BigDecimal("0.1").compareTo(entity.taxRate));
```

---

## 52. DBUnit getValue() Type Casting

### Before (Oracle — all numeric columns return BigDecimal)
```java
assertEquals(BigDecimal("12"), table.getValue(0, "INT_COLUMN"));
```

### After (PostgreSQL — returns native Java types)
```java
assertEquals(12, (table.getValue(0, "INT_COLUMN") as Number).toInt());
```

---

## 53. Date Comparison in Tests

### Before (Oracle — direct object comparison)
```java
assertThat(result.getCreatedAt(), is(DateUtil.toDate("20140101")));
```

### After (PostgreSQL — compare milliseconds)
```java
assertThat(result.getCreatedAt().getTime(), is(DateUtil.toDate("20140101").getTime()));
```

---

## 54. Test Data Split for Tables Without Primary Keys

### Before (single test data file)
```java
private static TestData testData;
@BeforeClass
public static void prepare() {
    testData = new TestData("MyDaoTest.xls");
    testData.load();
}
```

### After (split for noPK tables)
```java
private static TestData testData;
private static TestData testData_noPK;
@BeforeClass
public static void prepare() {
    testData = new TestData("MyDaoTest.xls");
    testData.load();
    testData_noPK = new TestData("MyDaoTest_noPK.xls");
    testData_noPK.load();
}
@AfterClass
public static void cleanUp() {
    TestDB.removeData("tbl_no_pk_table", "condition_col IN ('val1','val2')");
    testData.cleanup();
}
```

---

## 55. Doma2 Timestamp Literal Syntax

### Before (Oracle)
```sql
WHERE start_date = /* startDate */timestamp'2020-01-02 00:00:00'
```

### After (PostgreSQL)
```sql
WHERE start_date = /* startDate */'2020-01-02 00:00:00'::timestamp
```

---

## 56. BigDecimal Scale in AssertJ (Kotlin)

### Before (fails due to scale difference)
```kotlin
assertThat(actual).isEqualTo(expected)  // 0 vs 0.00
```

### After (handles scale)
```kotlin
assertThat(actual).usingRecursiveComparison()
    .withComparatorForType({ a, b -> a.compareTo(b) }, BigDecimal::class.java)
    .isEqualTo(expected)
```

---

## 57. Gradle Build Configuration

### Before (Oracle)
```groovy
dependencies {
    runtimeOnly("com.oracle.database.jdbc:ojdbc8:21.6.0.0.1")
    testRuntimeOnly("com.oracle.ojdbc:orai18n:19.3.0.0")
    testImplementation("org.testcontainers:oracle-xe")
}
```

### After (PostgreSQL)
```groovy
dependencies {
    runtimeOnly("org.postgresql:postgresql:42.7.8")
    testImplementation("org.testcontainers:postgresql:1.21.4")
    testImplementation("org.flywaydb:flyway-core:11.10.3")
    testImplementation("org.flywaydb:flyway-database-postgresql:11.10.3")
}
```

---

## 58. REGEXP_LIKE → ~ Operator

### Before (Oracle)
```sql
WHERE REGEXP_LIKE(column, '^\d+$')
```

### After (PostgreSQL)
```sql
WHERE column ~ '^\d+$'
```

---

## 59. DBMS_LOB Functions

### Before (Oracle)
```sql
DBMS_LOB.INSTR(column, search_string, 1, 1) = 1
dbms_lob.substr(column, length, start)
```

### After (PostgreSQL)
```sql
position(search_string in column) = 1
substring(column from start for length)
```


---

## 60. setParameterList() with Integer Columns

### Before (Oracle — implicit coercion allows String list on INTEGER column)
```java
List<String> idList = Arrays.asList("101", "102", "103");
query.setParameterList("idList", idList);
// SQL: WHERE entity_id IN (:idList)
```

### After (PostgreSQL — must match column type)
```java
List<Integer> intList = idList.stream()
    .map(Integer::parseInt)
    .collect(Collectors.toList());
query.setParameterList("idList", intList);
// SQL: WHERE entity_id IN (:idList)
```

---

## 61. BigInteger to BigDecimal Cast for Sequence in assignNewPk()

### Before (Oracle — returns BigDecimal)
```java
BigDecimal newpk = (BigDecimal) em.createSQLQuery(
    "SELECT nextval('seq_entity_id')").uniqueResult();
```

### After (PostgreSQL — returns BigInteger, handle both)
```java
Object result = em.createSQLQuery(
    "SELECT nextval('seq_entity_id')").uniqueResult();
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

## 62. REPLACE() Function — Three Arguments Required

### Before (Oracle — two-argument REPLACE removes occurrences)
```java
sb.append("AND TRIM(REPLACE(addr.telephone, '-')) = :tel ");
```

### After (PostgreSQL — three arguments required, cast CHARACTER to TEXT)
```java
sb.append("AND TRIM(REPLACE(addr.telephone::TEXT, '-', '')) = :tel ");
```

---

## 63. Reserved Words as Column Names — Double-Quoted

### Before (Oracle — reserved words usable without quoting)
```java
@Column(name = "limit")
private Integer limit;

sql.append("SELECT limit, offset FROM tbl_config ");
```

### After (PostgreSQL — reserved words must be double-quoted)
```java
@Column(name = "\"limit\"")
private Integer limit;

sql.append("SELECT \"limit\", \"offset\" FROM tbl_config ");
```

---

## 64. Entity Field Type vs Database Column Type Mismatch

### Before (Oracle — String field mapped to NUMBER column works)
```java
@Column(name = "region_id")
private String regionId;

public String getRegionId() { return this.regionId; }
public void setRegionId(String regionId) { this.regionId = regionId; }
```

### After (PostgreSQL — field type must match INTEGER column)
```java
@Column(name = "region_id")
private Integer regionId;

public Integer getRegionId() { return this.regionId; }
public void setRegionId(Integer regionId) { this.regionId = regionId; }
```

---

## 65. PostgreSQL Transaction Abort Cascading — Independent Transactions

### Before (Oracle — continues after error in same transaction)
```java
try {
    dao.insert(entity);  // Fails with constraint violation
} catch (Exception e) {
    log.warn(e);
}
Entity result = dao.findById(id);  // Works in Oracle
```

### After (PostgreSQL — must rollback before continuing)
```java
try {
    dao.insert(entity);  // Fails with constraint violation
} catch (SQLException e) {
    log.warn("Insert failed, rolling back", e);
    connection.getSQLConnection().rollback();
}
Entity result = dao.findById(id);  // Now works in PostgreSQL
```

---

## 66. DBUnit NoSuchTableException — Schema Completeness

### Before (missing tables in DDL causes NoSuchTableException)
```java
// Test data file references: tbl_audit_log, tbl_temp_data
// But DDL migration only has: tbl_audit_log
// Result: org.dbunit.dataset.NoSuchTableException: tbl_temp_data
```

### After (ensure all test-referenced tables exist in DDL)
```sql
-- Add missing table to migration DDL:
CREATE TABLE tbl_temp_data (
    id INTEGER NOT NULL,
    data CHARACTER VARYING(200),
    PRIMARY KEY (id)
);
```

---

## 67. NOT NULL Constraint — Oracle Empty String = NULL

### Before (Oracle — empty string treated as NULL, no constraint violation)
```sql
-- Test data: INSERT INTO tbl_config (id, name) VALUES (1, '');
-- Oracle treats '' as NULL, column allows NULL → passes
```

### After (PostgreSQL — empty string is NOT NULL, may violate constraints)
```sql
-- If column has NOT NULL with a check: change to explicit NULL or valid value
INSERT INTO tbl_config (id, name) VALUES (1, NULL);
-- Or update test data to use meaningful values
INSERT INTO tbl_config (id, name) VALUES (1, 'default');
```

---

## 68. sysdate Arithmetic → CURRENT_TIMESTAMP with INTERVAL

### Before (Oracle)
```java
sql.append("WHERE order_date > sysdate - 30 ");
sql.append("AND expire_date < sysdate + 7 ");
```

### After (PostgreSQL)
```java
sql.append("WHERE order_date > CURRENT_TIMESTAMP - interval '30 days' ");
sql.append("AND expire_date < CURRENT_TIMESTAMP + interval '7 days' ");
```

---

## 69. LIMIT Placement in Correlated Subqueries

### Before (Oracle — ROWNUM in correlated subquery)
```java
sql.append("SELECT t1.* FROM tbl_order t1 ");
sql.append("WHERE EXISTS (");
sql.append("  SELECT 1 FROM tbl_detail t2 ");
sql.append("  WHERE t2.order_id = t1.order_id AND ROWNUM <= 1");
sql.append(")");
```

### After (PostgreSQL — LIMIT inside subquery)
```java
sql.append("SELECT t1.* FROM tbl_order t1 ");
sql.append("WHERE EXISTS (");
sql.append("  SELECT 1 FROM tbl_detail t2 ");
sql.append("  WHERE t2.order_id = t1.order_id LIMIT 1");
sql.append(")");
```

---

## 70. TRUNC(date, 'month') in HQL — Use year() and month()

### Before (Oracle HQL — TRUNC works in Hibernate Oracle dialect)
```java
String hql = "FROM Entity e WHERE trunc(e.startDate, 'month') = trunc(e.endDate, 'month')";
```

### After (PostgreSQL HQL — use year/month extraction)
```java
String hql = "FROM Entity e WHERE year(e.startDate) = year(e.endDate) "
    + "and month(e.startDate) = month(e.endDate)";
```

---

## 71. executeUpdate() on PreparedStatement, Not Wrapper Objects

### Before (custom wrapper class lacks executeUpdate)
```java
// CustomStatement wrapper only has execute() and executeQuery()
CustomStatement stmt = connection.createStatement(sql);
stmt.executeUpdate();  // COMPILATION ERROR: method not found
```

### After (call executeUpdate on underlying PreparedStatement)
```java
try (PreparedStatement ps = connection.getSQLConnection().prepareStatement(sql)) {
    ps.setString(1, param);
    ps.executeUpdate();
}
```

Use try-with-resources, not a trailing `ps.close()`. Migration is when `executeUpdate()` throws most often (type mismatches), and a skipped `close()` leaks statements until the HikariCP pool is exhausted.

---

## 72. Connection Configuration — Preserve TLS and Keep Credentials Out of Config

### Before (Oracle, encrypted transport, credentials injected from a secret store)
```yaml
spring:
  datasource:
    driver-class-name: oracle.jdbc.OracleDriver
    url: jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=host)(PORT=2484))(CONNECT_DATA=(SID=ORCL)))
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

### After (PostgreSQL — `sslmode` explicit, credentials still externalised)
```yaml
spring:
  datasource:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://host:5432/dbname?sslmode=verify-full&sslrootcert=/opt/certs/rds-ca-bundle.pem
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

The `sslmode` parameter is not optional: the PostgreSQL driver defaults to unencrypted, so omitting it turns the `TCPS` connection above into plaintext with no error. Credentials continue to resolve from Secrets Manager, Parameter Store, or IAM database authentication — never inline them while editing this block.

---

## 73. Row Limiting — Keep the Parameter Bound

### Before (Oracle)
```java
sql.append(" WHERE t.batch_id = :batchId ");
sql.append("   AND rownum <= :searchMax ");
query.setInteger("searchMax", searchMax);
```

### Wrong (parameter inlined to work around a type error)
```java
sql.append(" LIMIT " + searchMax);   // string-built SQL — do not do this
```

### After (PostgreSQL — binding preserved, cast if the driver needs the type)
```java
sql.append(" WHERE t.batch_id = :batchId ");
sql.append(" ORDER BY t.send_date DESC ");
sql.append(" LIMIT CAST(:searchMax AS INTEGER) ");
query.setInteger("searchMax", searchMax);
```

The same rule applies to the `ctid` DELETE pattern (#15), dynamic `INTERVAL` arithmetic (#30), and HQL-to-native conversion (#44): the statement is being rebuilt, which is exactly when a bound parameter is most likely to be flattened into the string.
