

# PySpark Style Guide (Preview)

## Introduction

Idea of this code stype guide is improve consistency of our pipeline development in Foundry. Help teams write readable and maintainable programs using [Apache PySpark](https://spark.apache.org/docs/latest/api/python/).

This opinionated guide to PySpark code style presents common situations we've encountered and the associated best practices based on the most frequent recurring topics across PySpark repos.

Beyond PySpark specifics, the general practices of clean code are important in PySpark repositories- the Google [PyGuide](https://github.com/google/styleguide/blob/gh-pages/pyguide.md) is a strong starting point for learning more about these practices.

This pyspark code stype guide is based on [palantir/pyspark-style-guide](https://github.com/palantir/pyspark-style-guide).

---
## General

### Easy to read, consistent, explicit
#### Imports 
Import pyspark functions, types, and window narrowly and with consistent aliases.
```python

# bad
from pyspark import *

# bad
from pyspark.sql.functions import *

# good
from pyspark.sql import functions as F
from pyspark.sql import types as T
from pyspark.sql import window as W
from pyspark.sql import SparkSession, DataFrame

```
This prevents name collisions, as many PySpark functions have common names. This will make our code more consistent.

#### Use descriptive names for dataframes
```python

# bad
def get_large_downloads(df):
    return df.where(F.col("size") > 100)


# good
def get_large_downloads(downloads):
    return downloads.where(F.col("size") > 100)
```
Good naming is common practice for normal Python functions, Finding appropriate names for dataframes makes code easier to understand quickly.

#### Use column name strings to access columns

```python

# bad
df.col_name


# good
F.col("col_name")
```
Using the dot notation presents several problems. It can lead to name collisions if a column shares a name with a dataframe method. It requires the version of the dataframe with the column you want to access to be bound to a variable, but it may not be. It also doesn't work if the column name contains certain non-letter characters. The most obvious **exception** to this are joins where the joining column has a different name in the two tables.
```python
# OK
downloads.join(
  users, 
  downloads.user_id == user.id,
  how='inner',
)
```

#### When a function accepts a column or column name, use the column name option
```python
# bad
F.collect_set(F.col("client_ip"))


# good
F.collect_set("client_ip")
```
Expressions are easier to read without the extra noise of `F.col()`.

#### When the output of a function is stored as a column, give the column a concise name
```python
# bad
result = logs.groupby("user_id").agg(
    F.count("operation"),
)
result.printSchema()

# root
#  |-- user_id: string
#  |-- count(operation): long


# good
result = logs.groupby("user_id").agg(
    F.count("operation").alias("operation_count"),
)
result.printSchema()

# root
# |-- user_id: string
# |-- operation_count: long
```
The default column names are usually awkward, probably defy the naming style of other columns, and can get long.

#### Empty columns

If you need to add an empty column to satisfy a schema, always use `F.lit(None)` for populating that column. Never use an empty string or some other string signalling an empty value (such as `NA`).

Beyond being semantically correct, one practical reason for using `F.lit(None)` is preserving the ability to use utilities like `isNull`, instead of having to verify empty strings, nulls, and `'NA'`, etc.


```python
# bad
df = df.withColumn('foo', F.lit(''))

# bad
df = df.withColumn('foo', F.lit('NA'))


# good
df = df.withColumn('foo', F.lit(None))
```

#### Using comments

While comments can provide useful insight into code, it is often more valuable to refactor the code to improve its readability. The code should be readable by itself. If you are using comments to explain the logic step by step, you should refactor it.

```python
# bad

# Cast the timestamp columns
cols = ['start_date', 'delivery_date']
for c in cols:
    df = df.withColumn(c, F.from_unixtime(F.col(c) / 1000).cast(TimestampType()))
```

In the example above, we can see that those columns are getting cast to Timestamp. The comment doesn't add much value. Moreover, a more verbose comment might still be unhelpful if it only
provides information that already exists in the code. For example:

```python
# bad

# Go through each column, divide by 1000 because millis and cast to timestamp
cols = ['start_date', 'delivery_date']
for c in cols:
    df = df.withColumn(c, F.from_unixtime(F.col(c) / 1000).cast(TimestampType()))
```

Instead of leaving comments that only describe the logic you wrote, aim to leave comments that give context, that explain the "*why*" of decisions you made when writing the code. This is particularly important for PySpark, since the reader can understand your code, but often doesn't have context on the data that feeds into your PySpark transform. Small pieces of logic might have involved hours of digging through data to understand the correct behavior, in which case comments explaining the rationale are especially valuable.

```python
# good

# The consumer of this dataset expects a timestamp instead of a date, and we need
# to adjust the time by 1000 because the original datasource is storing these as millis
# even though the documentation says it's actually a date.
cols = ['start_date', 'delivery_date']
for c in cols:
    df = df.withColumn(c, F.from_unixtime(F.col(c) / 1000).cast(TimestampType()))
```

### Logical operations
#### Refactor complex logical operations

Logical operations, which often reside inside `.filter()` or `F.when()`, need to be readable. We apply the same rule as with chaining functions, keeping logic expressions inside the same code block to *three (3) expressions at most*. If they grow longer, it is often a sign that the code can be simplified or extracted out. Extracting out complex logical operations into variables makes the code easier to read and reason about, which also reduces bugs.

```python
# bad
F.when( (F.col('prod_status') == 'Delivered') | (((F.datediff('deliveryDate_actual', 'current_date') < 0) & ((F.col('currentRegistration') != '') | ((F.datediff('deliveryDate_actual', 'current_date') < 0) & ((F.col('originalOperator') != '') | (F.col('currentOperator') != '')))))), 'In Service')
```

The code above can be simplified in different ways. To start, focus on grouping the logic steps in a few named variables. PySpark requires that expressions are wrapped with parentheses. This, mixed with actual parenthesis to group logical operations, can hurt readability. For example the code above has a redundant `(F.datediff(df.deliveryDate_actual, df.current_date) < 0)` that the original author didn't notice because it's very hard to spot.

```python
# better
has_operator = ((F.col('originalOperator') != '') | (F.col('currentOperator') != ''))
delivery_date_passed = (F.datediff('deliveryDate_actual', 'current_date') < 0)
has_registration = (F.col('currentRegistration').rlike('.+'))
is_delivered = (F.col('prod_status') == 'Delivered')

F.when(is_delivered | (delivery_date_passed & (has_registration | has_operator)), 'In Service')
```

The above example drops the redundant expression and is easier to read. We can improve it further by reducing the number of operations.

```python
# good
has_operator = ((F.col('originalOperator') != '') | (F.col('currentOperator') != ''))
delivery_date_passed = (F.datediff('deliveryDate_actual', 'current_date') < 0)
has_registration = (F.col('currentRegistration').rlike('.+'))
is_delivered = (F.col('prod_status') == 'Delivered')
is_active = (has_registration | has_operator)

F.when(is_delivered | (delivery_date_passed & is_active), 'In Service')
```

Note how the `F.when` expression is now succinct and readable and the desired behavior is clear to anyone reviewing this code. The reader only needs to visit the individual expressions if they suspect there is an error. It also makes each chunk of logic easy to test if you have unit tests in your code, and want to abstract them as functions.

#### Factor out common logic
```python
# bad
csv_downloads_today = df.where(
    (F.col("operation") == "Download") & F.col("today") & (F.col("file_extension") == "csv")
)
exe_downloads_today = df.where(
    (F.col("operation") == "Download") & F.col("today") & (F.col("file_extension") == "exe")
)


# good
DOWNLOADED_TODAY = (F.col("operation") == "Download") & F.col("today")
csv_downloads_today = df.where(DOWNLOADED_TODAY & (F.col("file_extension") == "csv"))
exe_downloads_today = df.where(DOWNLOADED_TODAY & (F.col("file_extension") == "exe"))
```
It is okay to reuse these variables even though they include calls to `F.col`. This prevents repeated code and can make code easier to read.

### About `select()` and `withColumn()`
#### Use `select` statements to specify a schema contract

Doing a select at the beginning of a PySpark transform, or before returning, is considered good practice. This `select` statement specifies the contract with both the reader and the code about the expected dataframe schema for inputs and outputs. Any select should be seen as a cleaning operation that is preparing the dataframe for consumption by the next step in the transform.

Keep select statements as simple as possible. Due to common SQL idioms, allow only *one* function from `spark.sql.function` to be used per selected column, plus an optional `.alias()` to give it a meaningful name. Keep in mind that this should be used sparingly. If there are more than *three* such uses in the same select, refactor it into a separate function like `clean_<dataframe name>()` to encapsulate the operation.

Expressions involving more than one dataframe, or conditional operations like `.when()` are discouraged to be used in a select, unless required for performance reasons.


```python
# bad
aircraft = aircraft.select(
    'aircraft_id',
    'aircraft_msn',
    F.col('aircraft_registration').alias('registration'),
    'aircraft_type',
    F.avg('staleness').alias('avg_staleness'),
    F.col('number_of_economy_seats').cast('long'),
    F.avg('flight_hours').alias('avg_flight_hours'),
    'operator_code',
    F.col('number_of_business_seats').cast('long'),
)
```

Unless order matters to you, try to cluster together operations of the same type.

```python
# good
aircraft = aircraft.select(
    'aircraft_id',
    'aircraft_msn',
    'aircraft_type',
    'operator_code',
    F.col('aircraft_registration').alias('registration'),
    F.col('number_of_economy_seats').cast('long'),
    F.col('number_of_business_seats').cast('long'),
    F.avg('staleness').alias('avg_staleness'),
    F.avg('flight_hours').alias('avg_flight_hours'),
)
```

The `select()` statement redefines the schema of a dataframe, so it naturally supports the inclusion or exclusion of columns, old and new, as well as the redefinition of pre-existing ones. By centralising all such operations in a single statement, it becomes much easier to identify the final schema, which makes code more readable. It also makes code more concise.

#### Instead of using `withColumn()` to redefine type, cast in the select:
```python
# bad
df.select('comments').withColumn('comments', F.col('comments').cast('double'))


# good
df.select(F.col('comments').cast('double'))
```

But keep it simple:
```python
# bad
df.select(
    ((F.coalesce(F.unix_timestamp('closed_at'), F.unix_timestamp())
    - F.unix_timestamp('created_at')) / 86400).alias('days_open')
)


# good
df.withColumn(
    'days_open',
    (F.coalesce(F.unix_timestamp('closed_at'), F.unix_timestamp()) - F.unix_timestamp('created_at')) / 86400
)
```

Avoid including columns in the select statement if they are going to remain unused and choose instead an explicit set of columns - this is a preferred alternative to using `.drop()` since it guarantees that schema mutations won't cause unexpected columns to bloat your dataframe. However, dropping columns isn't inherintly discouraged in all cases; for instance- it is commonly appropriate to drop columns after joins since it is common for joins to introduce redundant columns. 

Finally, instead of adding new columns via the select statement, using `.withColumn()` is recommended instead for single columns. When adding or manipulating tens or hundreds of columns, use a single `.select()` for performance reasons.


### Joins
#### Be careful with joins
If you perform a left join, and the right side has multiple matches for a key, that row will be duplicated as many times as there are matches. This is called a "join explosion" and can dramatically bloat the output of your transforms job. Always double check your assumptions to see that the key you are joining on is unique, unless you are expecting the multiplication.

#### Explicitly specifying `how`
Bad joins are the source of many tricky-to-debug issues. There are some things that help like specifying the `how` (and `on`)explicitly, even if you are using the default value `(inner)`:


```python
# bad
flights = flights.join(aircraft, 'aircraft_id')

# also bad
flights = flights.join(aircraft, 'aircraft_id', 'inner')


# good
flights = flights.join(aircraft, on='aircraft_id', how='inner')
```

#### Use `left_outer` instead of `left`
`left_outer` is more explicit.

```python
# bad
flights = flights.join(aircraft, on='aircraft_id', how='left_outer')


# good
flights = flights.join(aircraft, on='aircraft_id', how='left_outer')
```

#### Avoid `right_outer` joins
 If you are about to use a `right_outer` join, switch the order of your dataframes and use a `left_outer` join instead. It is more intuitive since the dataframe you are doing the operation on is the one that you are centering your join around.

```python
# bad
flights = aircraft.join(flights, on='aircraft_id', how='right_outer')


# good
flights = flights.join(aircraft, on='aircraft_id', how='left_outer')
```

#### Avoid renaming all columns to avoid collisions
Avoid renaming all columns to avoid collisions. Instead, give an alias to the
whole dataframe, and use that alias to select which columns you want in the end.

```python
# bad
columns = ['start_time', 'end_time', 'idle_time', 'total_time']
for col in columns:
    flights = flights.withColumnRenamed(col, 'flights_' + col)
    parking = parking.withColumnRenamed(col, 'parking_' + col)

flights = flights.join(parking, on='flight_code', how='left_outer')

flights = flights.select(
    F.col('flights_start_time').alias('flight_start_time'),
    F.col('flights_end_time').alias('flight_end_time'),
    F.col('parking_total_time').alias('client_parking_total_time')
)


# good
flights = flights.alias('flights')
parking = parking.alias('parking')

flights = flights.join(parking, on='flight_code', how='left_outer')

flights = flights.select(
    F.col('flights.start_time').alias('flight_start_time'),
    F.col('flights.end_time').alias('flight_end_time'),
    F.col('parking.total_time').alias('client_parking_total_time')
)
```

In such cases, keep in mind:

1. It's probably best to drop overlapping columns *prior* to joining if you don't need both;
2. In case you do need both, it might be best to rename one of them prior to joining;
3. You should always resolve ambiguous columns before outputting a dataset. After the transform is finished running you can no longer distinguish them.

#### Don't use `.dropDuplicates()` or `.distinct()` as a crutch
 As a last word about joins, don't use `.dropDuplicates()` or `.distinct()` as a crutch.  
 If unexpected duplicate rows are observed, there's almost always an underlying reason for why those duplicate rows appear. Adding `.dropDuplicates()` only masks this problem and adds overhead to the runtime.

### Window Functions

Always specify an explicit frame when using window functions, using either [row frames](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/expressions/WindowSpec.html#rowsBetween-long-long-) or [range frames](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/expressions/WindowSpec.html#rangeBetween-long-long-). If you do not specify a frame, Spark will generate one, in a way that might not be easy to predict. In particular, the generated frame will change depending on whether the window is ordered (see [here](https://github.com/apache/spark/blob/v3.0.1/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/analysis/Analyzer.scala#L2899)). To see how this can be confusing, consider the following example:

```python
from pyspark.sql import functions as F, Window as W
df = spark.createDataFrame([('a', 1), ('a', 2), ('a', 3), ('a', 4)], ['key', 'num'])

# bad
w1 = W.partitionBy('key')
w2 = W.partitionBy('key').orderBy('num')
 
df.select('key', F.sum('num').over(w1).alias('sum')).collect()
# => [Row(key='a', sum=10), Row(key='a', sum=10), Row(key='a', sum=10), Row(key='a', sum=10)]

df.select('key', F.sum('num').over(w2).alias('sum')).collect()
# => [Row(key='a', sum=1), Row(key='a', sum=3), Row(key='a', sum=6), Row(key='a', sum=10)]

df.select('key', F.first('num').over(w2).alias('first')).collect()
# => [Row(key='a', first=1), Row(key='a', first=1), Row(key='a', first=1), Row(key='a', first=1)]

df.select('key', F.last('num').over(w2).alias('last')).collect()
# => [Row(key='a', last=1), Row(key='a', last=2), Row(key='a', last=3), Row(key='a', last=4)]
```

It is much safer to always specify an explicit frame:
```python
# good
w3 = W.partitionBy('key').orderBy('num').rowsBetween(W.unboundedPreceding, 0)
w4 = W.partitionBy('key').orderBy('num').rowsBetween(W.unboundedPreceding, W.unboundedFollowing)
 
df.select('key', F.sum('num').over(w3).alias('sum')).collect()
# => [Row(key='a', sum=1), Row(key='a', sum=3), Row(key='a', sum=6), Row(key='a', sum=10)]

df.select('key', F.sum('num').over(w4).alias('sum')).collect()
# => [Row(key='a', sum=10), Row(key='a', sum=10), Row(key='a', sum=10), Row(key='a', sum=10)]

df.select('key', F.first('num').over(w4).alias('first')).collect()
# => [Row(key='a', first=1), Row(key='a', first=1), Row(key='a', first=1), Row(key='a', first=1)]

df.select('key', F.last('num').over(w4).alias('last')).collect()
# => [Row(key='a', last=4), Row(key='a', last=4), Row(key='a', last=4), Row(key='a', last=4)]
```


#### Prefer use of window functions to equivalent re-joining operations
```python
# bad
result = downloads.join(
    downloads.groupby("user_id").agg(F.count("*").alias("download_count")), "user_id"
)


# good
window = W.Window.partitionBy(F.col("user_id"))
result = downloads.withColumn("download_count", F.count("*").over(window))

```
The window function version is usually easier to get right and is usually more concise.

#### Avoid empty `partitionBy()`

Spark window functions can be applied over all rows, using a global frame. This is accomplished by specifying zero columns in the partition by expression (i.e. `W.partitionBy()`).

Code like this should be avoided, as it forces Spark to combine all data into a single partition, which can be extremely harmful for performance.

Prefer to use aggregations whenever possible:

```python
# bad
w = W.partitionBy()
df = df.select(F.sum('num').over(w).alias('sum'))

# good
df = df.agg(F.sum('num').alias('sum'))
```

### UDFs (user defined functions)

It is highly recommended to avoid UDFs in all situations, as they are dramatically less performant than native PySpark. In most situations, logic that seems to necessitate a UDF can be refactored to use only native PySpark functions.

### Chaining of expressions

Keep in mind chaining expressions is a contentious topic. Feedback from team members are important !

#### Avoid chaining of expressions into multi-line expressions with different types
particularly if they have different behaviours or contexts. For example- mixing column creation or joining with selecting and filtering.

```python
# bad
df = (
    df
    .select('a', 'b', 'c', 'key')
    .filter(F.col('a') == 'truthiness')
    .withColumn('boverc', F.col('b') / F.col('c'))
    .join(df2, 'key', how='inner')
    .join(df3, 'key', how='left_outer')
    .drop('c')
)


# better (seperating into steps)
# first: we select and trim down the data that we need
# second: we create the columns that we need to have
# third: joining with other dataframes

df = (
    df
    .select('a', 'b', 'c', 'key')
    .filter(F.col('a') == 'truthiness')
)

df = df.withColumn('boverc', F.col('b') / F.col('c'))

df = (
    df
    .join(df2, 'key', how='inner')
    .join(df3, 'key', how='left_outer')
    .drop('c')
)
```

Having each group of expressions isolated into its own logical code block improves legibility and makes it easier to find relevant logic.
For example, a reader of the code below will probably jump to where they see dataframes being assigned `df = df...`.

```python
# bad
df = (
    df
    .select('foo', 'bar', 'foobar', 'abc')
    .filter(F.col('abc') == 123)
    .join(another_table, 'some_field')
)


# better
df = (
    df
    .select('foo', 'bar', 'foobar', 'abc')
    .filter(F.col('abc') == 123)
)

df = df.join(another_table, on='some_field', how='inner')
```

#### Multi-line expressions

To keep things consistent, please wrap the entire expression into a single parenthesis block, and avoid using `\`:

```python
# bad
df = df.filter(F.col('event') == 'executing')\
    .filter(F.col('has_tests') == True)\
    .drop('has_tests')


# good
df = (
  df
  .filter(F.col('event') == 'executing')
  .filter(F.col('has_tests') == True)
  .drop('has_tests')
)
```


#### When chaining several functions, open a cleanly indentable block using parentheses
```python
# bad
result = df.groupby("user_id", "operation").agg(
        F.min("creation_time").alias("start_time"),
        F.max("creation_time").alias("end_time"),
        F.collect_set("client_ip").alias("ips"),
)


# good
result = (
    df
    .groupby("user_id", "operation")
    .agg(
        F.min("creation_time").alias("start_time"),
        F.max("creation_time").alias("end_time"),
        F.collect_set("client_ip").alias("ips")
    )
)
```


#### Try to break the query into reasonably sized named chunks
```python

# bad (logical chunk not broken out)
downloading_user_operations = (
    logs.join(
        (logs.where(F.col("operation") == "Download").select("user_id").distinct()), "user_id"
    )
    .groupby("user_id")
    .agg(
        F.collect_set("operation").alias("operations_used"),
        F.count("operation").alias("operation_count"),
    )
)

# bad (chunks too small)
download_logs = logs.where(F.col("operation") == "Download")

downloading_users = download_logs.select("user_id").distinct()

downloading_user_logs = logs.join(downloading_users, "user_id")

downloading_user_operations = downloading_user_logs.groupby("user_id").agg(
    F.collect_set("operation").alias("operations_used"),
    F.count("operation").alias("operation_count"),
)


# good
downloading_users = logs.where(F.col("operation") == "Download").select("user_id").distinct()

downloading_user_operations = (
    logs.join(downloading_users, "user_id")
    .groupby("user_id")
    .agg(
        F.collect_set("operation").alias("operations_used"),
        F.count("operation").alias("operation_count"),
    )
)

```
Choosing when and what to name variables is always a challenge. Resisting the urge to create long PySpark function chains makes the code more readable.

---
## Foundry specific

Please see **[Foundry tech kit](https://shiftup.sharepoint.com/:o:/s/ictdata/Eook6G1NJD9KgsArYgDVSsQBddjDgaTU43SZ08Qyhi4IiA)** for other development guides.

### Don't repeat yourself
Don't repeat yourself with TO_RENAME and COLS_TO_KEEP and **rename only used columns**.
```python
from transforms.api import transform_df, Input, Output
from dataframe.transformer import rename_columns

# bad
TO_RENAME = {
    "no_das": "did_id",
    "no_dossier": "depil_id",
    "date_creation": "creation_date",
    "date_modification": "modification_date",
    "date_suppression": "deletion_date",
    "user_maj": "update_user"
}
COLS_TO_KEEP = [
    "did_id",
    "depil_id",
    "creation_date",
]

@transform_df(
    Output("Users/data/output"),
    df=Input("Users/data/input"),
)
def my_compute_function(df):

    df = rename_columns(df, TO_RENAME)
    df = df.select(*COLS_TO_KEEP)
    return df


# good
COLS_TO_KEEP = {
    "no_das": "did_id",
    "no_dossier": "depil_id",
    "date_creation": "creation_date",
}

@transform_df(
    Output("Users/data/output"),
    df=Input("Users/data/input"),
)
def my_compute_function(df):

    df = rename_columns(df, COLS_TO_KEEP)
    df = df.select(*list(COLS_TO_KEEP.values()))
    return df
```

### Use Check and expectations to explicit explain expected Output and Input

please see details in [Foundry documentation](https://bole.palantirfoundry.fr/workspace/documentation/product/transforms/data-expectations-quick-start).

```python
from transforms.api import transform_df, Input, Output, Check
from transforms import expectations as E
from dataframe.transformer import get_first_over_window

# bad
@transform_df(
    Output("Users/data/output"),
    df=Input("Users/data/input")
)
def my_compute_function(df):
    return (get_first_over_window(df, ["id"], "date_maj", ascOrder=False)


# good
@transform_df(
    Output(
        "Users/data/output",
        checks=Check(E.primary_key('id'), 'Primary Key', on_error='FAIL'),
    ),
    df=Input("Users/data/input")
)
def my_compute_function(df):
    return (get_first_over_window(df, ["id"], "date_maj", ascOrder=False)

```

### Add `require_incremental=True` if you are expecting an incremental build

To avoid unexpected snapshot buill create un-wanted data, please add `require_incremental=True` if you are expecting an incremental build.

```python

# bad

# BUMPING THE SEMANTIC_VERSION WILL CAUSE ALL HISTORY TO BE LOST
@incremental(
    semantic_version=1,
)
@transform(
    out=Output("Users/data/output"),
    raw=Input("Users/data/input"),
)
def my_compute_function(ctx, out, raw):
    ...


# good

# BUMPING THE SEMANTIC_VERSION WILL CAUSE ALL HISTORY TO BE LOST
@incremental(
    semantic_version=1,
    require_incremental=True  # if it is not incremental it will fails
)
@transform(
    out=Output("Users/data/output"),
    raw=Input("Users/data/input"),
)
def my_compute_function(ctx, out, raw):
    ...
```


### Optimization through (Re)partitioning
please see details in [Foundry documentation](https://bole.palantirfoundry.fr/workspace/documentation/product/foundry-training-portal/de_spark-optimization_module4)

Partition is the elementary entity on which Spark can execute computation. **Spark cannot parallelize computation inside one partition**(you need at leaset 2 partitions). The process of tuning number of dataset partitions is called “partitioning” or “re-partitioning.” 
> As a general rule, you should aim for a ratio of approximately one partition for every **128MB** of your dataset.

```python
# bad
@transform_df(
    Output("Users/data/output"),
    df=Input("Users/data/input")
)
def my_compute_function(df):
    return df    # by default it is df.repartition(4), this might not be optimized



# good
@transform_df(
    Output("Users/data/output"),
    df=Input("Users/data/input")
)
def my_compute_function(df):
    # calculate number of partitions following the rule mentioned above, sometimes you might want call df.coalesce
    return df.repartition(calculated_number)   

```
---
## Other Considerations and Recommendations

1. Be wary of functions that grow too large. As a general rule, a file
    should not be over 250 lines, and a function should not be over 70 lines.
2. Try to keep your code in logical blocks. For example, if you have
    multiple lines referencing the same things, try to keep them
    together. Separating them reduces context and readability.
3. Test your code! If you *can* run the local tests, do so and make
    sure that your new code is covered by the tests. If you can't run
    the local tests, build the datasets on your branch and manually
    verify that the data looks as expected.
4. Avoid `.otherwise(value)` as a general fallback. If you are mapping
    a list of keys to a list of values and a number of unknown keys appear,
    using `otherwise` will mask all of these into one value.
5. Do not keep commented out code checked in the repository. This applies
    to single line of codes, functions, classes or modules. Rely on git
    and its capabilities of branching or looking at history instead.
6. When encountering a large single transformation composed of integrating multiple different source tables, split it into the natural sub-steps and extract the logic to functions. This allows for easier higher level readability and allows for code re-usability and consistency between transforms.
7. Try to be as explicit and descriptive as possible when naming functions
    or variables. Strive to capture what the function is actually doing
    as opposed to naming it based the objects used inside of it.
8. Avoid using literal strings or integers in filtering conditions, new
    values of columns etc. Instead, to capture their meaning, extract them into variables, constants,
    dicts or classes as suitable. This makes the
    code more readable and enforces consistency across the repository.
---
## Contributing
One of the main purposes of this document is to encourage consistency. Some choices made here are arbitrary, but we hope they will lead to more readable code. Other choices may prove wrong with more time and experience. Suggestions for changes to the guide or additions to it are welcome. Please feel free to create an issue or pull request or chat directly in teams to start a discussion.
