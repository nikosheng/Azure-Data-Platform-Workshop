
# Amazon Redshift Migration

## Getting Started

### Create Redshift Cluster
Navigate to AWS Redshift Console, we may create a redshift cluster `dc2.large` with trial usage, which is sufficient for our workshop. 

![Create Redshift Cluster][redshift-screenshot](pics/01_redshift_creation.png)

### Sample Data Preparation
- Create a mysql docker environment to generate the sample data files for further loading to redshift
	```
	## unload to any location without permission restriction 
	docker exec mysql bash -c "echo 'secure_file_priv =""' >> /etc/mysql/mysql.conf.d/mysqld.cnf" 

	## create sample table
	create table demo.orders ( id MEDIUMINT NOT NULL AUTO_INCREMENT PRIMARY KEY, customer_id INT, order_total_usd DECIMAL(11,2), make VARCHAR(50), model VARCHAR(50), delivery_city VARCHAR(50), delivery_company VARCHAR(50), delivery_address VARCHAR(50), create_ts TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, update_ts TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP );
	
	select * into outfile '/tmp/orderds.csv' 
	fields terminated by ',' optionally enclosed by '"' escaped by '"' 
	lines terminated by '\n' 
	from orders
	```

- Prepare for sample data and load into table `orders`. Below is the sampling rows and you may refer to the full dataset in folder `data/orders.sql`
	```
	use demo; 
	
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (88, 188032.36, 'Toyota', 'MR2', 'Sheffield', 'Farrell Inc', '42 Bultman Street'); 
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (104, 178148.43, 'Dodge', 'Grand Caravan', 'Bristol', 'Leffler-Nicolas', '76 Warrior Parkway'); 
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (825, 141212.99, 'Ford', 'F150', 'London', 'Hamill-Weber', '57126 Pankratz Point'); 
	insert into orders (customer_id, order_total_usd, make, model, delivery_city, delivery_company, delivery_address) values (133, 78940.02, 'Nissan', 'Xterra', 'York', 'Beatty, Lindgren and Brakus', '70 Center Terrace');
	```
	
### Data Factory Integration Runtime
As our redshift cluster is in private subnet, which is not public accessed. Therefore, we need to create an `integration runtime`, which is a `agent` to help us get connected with redshift in the same networking environment. 

- Firstly, we need to create a `Windows` EC2 instance environment in AWS which is in the same VPC with redshift. Please ensure the `security group` rules are properly set.
- Secondly, we will install `Integration Runtime` client in the windows EC2 instance and connect to `Azure Data Factory` integration runtimes. For more information, please refer to [Create a self-hosted integration runtime - Azure Data Factory & Azure Synapse | Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-factory/create-self-hosted-integration-runtime?tabs=data-factory)
	![Integration runtime][IR-screenshot](https://github.com/nikosheng/Azure-Data-Platform-Workshop/blob/main/pics/02_IR.png)

- In addition, we need to install JDK/JRE runtime to perform the transfer operation in ADF, otherwise, you may encounter the errors below
	```
	ErrorCode=JreNotFound,'Type=Microsoft.DataTransfer.Common.Shared.HybridDeliveryException,Message=Java Runtime Environment cannot be found on the Self-hosted Integration Runtime machine. It is required for parsing or writing to Parquet/ORC files. Make sure Java Runtime Environment has been installed on the Self-hosted Integration Runtime machine.,Source=Microsoft.DataTransfer.Common,''Type=System.DllNotFoundException,Message=Unable to load DLL 'jvm.dll': The specified module could not be found. (Exception from HRESULT: 0x8007007E),Source=Microsoft.DataTransfer.Richfile.HiveOrcBridge,'
	```
- Once we finish the installation, we might need to copy the data stored in S3 to Redshift


-  Navigate to Redshift query editor V2 for an online editor. Create a redshift sample schema `azure` 
	```
	create schema azure
	```
- Create sample table `orders` for migration usage
	```
	create table dev.azure.orders ( id INT, customer_id INT, order_total_usd DECIMAL(11,2), make VARCHAR(50), model VARCHAR(50), delivery_city VARCHAR(50), delivery_company VARCHAR(50), delivery_address VARCHAR(50), create_ts TIMESTAMP, update_ts TIMESTAMP )distkey(customer_id);
	```
- Import data into redshift with `COPY` command
	```
		copy dev.azure.orders from 's3://nikofengazuremigration/redshift/orders.csv' 
		iam_role 'arn:aws:iam::893573916412:role/AmazonRedshiftS3Role' 
		TIMEFORMAT 'auto' 
		csv ;
	```
	
### ADF Pipeline Setup

- Create ADF Pipeline to export Amazon Redshift data to ADLS Gen2, and the pipeline will automatically triggered for data transformation, compression and delta lake export. Please refer to `ECommerce_Load_Redshift_To_Synapse_support_live.zip` and import the file in ADF to reproduce the pipeline.
- Trigger the pipeline
	![Trigger Pipeline][TP-screenshot](https://github.com/nikosheng/Azure-Data-Platform-Workshop/blob/main/pics/03_trigger_pipeline.png)
- Wait for the pipline to run automatically and it will trigger a subflow to merge the data into delta lake format by Azure Databricks
	![Wait Pipeline][WP-screenshot](https://github.com/nikosheng/Azure-Data-Platform-Workshop/blob/main/pics/04_pipeline_running.png)

### Azure Synapse Delta Lake Query

- Use serverless sql pool to query the delta lake data which is imported by ADF pipeline
	```
	CREATE DATABASE demo;

	CREATE EXTERNAL DATA SOURCE DeltaLakeStorage
	WITH ( LOCATION = 'https://msftnikoadlsgen2storage.blob.core.windows.net/ecommerce-gold/' );
	GO

	SELECT COUNT(*)
	FROM OPENROWSET(
			BULK 'orders',
			DATA_SOURCE = 'DeltaLakeStorage',
			FORMAT = 'delta'
		) as rows;
	```