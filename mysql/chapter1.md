# How to Use JSON With MySQL

### Java + MySQL + JSON + MyBatis + Jackson



## 0x1 Fusion of SQL & NoSQL

Datum in real world is categorized to **TWO** distincts: **structurized or SQL** and **non-structurized or NoSQL**. However it is hard to modeling data for all kinds of scenarios with one single principle. You can't find one relational model which is appreciate for all cases, and you also can't build up one non-structurized model fitting for remains. Humans are greedy, and are keep walking on the lonely way to seek one uniformed method to describe all things in the universe. Developers are also wondering one model that could capsule non-structures into structurized datums, and vice versa. That's say we need a third data form: __the fusion of SQL and NoSQL__ to modeling the real world. Nevertheless God is fair, just like relativity therory and quantum mechanics, we thought they are both correct but they are also incompatible with each other. You can't bind structures and non-structrues into one model smoothly, since they have different characters. But in the development there are huge requirements of modeling structures with non-structures together. For instance, we have a simple table storing the data of coupon:

```sql
create table `coupon` (
    id int not null auto_increment,
    title varchar(16) not null default '',
    value Decimal(19,3) not null default '0.0',
    status tinyint(4) not null default '0',
    primary key(`id`)
);

```
This table is simple and could be used to record coupon with id, title, status, and value. Nevertheless as long as biz evolving, restrictions would be added to coupon retrieving logic. For example, coupons only could be issued to the accounts that greater than specific levels, or the accounts that invested speicific products. Traditionally we will define tables linking to table coupon to store additional restrictions.

```sql
create table `coupon_product` {
     prod_code_id int not null default '0',
     prod_name varchar(16) not null default '',
     coupon_id int not null default '0'
     primary key(`prod_code_id`)
);
```
Table *coupon_product* defines the specific products linking to specici *coupon_id*. In biz code we need to filter out the coupons that have no specific products. With Java and mybatis the code looks like: 

```java
public List<Integer> couponIdsWithProduct(Integer prodCodeId, String user, String requestId) {
   logger.info("X-Request-Id: " + requestId + " queries coupon ids by user: " + user);
	CouponProductExample example = new CouponProductExample();
	example.or().andCouponIdEqualTo(prodCodeId);
	return couponProductMapper.selectByExample(example);
}

public Coupon getCouponById(Integer id, String user, String requestId) {
   logger.info("X-Request-Id: " + requestId + " queries coupon by user: " + user);
   return couponMapper.selectByPrimaryKey(couponId);
}

public List<Coupon> getCouponWithProduct(Integer prodCodeId, String user, String requestId) {
	logger.info("X-Request-Id: " + requestId + " queries coupons by user: " + user);
	List<Coupon> coupons = new LinkedList<>();
	couponidsWithProduct(prodCodeId, user, requestId).forEach(id->coupons.add(getCouponById(id))));
}
```
For performance concern, we can replace the method with **inner join** while fetching coupons. However more restrictions may be added as long as biz evolves rather than only one or two existing restrictions. In such cases, we need to change logic codes or MySQL preparment SQL statements in mybatis mapper files. This definitely makes more effort to maintain the project as long as new features involved in. Let's check the logic in previous code, the new retriction only play as a query filter, and no data modification or transaction drawbacks, aka, they only works as static template rules for filtering. I think **relational** and **structurized** don't have any benifits to it, except for making complexities growing.

## 0x2 MySQL JSON

JSON is a good data format to describe the world in **non-structrurized** pattern. MySQL starts to support native **JSON** data type since 5.7.8. With JSON introduced MySQL equipmented with huge NoSQL power. 

In MySQL JSON data is stored in binary format with **utf8mb4** character set and **utf8mb4_bin** collation. Although **UTF-8** is a subset of **utf8mb4**, in practice this may result in tricky issue with chinese characters. Since **utf8mb4** characters are converted to **ISO-8859-1** character while reading JSON data from MySQL, in Java codes the change will make chinese characters messy. To correct this issue, user needs to convert data from **ISO-8859-1** to **UTF-8** explicitly as below workaround:

```Java
String js = new String(js.getBytes("ISO-8859-1"), "UTF-8");
```

### JSON functions

With native JSON introduced, convenient native json functions are also provided to operate on JSON data. 

#### Search functions

##### JSON\_SEARCH(json\_doc, one\_or\_all, search\_str[, escape\_char[, path] ...]) 

Returns the path to the given string within JSON document. Returns **NULL** if any of *json\_doc*, *one\_or\_all*, *search\_str* or *path* arguments are **NULL**;no *path* exists within the document; or *search\_str* is not found.  

An error occurs if the *json\_doc* argument is not a valid JSON document, any path argument is not a valid path expression, *one\_or\_all* is not **'one'** or **'all'** , or *escape\_char* is not a constant expression.  

Within the **search\_str** search string argument, the **%** and _ characters work as for the **LIKE** operator: **%** matches any number of characters (including zero characters), and _ matches exactly one character.

To specify a literal **%** or _ character in the search string, precede it by the escape character. The default is **\\** if the **escape\_char** argument is missing or **NULL**. Otherwise, **escape\_char** must be a constant that is empty or one character.

```sql
mysql> SET @j = '["abc", [{"k": "10"}, "def"], {"x":"abc"}, {"y":"bcd"}]';

mysql> SELECT JSON_SEARCH(@j, 'one', 'abc');
+-------------------------------+
| JSON_SEARCH(@j, 'one', 'abc') |
+-------------------------------+
| "$[0]"                        |
+-------------------------------+

mysql> SELECT JSON_SEARCH(@j, 'all', 'abc');
+-------------------------------+
| JSON_SEARCH(@j, 'all', 'abc') |
+-------------------------------+
| ["$[0]", "$[2].x"]            |
+-------------------------------+

mysql> SELECT JSON_SEARCH(@j, 'all', 'ghi');
+-------------------------------+
| JSON_SEARCH(@j, 'all', 'ghi') |
+-------------------------------+
| NULL                          |
+-------------------------------+
```

##### JSON\_CONTAINS(json_doc, val[, path])

Returns **0** or **1** to indicate whether a specific value is contained in a target JSON document, or, if a path argument is given, at a specific path within the target document. 

Returns **NULL** if any argument is **NULL** or the path argument does not identify a section of the target document. 

An error occurs if either document argument is not a valid JSON document or the path argument is not a valid path expression or contains a **\*** or **\*\*** wildcard.

The following rules define containment: 

* A candidate scalar is contained in a target scalar **if and only if** they are comparable and are equal. Two scalar values are comparable if they have the same JSON_TYPE() types, with the exception that values of types INTEGER and DECIMAL are also comparable to each other.
* A candidate array is contained in a target array **if and only if** every element in the candidate is contained in some element of the target.
* A candidate nonarray is contained in a target array **if and only if** the candidate is contained in some element of the target.
* A candidate object is contained in a target object **if and only if** for each key in the candidate there is a key with the same name in the target and the value associated with the candidate key is contained in the value associated with the target key.

```sql
mysql> SET @j = '{"a": 1, "b": 2, "c": {"d": 4}}';
mysql> SET @j2 = '1';
mysql> SELECT JSON_CONTAINS(@j, @j2, '$.a');
+-------------------------------+
| JSON_CONTAINS(@j, @j2, '$.a') |
+-------------------------------+
|                             1 |
+-------------------------------+
mysql> SELECT JSON_CONTAINS(@j, @j2, '$.b');
+-------------------------------+
| JSON_CONTAINS(@j, @j2, '$.b') |
+-------------------------------+
|                             0 |
+-------------------------------+
```

##### JSON\_EXTRACT(json_doc, path[, path] ...)

Returns data from a JSON document, selected from the parts of the document matched by the path arguments. 

Returns **NULL** if any argument is NULL or no paths locate a value in the document. 

An error occurs if the **json\_doc** argument is not a valid JSON document or any path argument is not a valid path expression.

The return value consists of all values matched by the **path** arguments. If it is possible that those arguments could return multiple values, the matched values are autowrapped as an array, in the order corresponding to the paths that produced them. Otherwise, the return value is the single matched value.

```sql
mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]');
+--------------------------------------------+
| JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]') |
+--------------------------------------------+
| 20                                         |
+--------------------------------------------+
mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]', '$[0]');
+----------------------------------------------------+
| JSON_EXTRACT('[10, 20, [30, 40]]', '$[1]', '$[0]') |
+----------------------------------------------------+
| [20, 10]                                           |
+----------------------------------------------------+
mysql> SELECT JSON_EXTRACT('[10, 20, [30, 40]]', '$[2][*]');
+-----------------------------------------------+
| JSON_EXTRACT('[10, 20, [30, 40]]', '$[2][*]') |
+-----------------------------------------------+
| [30, 40]                                      |
+-----------------------------------------------+
```

##### column->path

In MySQL 5.7.9 and later, the **->** operator serves as an alias for the **JSON\_EXTRACT()** function when used with two arguments, a column identifier on the left and a JSON path on the right that is evaluated against the JSON document (the column value). You can use such expressions in place of column identifiers wherever they occur in SQL statements. This also works with JSON array values. 

JSON object example as below:

```sql

mysql> SELECT json_type(options) FROM coupon WHERE coupon_id = 72;
+--------------------+
| json_type(options) |
+--------------------+
| OBJECT             |
+--------------------+
1 row in set (0.01 sec)


mysql> SELECT options FROM coupon WHERE coupon_id = 72\G
*************************** 1. row ***************************
options: {"prodsEx": [], "prodsIn": [], "accounts": [], "prodTerms": [], "valueType": 1, "dailyCount": 100, "dailyAmount": 10000}
1 row in set (0.00 sec)

mysql> SELECT options->'$.dailyCount' FROM coupon WHERE coupon_id = 72;
+-------------------------+
| options->'$.dailyCount' |
+-------------------------+
| 100                     |
+-------------------------+
1 row in set (0.00 sec)

```

JSON array example as below:

```sql
mysql> SELECT json_type(activities) FROM coupon WHERE coupon_id = 72;
+-----------------------+
| json_type(activities) |
+-----------------------+
| ARRAY                 |
+-----------------------+
1 row in set (0.00 sec)

mysql> SELECT activities FROM coupon WHERE coupon_id = 72\G
*************************** 1. row ***************************
activities: [{"activityId": "2016122600000062", "activityTitle": "狼心狗行之辈滚滚当道"}, {"activityId": "2016122700000063", "activityTitle": "忠恕之道难也"}, {"activityId": "2016122700000065", "activityTitle": "呦呦鹿鸣"}, {"activityId": "2016122700000066", "activityTitle": "青青子衿悠悠我心"}]
1 row in set (0.00 sec)

mysql> SELECT activities->'$[0]' FROM coupon WHERE coupon_id = 72;
+---------------------------------------------------------------------------------------+
| activities->'$[0]'                                                                    |
+---------------------------------------------------------------------------------------+
| {"activityId": "2016122600000062", "activityTitle": "狼心狗行之辈滚滚当道"}           |
+---------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT activities->'$[0].activityTitle' FROM coupon WHERE coupon_id = 72;
+----------------------------------+
| activities->'$[0].activityTitle' |
+----------------------------------+
| "狼心狗行之辈滚滚当道"             |
+----------------------------------+
1 row in set (0.01 sec)

```

##### column->>path

This is an improved, unquoting extraction operator available in *MySQL 5.7.13* and later. Whereas the -> operator simply extracts a value, the ->> operator in addition unquotes the extracted result. In other words, given a JSON column value column and a path expression path, the following three expressions return the same value:

* JSON\_UNQUOTE( JSON\_EXTRACT(column, path) )

* JSON\_UNQUOTE(column -> path)

* column->>path

The ->> operator can be used wherever JSON\_UNQUOTE(JSON_\EXTRACT()) would be allowed. This includes (but is not limited to) **SELECT** lists, **WHERE** and **HAVING** clauses, and **ORDER BY** and **GROUP BY** clauses. 

```sql
mysql> SELECT activities->'$[0].activityId' FROM coupon WHERE coupon_id = 72;
+-------------------------------+
| activities->'$[0].activityId' |
+-------------------------------+
| "2016122600000062"            |
+-------------------------------+
1 row in set (0.01 sec)

mysql> SELECT activities->>'$[0].activityId' FROM coupon WHERE coupon_id = 72;
+--------------------------------+
| activities->>'$[0].activityId' |
+--------------------------------+
| 2016122600000062               |
+--------------------------------+
1 row in set (0.01 sec)
```


#### JSON modify functions

Since data modification is executed in service layer, the json modification functions are never used in practice. For details please refer MySQL dev document of [*13.16.4 Functions That Modify JSON Values*](http://dev.mysql.com/doc/refman/5.7/en/json-modification-functions.html)

## 0x3 MyBatis + JSON

### JDBC Mapping

Since MySQL suuports JSON so exciting, we definitely want to use it in service development. The question is how to do it? How to make current service code co-work with MySQL JSON type with little effort? Answer is from the changelog of **JDBC**. Here is the synopsis from [MySQL Connector/J 5.1.37(2015-10-15)](https://dev.mysql.com/doc/relnotes/connector-j/5.1/en/news-5-1-37.html):

**Functionality Added or Changed**

* Connector/J now supports the JSON data type, which has been supported by the MySQL server since release 5.7.8.

Although JSON type was supported since Connector/J 5.1.37 as described in the changelog of Connect/J, the document doest not update synchronizely, I am not sure what jdbc type will it be. After checking source code of Connector/J I found JSON is translated to **String** Java type, and jdbc type **CHAR**. Hmmm...it looks good! Here is the jdbcType definition in Connector/J source code.

```java(MysqlDefs.java)
class MysqlDefs {

static int mysqlToJavaType(int mysqlType) {
        int jdbcType;
        switch(mysqlType) {
            ...
            case MysqlDefs.FIELD_TYPE_JSON:
            case MysqlDefs.FIELD_TYPE_STRING:
                jdbcType = Types.CHAR;

                break;
            ...
        }
        return jdbcType;
}

```

### MyBatis Integration

With JSON introduced in MySQL and Connector/J supporting, we can refactor the new restrictions to coupon in previous by adding a new column **options** in JSON type.

```sql
sql> ALTER TABLE coupon ADD column options json AFTER status;

```
Now we have an additional options column in coupon table, and the options will have data as below format:

```json
{
	"prodCodes":[12345, 23456]
}

```

MyBatis plays an important role in service development. It bridges POJO and table with formatted mappers. With latest JDBC driver, JSON type would be mapped to **JDBC CHAR** type and **String** in POJO accordingly by MyBatisGenerator. Co-work with MySQL native JSON functions we can build up powerful query statments for JSON data. We can use MyBatis *example* model to build query statements for json data. For example, we add methods to *CouponExample.java* to query if some product code is exsiting in otpions or not, to check if prodCodes is empty or not, etc.

```java(CouponExample.java)
public Criteria andProdCodesContains(Integer value) {
     addCriterion("json_contains(options, '" + value + "', '$.prodCodes') = 1");
     return (Criteria)this;
}
        
public Criteria andProdCodesNotEmpty() {
     addCriterion("json_search(options->'$.prodCodes[*], 'all', '%') is not null);
     return (Criteria)this;
}

```

As long as biz evolved, we may need to add a flag to identify if the coupon only applies to iOS or Android clients. It could be simply implemented by adding a new field into **options** without modifing table strcuture. Now options looks as below:

```json
{
   "appOnly": true,
	"prodCodes":[12345, 23456]
}

```
Then we can add new methods to *CouponExample.java* to make querying easily.

```java(CouponExample.java)
public Criteria andAppOnlyIs(Boolean value) {
    addCriterion("options->'$.appOnly' = " + value);
    return (Criteria)this;
}

```

Now the restrictions validation could be complete in place with coupon formal parameters check together.

```java
public List<Coupon> getCouponWithProduct(Integer prodCodeId, String user, String requestId) {
	logger.info("X-Request-Id: " + requestId + " queries coupons by user: " + user);
	CouponExample example = new CouponExample();
	example.or().andProdCodesContains(prodCodeId);

	return getCouponsByExample(example, user, requestId);
}
```

## 0x4 Jackson + JSON

**options** is created as JSON type, consequently it will be schema-less. How to modeling it to Java POJO without processing each string explicitly? Since we use Jackson as JSON serializer/deserializer, we can process POJO to and from MySQL table by using Jackson. 

To get to the target, we need to define a Jackson JSON parser to convert JSON string to POJO, and vice versa.

```java
public class JSONUtils {
    private static final Logger logger = Logger.getLogger(JSONUtils.class);
    private static final ObjectMapper objectMapper;

    static {
        objectMapper = new ObjectMapper();
        objectMapper.setDateFormat(new SimpleDateFormat(Constants.DATETIME_FMT));
    }

    /**
     * Instantiates a new Json utils.
     */
    public JSONUtils() {
    }

    /**
     * Gets object mapper.
     *
     * @return the object mapper
     */
    public static ObjectMapper getObjectMapper() {
        return objectMapper;
    }

    /**
     * pojo,list,array convert to json string
     *
     * @param <T>      the type parameter
     * @param pojo     the pojo to convert json string
     * @param isPretty the is pretty
     * @return the json string of pojo
     * @throws Exception the exception
     */
    public static <T> String pojo2Json(T pojo, boolean isPretty) throws Exception {
        return !isPretty?
                objectMapper.writeValueAsString(pojo):
                objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(pojo);
    }

    /**
     * json string convert to pojo
     *
     * @param <T>     the type parameter
     * @param jsonStr the json str
     * @param clazz   the clazz
     * @return the t
     * @throws Exception the exception
     */
    public static <T> T json2Pojo(String jsonStr, Class<T> clazz) throws Exception {
        if (jsonStr == null || jsonStr.equals("")) {
            logger.warn("jsonStr is " + jsonStr);
            return null;
        }
        return objectMapper.readValue(jsonStr, clazz);
    }
}
```

Now we can define a POJO to represent options column in table of coupon as below:

```java
class CouponWrapper extend Coupon {
    private OptionsManager optionsMgr;
    ...
    
    public static class OptionsManager {
        private Boolean appOnly;
        private List<Integer> prodCodes;
    
        public void setAppOnly(Boolean appOnly) {
            this.appOnly = appOnly;
        }
    
        public Boolean getAppOnly() {
            return this.appOnly;
        }
    
        public void setProdCodes(List<Integer> prodCodes) {
            this.prodCodes = prodCodes;
        }
    
        public List<Integer> getProdCodes() {
            return this.prodCodes;
        }
    
        public static String convert2JsonString(Options options) {
             String js = "{}";
             try {
         	       js = JSONUtils.pojo2json(options, false);
             } catch (Exception e) {
                e.printStackTrace();
             }
             return js;
        }
    
        public static Options convertJson2Options(String js) {
            Options options = null;
            try {   // If there are chinese characters in js, needs to 
                    // convert the charset from utf8mb4->iso-8859-1->utf8
                    // explictly here. Since json in MySQL is read into
                    // iso-8859-1 charset defaultly.
                    js = new String(js.getBytes("ISO-8859-1"), "UTF-8");
                    options = JSONUtils.json2pojo(js, Options.class);
            } catch (Exception e) {
                e.printStackTrace();
            }
        
            return options;
        }
     }
}
```
Now we can use Options in coupon **CRUD** easily. For example we try to create a new coupon and get the options data as below:

```java

public int addNewCoupon(CouponWrapper couponWrapper, String user, String requestId) {

    logger.info("X-Request-Id: " + requestId + " add new coupon: " + couponWrapper + " by " + user);
    
    Coupon coupon = new Coupon();
    
    coupon.setOptions(CouponWrapper.OptionsManager.convert2JsonString(couponWrapper.getOptionsMgr()));
    
    int id = couponMapper.insertSelective(coupon);
    
    return id;
}

public Options getOptionsFromCoupon(Coupon coupon) {
    return CouponWrapper.OptionsManager.convertJson2Options(coupon.getOptions());
}
```


## 0x5 Perforamnce

### Generated Column

JSON data is stored as binary string MySQL, conseqently it could not be indexed. In product environment the data will be increasing as long as time lapses, and the performance will be slow down definitely. How to solve this issue? 

As of MySQL 5.7.6, **CREATE TABLE** supports the specification of generated columns. Values of a generated column are computed from an expression included in the column definition.

Generated columns are supported by the NDB storage engine beginning with MySQL Cluster NDB 7.5.3. For example, if we always want to calculate the third edege length of a triangle, we can add a generated column to peform calcualtion automatically.

```sql
CREATE TABLE triangle (
  sidea DOUBLE,
  sideb DOUBLE,
  sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb))
);
INSERT INTO triangle (sidea, sideb) VALUES(1,1),(3,4),(6,8);

sql>SELECT * FROM triangle;
+-------+-------+--------------------+
| sidea | sideb | sidec              |
+-------+-------+--------------------+
|     1 |     1 | 1.4142135623730951 |
|     3 |     4 |                  5 |
|     6 |     8 |                 10 |
+-------+-------+--------------------+
```
Generated column definitions have this syntax:

	col_name data_type [GENERATED ALWAYS] AS (expression)
  		[VIRTUAL | STORED] [UNIQUE [KEY]] [COMMENT comment]
		[[NOT] NULL] [[PRIMARY] KEY]

AS (expression) indicates that the column is generated and defines the expression used to compute column values. AS may be preceded by GENERATED ALWAYS to make the generated nature of the column more explicit. Constructs that are permitted or prohibited in the expression are discussed later.

The **VIRTUAL** or **STORED** keyword indicates how column values are stored, which has implications for column use:

* **VIRTUAL**: Column values **are not stored**, but are evaluated when rows are read, immediately after any BEFORE triggers. A virtual column takes no storage.

* **STORED**: Column values are evaluated and stored when rows are inserted or updated. A stored column does require storage space and can be indexed.

Notes:

* Prior to *MySQL 5.7.8*, virtual columns **cannot be indexed**. As of MySQL 5.7.8, InnoDB supports secondary indexes on virtual columns. See [Section 14.1.18.8, “Secondary Indexes and Generated Columns”](https://dev.mysql.com/doc/refman/5.7/en/create-table-secondary-indexes.html).

* The **default is VIRTUAL** if neither keyword is specified.

* It is permitted to mix VIRTUAL and STORED columns within a table.

* Generate columns also could work with **ALTER TABLE** operations.


### Indexing On Generated Virtual Columns


As of MySQL 5.7.8, InnoDB supports secondary indexes on generated virtual columns. Other index types are not supported.

A secondary index may be created on one or more virtual columns or on a combination of virtual columns and non-generated virtual columns. Secondary indexes on virtual columns may be defined as UNIQUE.

When a secondary index is created on a generated virtual column, generated column values are materialized in the records of the index. If the index is a covering index (one that includes all the columns retrieved by a query), generated column values are retrieved from materialized values in the index structure instead of computed “on the fly”.

There are **additional** write costs to consider when using a secondary index on a virtual column due to computation performed when materializing virtual column values in secondary index records during **INSERT** and **UPDATE** operations. Even with additional write costs, secondary indexes on virtual columns may be preferable to **STORED** generated columns, which are materialized in the clustered index, resulting in larger tables that require more disk space and memory. If a secondary index is not defined on a virtual column, there are additional costs for reads, as virtual column values must be computed each time the column's row is examined.


### Indexing JSON document

Based on generated virtual column, JSON data could be indexed by adding serials of virtual columns. For example, if we want to find the least value of product code in options, we can get it as below(Suppose the first product codes is the least one).

First, add a generated virtual column prod_code_min with indexing.

```sql
ALTER TABLE `coupon` ADD COLUMN `prod_code_min` int(11) GENERATED ALWAYS AS (json_extract(`options`,'$.prodCodes[0]')) VIRTUAL,
ALTER TABLE `coupon` ADD INDEX `idx_coupon_prod_code_min` (prod_code_min);
```

Second, add methods to CouponExample.java

```java
 public Criteria andProdCodeMinGreaterThanOrEqualTo(Integer value) {
        addCriterion("prod_code_min >= " + value);
        return (Criteria)this;
 }

 public Criteria andProdCodeMinLessThanOrEqualTo(Integer value) {
       addCriterion("prod_code_min <= " + value);
       return (Criteria)this;
 }
```
Then, we can use previous methods to filter coupons as below:

```java
public List<Coupon> getCoupons(Integer prodCode, String... args) {
     CouponExample example = new CouponExample();
     CouponExample.Criteria criteria = example.createCriteria();
     ...
     criteria.andProdCodeMinLessThanOrEqualTo(prodCode);
     ...
     return couponMapper.selectByExample(example);
}
```
Because the generated virutal column *prod\_code\_min* is indexed, the query will not perform full table scan, but from indexing quickly. We'd better add indexing to JSON document to improve query performance if it is used in our development.

## 0x6 Discussion

*TBD*



## 0x7 Appendix

[JSON document fast lookup with MySQL 5.7](https://www.percona.com/blog/2016/03/07/json-document-fast-lookup-with-mysql-5-7/)

[JSON data type](http://dev.mysql.com/doc/refman/5.7/en/json.html)

[Functions That Search JSON Values](http://dev.mysql.com/doc/refman/5.7/en/json-search-functions.html)



