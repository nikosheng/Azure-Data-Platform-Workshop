# Amazon RDS Migration To Delta Lake

In this workshop, we will leverage Amazon RDS MySQL Compatibility databases and try to perform batch and CDC migration. 

## Getting Started

### Batch Migration
In this lab, we are trying to use Azure Data Migration Service to firstly migrate the data in batches. With Azure DMS, we could start the DMS task to help us migrate the relational data by simple clicks.

- Firstly, we need to install `mysqldump` to export the table schema
	```
	sudo apt update sudo apt install mysql-client
	```
- Run`mysqldump` to dump database and table schema, no need to export data in this action
	```
	mysqldump -h 10.10.123.123 -u root -p --databases migtestdb --no-data > migtestdb.sql
	```
- Import the schema to target Azure Flexible DB for MySQL
	```
	mysql -h <azuredb> -u <user> -p migtestdb < demo.sql
	```
For more instructions, you may refer to [Tutorial: Migrate MySQL to Azure Database for MySQL offline using DMS - Azure Database Migration Service | Microsoft Docs](https://docs.microsoft.com/en-us/azure/dms/tutorial-mysql-azure-mysql-offline-portal) to learn how to create the DMS task in console. Here, we will skip this part.

### CDC Migration
In this lab, we will focus on how to leverage `Debezium MySQL CDC` connector with `Kafka Connect` to migrate the incremental records from source database to delta lake in Azure.

MySQL has a binary log (`binlog`) that records all operations in the order in which they are committed to the database. This includes changes to table schemas as well as changes to the data in tables. MySQL uses the binlog for replication and recovery.

The Debezium MySQL connector reads the binlog, produces change events for row-level  `INSERT`,  `UPDATE`, and  `DELETE`  operations, and emits the change events to Kafka topics. Client applications read those Kafka topics.

In the previous batch migration, we transfer the data from source to Azure Flex DB of MySQL, then we would create a replica instance to be the source of our CDC migration upstream.

- Firstly, we will record the binlog postion of the last batch migration, to ensure the starting position
- Then we will launch the Kafka connect distributed mode
	```
	bin/connect-distributed.sh -daemon config/connect-distributed.properties
	```
- After the connect cluster is running, we will setup the Debezium MySQL CDC connector via the REST interface
	```
	curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/source-azuredb-orders-00/config \
    -d '{
            "connector.class": "io.debezium.connector.mysql.MySqlConnector",
            "database.hostname": "azureflexdbreplica.mysql.database.azure.com",
            "database.port": "3306",
		    "snapshot.mode": "when_needed",
            "database.user": "sqluser",
            "database.password": "password",
            "database.server.id": "608266110",
            "database.server.name": "demo",
            "value.converter": "io.confluent.connect.avro.AvroConverter",
            "value.converter.schema.registry.url": "http://10.0.0.10:8081",
            "database.history.kafka.bootstrap.servers": "wn0-nikoaz.xxx.bx.internal.cloudapp.net:9092,wn1-nikoaz.xxx.bx.internal.cloudapp.net:9092",
            "database.history.kafka.topic": "dbhistory.demo" ,
            "decimal.handling.mode": "double",
            "include.schema.changes": "true",
            "transforms": "unwrap,addTopicPrefix",
			"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
			"transforms.unwrap.drop.tombstones": "false",
			"transforms.unwrap.delete.handling.mode": "rewrite",
            "transforms.addTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
            "transforms.addTopicPrefix.regex":"(.*)\\.(.*)\\.(.*)",
            "transforms.addTopicPrefix.replacement":"$3"
    }'
	```
- Then, we might use `kafka-avro-console-consumer` to parse the CDC binlog event record in AVRO format, here, I will use docker container including the avro consumer to consume the records
	```
	docker exec -it schema-registry bash -c 'kafka-avro-console-consumer --bootstrap-server wn0-hdinsi.sthfnkbp2cvunkqnnfyrizgesd.bx.internal.cloudapp.net:9092,wn1-hdinsi.sthfnkbp2cvunkqnnfyrizgesd.bx.internal.cloudapp.net:9092\
	--topic orders --from-beginning \
	--property schema.registry.url="http://schema-registry:8081"'
	```

- When the CDC binlog records are correctly consumed, we could start the ADLS Gen2 connector
	```
	curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/sink-debezium-orders-adlsgen2/config \
    -d '{
        "connector.class": "io.confluent.connect.azure.datalake.gen2.AzureDataLakeGen2SinkConnector", 
        "tasks.max": 1,
        "topics": "orders",
        "flush.size": 5,
	    "partition.duration.ms": 86400000,
		"locale": "zh_HK",
        "timezone": "UTC",
        "value.converter": "io.confluent.connect.avro.AvroConverter",
        "value.converter.schema.registry.url": "http://10.0.0.10:8081",
        "azure.datalake.gen2.account.name": "msftnikoadlsgen2storage",
        "azure.datalake.gen2.client.id": "91e977a0-b5fc-4f1b-882e-6df9e1f1001d",
        "azure.datalake.gen2.client.key": "xxx",
        "azure.datalake.gen2.token.endpoint": "https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47/oauth2/token",
        "topics.dir": "ecommerce-bronze",
        "storage.url": "https://msftnikoadlsgen2storage.dfs.core.windows.net/ecommerce-bronze/",
        "behavior.on.null.values": "ignore",
		"partitioner.class": "io.confluent.connect.storage.partitioner.DailyPartitioner",
        "format.class": "io.confluent.connect.azure.storage.format.parquet.ParquetFormat",
		"parquet.codec": "snappy",
	    "confluent.topic.bootstrap.servers": "wn0-nikoaz.xxx.bx.internal.cloudapp.net:9092,wn1-nikoaz.xxx.bx.internal.cloudapp.net:9092",
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
		"transforms.unwrap.drop.tombstones": "false",
		"transforms.unwrap.delete.handling.mode": "rewrite"
    }'
	```

- We could validate the connectos status via REST interface exposed by Kafka Connect
	```
	curl -s -XGET http://localhost:8083/connectors/<connector>/status
	```
- Insert several new records to `orders`
	```
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (309, 187536.91, 'Mitsubishi', 'RVR', 'London', 'Marquardt-Anderson', '74 Schiller Hill');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (829, 78643.44, 'Toyota', 'Sequoia', 'Bristol', 'Dach and Sons', '54897 Lerdahl Crossing');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (835, 112514.96, 'Lincoln', 'Aviator', 'York', 'Botsford and Sons', '859 Alpine Pass');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (303, 70319.18, 'Ford', 'F150', 'Sheffield', 'Frami-Moen', '4 Meadow Valley Place');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (852, 61491.03, 'GMC', '2500', 'Sheffield', 'Senger Inc', '52744 Merrick Street');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (814, 98162.3, 'Nissan', 'Xterra', 'Leeds', 'Boyer-Hickle', '72 Jay Way');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (187, 135214.67, 'Ford', 'Laser', 'London', 'Botsford-Wilderman', '3132 Kropf Park');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (360, 83225.69, 'GMC', 'Sierra Denali', 'Exeter', 'Halvorson, Brown and Leffler', '3167 Golf Course Plaza');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (655, 50707.65, 'Pontiac', 'Bonneville', 'Leeds', 'Ziemann-Orn', '03 6th Lane');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (919, 168193.58, 'Mercedes-Benz', 'CL-Class', 'York', 'McLaughlin, Swift and Swift', '19091 Brown Crossing');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (388, 98558.51, 'Pontiac', 'G5', 'Leeds', 'Dibbert and Sons', '20204 Kipling Terrace');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (800, 80068.02, 'Audi', 'S8', 'York', 'Price-Davis', '97 Shasta Park');
	```
- Move to ADLS console to validate the CDC records are commited to object storage
- Validate the incremental CDC records in Synapse workspace
	```
	SELECT *
	FROM OPENROWSET(
			BULK 'orders',
			DATA_SOURCE = 'DeltaLakeStorage',
			FORMAT = 'delta'
		) as rows
	WHERE customer_id IN (309, 829);
	```
- Let's update and delete records to validate whether the change events could be applied to delta lake in real-time
	```
	update orders set order_total_usd = order_total_usd + 1000 where customer_id = 829;

	delete from orders where customer_id = 309;
	```
- Validate the incremental CDC records in Synapse workspace again to check the latest status
	```
	SELECT *
	FROM OPENROWSET(
			BULK 'orders',
			DATA_SOURCE = 'DeltaLakeStorage',
			FORMAT = 'delta'
		) as rows
	WHERE customer_id IN (309, 829);
	```

### Schame Change
- If we try to add a column `gender` to table `orders`, the schema change will be captured by Debezium, and it will send to topic `demo`, which is the server name of the debezium config.
	```
	# Add a new column gender to table orders
	alter table demo.orders add column gender VARCHAR(10) NOT NULL DEFAULT 'NA';

	docker exec -it schema-registry bash -c 'kafka-avro-console-consumer --bootstrap-server wn0-nikoaz.55gugjr3douuxdwqs241cjkktf.bx.internal.cloudapp.net:9092,wn1-nikoaz.55gugjr3douuxdwqs241cjkktf.bx.internal.cloudapp.net:9092 \
	--topic demo \
	--property schema.registry.url="http://schema-registry:8081"'
	```

- insert new records with the latest schema and validate whether the change events will be applied to delta lake.
	```
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (388, 98558.51, 'Pontiac', 'G5', 'Leeds', 'Dibbert and Sons', '20204 Kipling Terrace');
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (800, 80068.02, 'Audi', 'S8', 'York', 'Price-Davis', '97 Shasta Park');
	```