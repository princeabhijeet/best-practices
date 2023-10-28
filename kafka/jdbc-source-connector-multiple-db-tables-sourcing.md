**Multiple DBs multiple tables sourcing using JDBC Source Connector**

POC has been done in SQL Server (EFX-QA-SQLRP01).

Technologies used:

1.	SQL Server
     * SQL – Table, View
     * CDC (Change Data Capture) in SQL Server – CT Tables
     
2.	Kafka
     * Kafka Topic
     * JDBC Source Connector
     * JDBC Sink Connector

**POC**: Build a data flow pipeline where data should flow from tables of same schema present in multiple databases within same database server by getting sourced from a JDBC Source Connector to a single Kafka topic - which can later by sink-ed to a sink table by using JDBC Sink Connector.

1.	Given: Consider single SQL Server: EFX-QA-SQLRP01
2.	Two databases with same table (same schema) - acting as source tables:

    a.	ManagerTrackingDM - dbo.Customer

	```
	CREATE TABLE ManagerTrackingDM.dbo.Customer (
	        CustomerID        INT              PRIMARY KEY  IDENTITY(1,1),
	        Tenant            NVARCHAR(50),
	        FirstName         NVARCHAR(50)     NOT NULL,
	        LastName          NVARCHAR(50)     NOT NULL,
	        Email             NVARCHAR(100)    UNIQUE      NOT NULL,
	        Phone             NVARCHAR(20),
	        CreatedDate       DATETIME2        DEFAULT     SYSDATETIME(),
	        UpdatedDate       DATETIME2        DEFAULT     SYSDATETIME()
	);
	```

    b.	ReportSubscription - dbo.Customer

	```
    	CREATE TABLE ReportSubscription.dbo.Customer (
	        CustomerID        INT              PRIMARY KEY  IDENTITY(1,1),
	        Tenant            NVARCHAR(50),
	        FirstName         NVARCHAR(50)     NOT NULL,
	        LastName          NVARCHAR(50)     NOT NULL,
	        Email             NVARCHAR(100)    UNIQUE      NOT NULL,
	        Phone             NVARCHAR(20),
	        CreatedDate       DATETIME2        DEFAULT     SYSDATETIME(),
	        UpdatedDate       DATETIME2        DEFAULT     SYSDATETIME()
    	);
	```

3.	Enable CDC on source tables:

    a.	[ManagerTrackingDM].[dbo].[Customer] = [ManagerTrackingDM].[cdc].[dbo_Customer_CT]

	```
	USE [ManagerTrackingDM];
	GO
	
	EXECUTE sp_cdc_enable_db
	GO
	
	IF OBJECT_ID('cdc.dbo_Customer_CT') IS NULL
	EXECUTE sys.sp_cdc_enable_table 
		@source_schema 		= N'dbo',
		@source_name 		= N'Customer',
		@role_name 		= N'MTcdcReader',
		@supports_net_changes 	= 1;
	GO 
	```

    b.	[ReportSubscription].[dbo].[Customer] = [ReportSubscription].[cdc].[dbo_Customer_CT]

	```
	USE [ReportSubscription];
	GO
	
	EXECUTE sp_cdc_enable_db
	GO
	
	IF OBJECT_ID('cdc.dbo_Customer_CT') IS NULL
	EXECUTE sys.sp_cdc_enable_table 
		@source_schema 		= N'dbo',
		@source_name 		= N'Customer',
		@role_name 		= N'MTcdcReader',
		@supports_net_changes 	= 1;
	GO 
	```

4.	Create a database view to be executed by JDBC Source Connector:

    a.	[ManagerTrackingDM].[dbo].[vwMedHubCustomerRecord]

	```
	USE [ManagerTrackingDM]
	GO
	
	SET ANSI_NULLS ON
	GO
	
	SET QUOTED_IDENTIFIER ON
	GO
	
	CREATE OR ALTER VIEW [dbo].[vwMedHubCustomerRecord]
	AS
	SELECT CustRecord.CustomerId,
	CustRecord.Tenant,
	CustRecord.FirstName,
	CustRecord.LastName,
	CustRecord.Email,
	CustRecord.Phone,
	CustRecord.CreatedDate,
	CustRecord.UpdatedDate,
	CAST(CustRecord.ChangeDate AS DATETIME2) AS ChangeTime

	FROM
	(
		SELECT MgrCust.*,
			CAST(COALESCE(CLTM.tran_end_time, Getdate()) AS DATETIME2) AS ChangeDate
			FROM ManagerTrackingDM.cdc.dbo_Customer_CT AS MgrCust WITH (NOLOCK)
				INNER JOIN ManagerTrackingDM.cdc.lsn_time_mapping AS CLTM WITH (NOLOCK) 
				ON MgrCust.__$start_lsn = CLTM.start_lsn
			WHERE (MgrCust.__$operation = 4 OR MgrCust.__$operation = 2)
		
		UNION ALL
		
		SELECT RptCust.*,
			CAST(COALESCE(CLTM.tran_end_time, Getdate()) AS DATETIME2) AS ChangeDate
			FROM ReportSubscription.cdc.dbo_Customer_CT AS RptCust WITH (NOLOCK)
				INNER JOIN ReportSubscription.cdc.lsn_time_mapping AS CLTM WITH (NOLOCK) 
				ON RptCust.__$start_lsn = CLTM.start_lsn
			WHERE (RptCust.__$operation = 4 OR RptCust.__$operation = 2)
	) 
	AS CustRecord
	```

5.	Create a topic in which source connector will publish message:

    a.	med.reporting.customer

6.	Create JDBC Source Connector & refer the view [ManagerTrackingDM].[dbo].[vwMedHubCustomerRecord]:

    a.	MED-Reporting-Customer-Source-incr

	```
	{
		"name": "MED-Reporting-Customer-Source-incr",
		"config": {
			"connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
			"name": "MED-Reporting-Customer-Source-incr",
			"mode": "timestamp",
			"timestamp.column.name": "ChangeTime",
			"topic.prefix": "med.reporting.customer",
			"tasks.max": "1",
			"connection.user": "${azurekv:aks-ana-efx-nonprodkv:kafkaconnect-jdbc-user}",
			"connection.password": "${azurekv:aks-ana-efx-nonprodkv:kafkaconnect-jdbc-password}",
			"connection.url": "jdbc:sqlserver://EFX-QA-SQLRP01:1433;databaseName=ManagerTrackingDM",
			"query": "Select * from [ManagerTrackingDM].[dbo].[vwMedHubCustomerRecord]"
		}
	}
	```
7.	Create a sink table in any one of the databases which will sink data from topic (med.reporting.customer)

    a.	[ManagerTrackingDM].[dbo].[CustomerSink]

	```
	CREATE TABLE ManagerTrackingDM.dbo.CustomerSink (
	        CustomerID        INT              PRIMARY KEY  IDENTITY(1,1),
	        Tenant            NVARCHAR(50),
	        FirstName         NVARCHAR(50)     NOT NULL,
	        LastName          NVARCHAR(50)     NOT NULL,
	        Email             NVARCHAR(100)    UNIQUE      NOT NULL,
	        Phone             NVARCHAR(20),
	        CreatedDate       DATETIME2        DEFAULT     SYSDATETIME(),
	        UpdatedDate       DATETIME2        DEFAULT     SYSDATETIME()
	);
	```

8.	Create a JDBC Sink Connector which will read data from topic (med.reporting.customer) and sink into [ManagerTrackingDM].[dbo].[CustomerSink]

    a.	MED-Reporting-Customer-Sink-incr

	```
    	{
 		"name": "MED-Reporting-Customer-Sink-incr",
		"config": {
			"connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
			"table.name.format": "CustomerSink",
			"fields.whitelist": "CustomerId,Tenant,FirstName,LastName,Email,Phone",
			"connection.password": "${azurekv:aks-ana-efx-nonprodkv:kafkaconnect-jdbc-password}",
			"tasks.max": "1",
			"topics": "med.reporting.customer",
			"connection.user": "${azurekv:aks-ana-efx-nonprodkv:kafkaconnect-jdbc-user}",
			"name": "MED-Reporting-Customer-Sink-incr",
			"connection.url": "jdbc:sqlserver://EFX-QA-SQLRP01:1433;databaseName=ManagerTrackingDM",
			"insert.mode": "upsert",
			"pk.mode": "record_value",
			"pk.fields": "CustomerId,Tenant"
		}
	}
	```

9.	Test POC:

    a.	INSERT few dummy rows in [ManagerTrackingDM].[dbo].[Customer] and [ReportSubscription].[dbo].[Customer] and verify sink connector should get new records in [ManagerTrackingDM].[dbo].[CustomerSink]

	```
    	INSERT INTO ManagerTrackingDM.dbo.Customer (Tenant, FirstName, LastName, Email, Phone) 
    	VALUES ('T1', 'Prince', 'A', 'Prince.A@email.com', '123-456-7890');
    

    	INSERT INTO ReportSubscription.dbo.Customer (Tenant, FirstName, LastName, Email, Phone) 
    	VALUES ('T2', 'Joy', 'Toy', 'Joy.Toy@email.com', '900-123-4544');
	```

    b.	Update a record in [ManagerTrackingDM].[dbo].[Customer] and [ReportSubscription].[dbo].[Customer] and verify sink connector should update existing records in [ManagerTrackingDM].[dbo].[CustomerSink]

	```
    	UPDATE ManagerTrackingDM.dbo.Customer SET FirstName='Joe' WHERE CustomerId=1     
   
    	UPDATE ReportSubscription.dbo.Customer SET FirstName='Jan' WHERE CustomerId=1
	```

10.	Verify if topic offsets are increasing after each insert or update as per previous step.


Dependency:
In this POC, there is use of SQL Server’s CDC (Change Data Capture) – CT Tables. Hence, this POC directly depends on CT Tables (once enabled) provided by SQL Server. To do similar POC using MySQL, we don’t have similar CT tables that we can make use of. The CDC in MySQL is not directly provided by MySQL but is provided by third party (based on initial research). These third-party providers don’t generate CT tables like SQL Server does. They read MySQL’s binary log file and somehow show changes in Kafka topics (example: using Debezium connector) unlike showing changes in CT tables (CT tables are not generated in MySQL databases). Even if we make use of third-party connectors like Debezium, we have no flexibility like CT tables as we can make JOIN conditions on CT tables like we did in Step 4.a.
