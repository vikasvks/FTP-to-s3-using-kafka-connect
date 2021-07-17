# Data from FTP to S3 using Kafka Connect in Docker
<br>
(Data from FTP to Kafka and kafka to S3 using Kafka Connect)

This uses Docker Compose to run the Kafka Connect worker and other kafka dependency.
### Prerequisites
*In docker-compose.yml,I already added the both FTP source and s3 sink connectors (change version if you needed)*
1. Install Docker (for kafka)
2. Running FTP sever and its details `Hostname`,`username`,`password`and `port`
3. Create the S3 bucket, make a note of the region
4. Obtain your access key pair
5. set `environment` variable for `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` 

```bash
#In linux
export AWS_ACCESS_KEY_ID=xxxxxxxxxxxxxxxxxxx
export AWS_SECRET_ACCESS_KEY=yyyyyyyyyyyyyyyyyyyyyyy
```
## Getting Started
1. Clone this repo and Bring the Docker Compose up

```bash
docker-compose up -d
```

2. Make sure everything is up and running

```bash
$ docker-compose ps
     Name                  Command               State                    Ports
---------------------------------------------------------------------------------------------
broker            /etc/confluent/docker/run   Up             0.0.0.0:9092->9092/tcp
kafka-connect     bash -c #                   Up (healthy)   0.0.0.0:8083->8083/tcp, 9092/tcp
                  echo "Installing ...
schema-registry   /etc/confluent/docker/run   Up             0.0.0.0:8081->8081/tcp
zookeeper         /etc/confluent/docker/run   Up             2181/tcp, 2888/tcp, 3888/tcp
ksqldb            /usr/bin/docker/run         Up             0.0.0.0:8088->8088/tcp

```

3. Create the FTP Source connector

```javascript
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/ftp-source/config \
    -d '
{
        "connector.class":"com.github.mmolimar.kafka.connect.fs.FsSourceConnector",
        "errors.log.enable":"true",
        "errors.log.include.messages":"true",
        "policy.class":"com.github.mmolimar.kafka.connect.fs.policy.SleepyPolicy",
        "policy.sleepy.sleep":"60000",
        "file_reader.class":"com.github.mmolimar.kafka.connect.fs.file.reader.CsvFileReader",
        "policy.fs.fs.ftp.data.connection.mode":"PASSIVE_LOCAL_DATA_CONNECTION_MODE",
        "policy.fs.fs.ftp.transfer.mode":"STREAM_TRANSFER_MODE","errors.tolerance":"all",
        "fs.uris":"ftp://<username>:<password>@{host}:{port}/file_path/",
        "topic":"ftp-source-topic"
}'
```
**customise  the connector for your environment** for more information about ftp source connector

4. Check that required connectors are loaded successfully
  (ftp-source is connector name)
  run this line:
   
         curl http://0.0.0.0:8083/connectors/ftp-source/status | jq .
5. Now Check FTP data in kafka topic

    Run ksqldb cli:
   
       docker exec -it ksqldb ksql http://ksqldb:8088

*  For show connector list `SHOW CONNECTORS;`
*  For show topics list `SHOW TOPICS;`
*  For show connector list `PRINT "ftp-source-topic" FROM BEGINNING;`
for more details of [ksqldb](https://docs.ksqldb.io/en/latest/developer-guide/ksqldb-reference/create-connector/) 
   
6. Now we make s3 sink connector(Kafka topic to s3)

```javascript
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/sink-s3/config \
    -d '
 {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "1",
    "topics": "ftp-source-topic",
    "s3.region": "us-east-2",
    "s3.bucket.name": "kafkatestdata",
    "flush.size": "3",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "schema.generator.class": "io.confluent.connect.storage.hive.schema.DefaultSchemaGenerator",
    "schema.compatibility": "NONE"
  }'
```


**Things to customise for your environment:**

* `topics` :  the source topic(s) you want to send to S3
* `key.converter` : match the serialisation of your source data. [see]https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)
* `value.converter` : match the serialisation of your source data. [see] https://www.confluent.io/blog/kafka-connect-deep-dive-converters-serialization-explained/)
* And many more


7. Bravo 🎉🎉,All done now check data into S3.

References

* https://hub.confluent.io [Confluent Hub]
* https://docs.confluent.io/current/connect/kafka-connect-s3/index.html#connect-s3 [S3 Sink connector docs]
* https://github.com/mmolimar/kafka-connect-fs [FTP source connector]
