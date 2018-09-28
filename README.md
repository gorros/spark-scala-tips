# spark-scala-tips
A collection of Spark (Scala) tips or best practices based on my experience.

## Data skew
### Problem:
A data skew is a condition when few partitions contain much more data than on average other. This can happen when for example you process website visits and partitioning of data is done by the domain name.
Obviously, some sites can have multiple time more visitors than the others. As a result, you will have a few large partitions and many small ones.
And event thous Spark processes partitions in parallel you will get overall pure performance, since each stage will not be finished until large partitions are not processed.
To avoid this conditions, monitor your application in Spark Web UI. You will notice if there are tasks that are taking much longer than the others.
This is a sign of data skew. 
### Solution:
I would say that there is no one solution for all case. But being aware of data skew is the key.
If possible use another partitioning key which provides better distribution. This is not always the case since processing logic defines partitioning. 
Very often data skew may occur when you are joining  dataframes, and in this case, partitioning happens by joining the field.
 In this case, best approach is to broadcast smaller dataframe:
 ```
 val joinedDF = broadcast(df1).join(df2, "id")
 ```
 Also, if partitioning is not a result of processing logic like in case of join, you can repartition dataframes by a column which will provide more even distribution.
 For example, if we are talking about site visits then partitioning by domain name, date or even hour may result in data skew. But if you partition by 
 minute or second this will provide quite even distribution.
 
 
## Use schemas for JSONs
### Problem:
You probably know that Spark provides convenient methods to read and write JSON data. 
Let's focus on reading. If you have dataset consisting of JSONs and you want to load them as dataframe
than you can do this:
```
val df = spark.read.json("path/to/jsons")
```
It is pretty straight forward. And this is probably fine as long as you 100% percent sure that all JSONs have same structure so your data frame will have expected schema. 
But data is not always clean and consistent, so it may occur that when you process another batch of data there will be JSONs with different schema (some fields will miss for example). 
This will result in a wrong interpretation of schema by Spark and farther issues of processing.

### Solution
To avoid this type of issues, we should "help" Spark in schema detection, by providing exact schema:
```
val df = spark.read.schema(schema).json("path/to/jsons")
```
This way if there will be a field missing in JSON, Spark will fill his filed with null instead of omitting it at all. 
The schema is especially helpful if you have nested structures. 
Also, this schema can be used if you have stringified JSONs. You can use `from_json` method to extract JSON from a string.


## Working with Redshift
There is a very nice [library](https://github.com/databricks/spark-redshift) to load/write data from/to Redshift. The tip regarding loading data from Redshift
is quite short. **After you load data persist dataframe**:
```
val df = getDfFromRedshift(ss, config)
df.persist(StorageLevel.MEMORY_AND_DISK)
# do processing
df.unpersist()
```
As we know, dataframe does not contain data it is actually sequence of transformations which are performed only when action is triggered. 
This way, if Spark wails at some stage it can reconstruct the current state from an initial data source. In our case, an initial data source is Redshift.
And data is loaded to Spark via `UNLAOD` command. So if you do not persist (cache) dataframe, in case calculation fails and Spark resubmits it, it will trigger
new `UNLOAD` and therefore unnecessary load on Redshift (not to mention longer overall processing time).

Whereas, while writing data to redshift definitely use `CSV GZIP` as  `tempformat`. Here is a nice [benchmark](https://www.stitchdata.com/blog/redshift-database-benchmarks-copy-performance-of-csv-json-and-avro/) confirming that.

## Working with S3

While reading files from S3 bare in mind that depending on API (__s3a__ or __s3n__) number of partitions for files on S3 will be different ([source](https://hadoop.apache.org/docs/current/hadoop-aws/tools/hadoop-aws/index.html#Features)):

```
<property>
  <name>fs.s3a.block.size</name>
  <value>32M</value>
  <description>Block size to use when reading files using s3a: file system.
  </description>
</property>
``` 
and
```
<property>
  <name>fs.s3n.block.size</name>
  <value>67108864</value>
  <description>Block size to use when reading files using the native S3
  filesystem (s3n: URIs).</description>
</property>
```
Generally, I would suggest using s3a since it is more recent API. But knowing the number of partitions will help you better configure resource allocation. (Here)[http://site.clairvoyantsoft.com/understanding-resource-allocation-configurations-spark-application/] you can find quite nice example of calcuation of recources for Spark application. 

(to be continued)


