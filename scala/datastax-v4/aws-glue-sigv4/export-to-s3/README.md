## Using Glue Export Example
This example provides scala script for exporting Amazon Keyspaces table data to S3 using AWS Glue. This allows you to export data from Amazon Keyspaces without setting up a spark cluster.

## Prerequisites
* Amazon Keyspaces table to export
* Amazon S3 bucket to store backups
* Amazon S3 bucket to store Glue job configuration and script

## Update the partitioner for your account
In Apache Cassandra, partitioners control which nodes data is stored on in the cluster. Partitioners create a numeric token using a hashed value of the partition key. Cassandra uses this token to distribute data across nodes.  To use Apache Spark or AWS glue you will need to update the partitioner. You can execute this CQL command from the Amazon Keyspaces console [CQL editor](https://console.aws.amazon.com/keyspaces/home#cql-editor) 

To update the partitioner to the Murmur3Partitioner, you can use the following query.

```UPDATE system.local set partitioner='org.apache.cassandra.dht.Murmur3Partitioner' where key='local';```


To see which partitioner is configured for the account, you can use the following query.

```SELECT partitioner from system.local;```

If the partitioner was changed, the query has the following output.

```
partitioner
--------------------------------------------
org.apache.cassandra.dht.Murmur3Partitioner
```


For more info see [Working with partitioners](https://docs.aws.amazon.com/keyspaces/latest/devguide/working-with-partitioners.html)

## Create IAM ROLE for AWS Glue
Create a new AWS service role named 'GlueKeyspacesExport' with AWS Glue as a trusted entity.

Included is a sample permissions-policy for executing Glue job. You can use managed policies AWSGlueServiceRole, AmazonKeyspacesReadOnlyAccess, read access to S3 bucket containing spack-cassandra-connector jar, configuration. Write access to S3 bucket containing backups.


## Cassandra driver configuration to connect to Amazon Keyspaces
The following configuration for connecting to Amazon Keyspaces with the spark-cassandra connector.

The G1.X DPU creates one executor per worker and the number of cores per executor is 4. The RateLimitingRequestThrottler in this example is set for 100 request per second. With this configuration and G.1X DPU can achieve a max of 100 request per Glue worker with 1000 rows per driver request. The Cassandra Java driver creates an event loop group used for I/O operations, which affects the throughput per executor. Increase the number of workers to scale throughput to a table.

```

datastax-java-driver {
  basic.request.consistency = "LOCAL_ONE"
  basic.request.default-idempotence = true
  basic.contact-points = [ "cassandra.us-east-2.amazonaws.com:9142"]
  advanced.reconnect-on-init = true

   basic.load-balancing-policy {
        local-datacenter = "us-east-2"
     }

   advanced.auth-provider = {
          class = software.aws.mcs.auth.SigV4AuthProvider
          aws-region = us-east-2
     }
    
   advanced.throttler = {
      class = RateLimitingRequestThrottler
      max-requests-per-second = 100
      max-queue-size = 50000
      drain-interval = 1 millisecond
    }

   advanced.ssl-engine-factory {
      class = DefaultSslEngineFactory
      hostname-validation = false
    }

    advanced.connection.pool.local.size = 3
    advanced.resolve-contact-points = false

}

```
Another important factor to consider before exporting the data is consistency. Amazon Keyspaces supports ONE, LOCAL_ONE, and LOCAL_QUORUM consistencies for reads. Depending on your use case, you can choose either an eventually consistent or strongly consistent dataset. It's worth noting that LOCAL_ONE requires half the read request units compared to LOCAL_QUORUM, which can impact your choice based on cost and performance considerations.

### Export to S3
The following example exports data to S3 using the spark-cassandra-connector. The script takes four parameters KEYSPACE_NAME, KEYSPACE_TABLE, S3_URI for backup files and FORMAT option (parquet, csv, json).  

The data export Glue job will facilitate the process of exporting data from Amazon Keyspaces to Amazon S3 in Scala using AWS Glue, and we will configure various Spark Cassandra connector parameters to optimize the data export process.
Here are some recommended spark settings to start with when exporting data from Amazon Keyspaces tables:

Start with a lower number of task.max Failures (default is 4). For example, you can start from 10 and increase as needed. This option helps increase the number of tasks retries in case of failure

Add the following option to the spark parameters: 
```
    spark.task.maxFailures=10
```
Set cassandra.query.retry.count to a bounded value (default is 60). This approach helps increase the number of query retries in case of failure, better to have more query retries rather than the task retry as this ensures a more robust and resilient data export process.

Add the following options to the spark parameters:  
```
   spark.cassandra.query.retry.count=100
```

```
import com.amazonaws.services.glue.GlueContext
import com.amazonaws.services.glue.util.GlueArgParser
import com.amazonaws.services.glue.util.Job
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
import org.apache.spark.sql.Dataset
import org.apache.spark.sql.Row
import org.apache.spark.sql.SaveMode
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.from_json
import org.apache.spark.sql.streaming.Trigger
import scala.collection.JavaConverters._
import com.datastax.spark.connector._
import org.apache.spark.sql.cassandra._
import org.apache.spark.sql.SaveMode._

object GlueApp {

  def main(sysArgs: Array[String]) {

    val args = GlueArgParser.getResolvedOptions(sysArgs, Seq("JOB_NAME", "KEYSPACE_NAME", "TABLE_NAME", "DRIVER_CONF", "FORMAT", "S3_URI").toArray)

    val driverConfFileName = args("DRIVER_CONF")

    val conf = new SparkConf()
        .setAll(
         Seq(
            ("spark.task.maxFailures",  "100"),
            ("spark.cassandra.connection.config.profile.path",  driverConfFileName),
            ("spark.cassandra.query.retry.count", "1000"),
            ("spark.cassandra.sql.inClauseToJoinConversionThreshold", "0"),
            ("spark.cassandra.sql.inClauseToFullScanConversionThreshold", "0"),
            ("spark.cassandra.concurrent.reads",  "512"),
            ("spark.cassandra.input.split.sizeInMB",  "64")

        ))

    val spark: SparkContext = new SparkContext(conf)
    val glueContext: GlueContext = new GlueContext(spark)
    val sparkSession: SparkSession = glueContext.getSparkSession

    import com.datastax.spark.connector._
    import org.apache.spark.sql.cassandra._
    import sparkSession.implicits._

    Job.init(args("JOB_NAME"), glueContext, args.asJava)

    val tableName = args("TABLE_NAME")
    val keyspaceName = args("KEYSPACE_NAME")
    val backupS3 = args("S3_URI")
    val backupFormat = args("FORMAT")

    val tableDf = sparkSession.read
      .format("org.apache.spark.sql.cassandra")
      .options(Map( "table" -> tableName, "keyspace" -> keyspaceName))
      .load()

    tableDf.write.format(backupFormat).mode(SaveMode.ErrorIfExists).save(backupS3)

    Job.commit()
  }
}

```
you can also explore a couple of Spark Cassandra configurations to further optimize the process:

1.	you can use a smaller Spark partition size (grouping multiple Cassandra rows) of 64 MB to avoid replaying large Spark tasks during a task failure. Default spark.cassandra.input.split.sizeInMB is 512 MB

```
"spark.cassandra.input.split.sizeInMB"  "64"
```

2.	You can decrease the fetch size in rows to improve performance and reduce memory consumption. Default fetch.sizeInRows is 1000 rows per driver request

```
"spark.cassandra.input.fetch.sizeInRows"	 "800" 
```

Add environment variables for your account ID from your AWS configuration, using aws-cli to create s3 bucket
```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```
## Create S3 bucket to store job artifacts
The AWS Glue ETL job will need to access jar dependencies, driver configuration, and scala script.
```
aws s3 mb s3://amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID
```

## Create S3 bucket to store job artifacts
The AWS Glue ETL job will use an s3 bucket to backup keyspaces table data.
```
aws s3 mb s3://amazon-keyspaces-backups-$AWS_ACCOUNT_ID
```



## Upload job artifacts to S3
The job will require
* The spark-cassandra-connector to allow reads from Amazon Keyspaces. Amazon Keyspaces recommends version 2.5.2 of the spark-cassandra-connector or above.
* application.conf containing the cassandra driver configuration for Keyspaces access
* export-sample.scala script containing the export code.

```
curl -L -O https://repo1.maven.org/maven2/com/datastax/spark/spark-cassandra-connector-assembly_2.12/3.1.0/spark-cassandra-connector-assembly_2.12-3.1.0.jar

curl -L -O https://github.com/aws/aws-sigv4-auth-cassandra-java-driver-plugin/releases/download/4.0.6-shaded-v2/aws-sigv4-auth-cassandra-java-driver-plugin-4.0.6-shaded.jar

aws s3api put-object --bucket amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID --key jars/spark-cassandra-connector-assembly_2.12-3.1.0.jar --body spark-cassandra-connector-assembly_2.12-3.1.0.jar

aws s3api put-object --bucket amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID --key jars/aws-sigv4-auth-cassandra-java-driver-plugin-4.0.6-shaded.jar --body aws-sigv4-auth-cassandra-java-driver-plugin-4.0.6-shaded.jar

aws s3api put-object --bucket amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID --key conf/cassandra-application.conf --body cassandra-application.conf

aws s3api put-object --bucket amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID --key scripts/export-sample.scala --body export-sample.scala

```
### Create AWS Glue ETL Job
You can use the following command to create a glue job using the script provided in this example. You can also take the parameters and enter them into the AWS Console.
```
aws glue create-job \
    --name "AmazonKeyspacesExport" \
    --role "GlueKeyspacesRestore" \
    --description "Export Amazon Keyspaces table to s3" \
    --glue-version "3.0" \
    --number-of-workers 2 \
    --worker-type "G.1X" \
    --command "Name=glueetl,ScriptLocation=s3://amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID/scripts/export-sample.scala" \
    --default-arguments '{
        "--job-language":"scala",
        "--FORMAT":"parquet",
        "--KEYSPACE_NAME":"my_keyspace",
        "--TABLE_NAME":"my_table",
        "--S3_URI":"s3://amazon-keyspaces-backups-$AWS_ACCOUNT_ID/snap-shots/",
        "--DRIVER_CONF":"cassandra-application.conf",
        "--extra-jars":"s3://amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID/jars/spark-cassandra-connector-assembly_2.12-3.1.0.jar,s3://amazon-keyspaces-artifacts/jars/aws-sigv4-auth-cassandra-java-driver-plugin-4.0.6-shaded.jar",
        "--extra-files":"s3://amazon-keyspaces-artifacts-$AWS_ACCOUNT_ID/conf/cassandra-application.conf",
        "--enable-continuous-cloudwatch-log":"true",
        "--user-jars-first":"true",
        "--class":"GlueApp"
    }'
```

you explore various Spark configuration parameters and AWS Glue worker settings to fine-tune the export performance, including altering split size, adjusting fetch row size, and modifying the number of Glue workers. By applying these adjustments and closely monitoring CloudWatch metrics, you can strike the right balance between throughput, latency, and resource allocation for your specific use case, ensuring a robust and efficient data migration process.