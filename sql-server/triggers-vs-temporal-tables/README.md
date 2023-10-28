**Update of "CreatedDate" and "UpdatedDate" columns in SQL Server**
There are multiple ways by which we can update timestamp columns in a table.
DBAs never suggest use of triggers. We will understand how we can replace triggers with non-versioned temporal tables.

**Triggers**
   * Triggers re special stored procedure that automatically runs when an event occurs in the database server.
   * Triggers are executed within the transaction scope.
   * Internal working: When a trigger is created, trigger information, its associated event, and its linked table - all these metadata is stored in SQL Server's internal system tables & views like sys.triggers, sys.trigger_events, sys.trigger_event_types, sys.tables. When a trigger event occurs, SQL Server references these system tables & views to identify & execute the appropriate triggers. Usages of these extra system tables & views adds complexity and performance issues while using triggers.

**Temporal Tables**
(1) System versioned Temporal Tables
(2) Non versioned Temporal Tables

**System versioned Temporal Tables**: have two parts
(1) Current state of table
(2) History table
		- WITH (SYSTEM_VERSIONING=ON)
		- extra table to manage history state or previous versions of data (this history table makes temporal tables - a "system versioned" table)

**Non versioned Temporal Tables**: have one part (current state) with history off
(1) Current state of table
		- we don't write WITH (SYSTEM_VERSIONING=ON) - so that history tables are not created
		

**Creating Non-versioned Temporal Table**:

CREATE TABLE dbo.Customer (
    CREATE TABLE ManagerTrackingDM.dbo.Customer (
		CustomerID INT PRIMARY KEY IDENTITY(1,1),
		FirstName NVARCHAR(50) NOT NULL,
		LastName NVARCHAR(50) NOT NULL,
		Email NVARCHAR(100) UNIQUE NOT NULL,
		Phone NVARCHAR(20),
		CreatedDate DATETIME2 DEFAULT (SYSUTCDATETIME()) NOT NULL,
		UpdatedDate DATETIME2 **GENERATED ALWAYS** AS **ROW START** NOT NULL,
		**ValidUntil**  DATETIME2 **GENERATED ALWAYS** AS **ROW END** NOT NULL,
		**PERIOD FOR SYSTEM_TIME** (**UpdatedDate**, ValidUntil)
	);

(1) Objective: UpdatedDate column should be updated automaticall on every insert/update of a row of customer table.
(2) Above CREATE statement creates a Non-versioned Temporal Table as we have not included "WITH (SYSTEM_VERSIONING=ON)" due to which history table is not created.
(3) GENERATED ALWAYS: columns keep track of the update timestamp.
(4) When you create a temporal table, you need to specify both a “ROW START” and “ROW END” columns.
(5) ROW START and ROW END columns will live in both the base table and the history table (if any).
(6) ValidUnit columns might be appearing as an extra non-needed column for developers usages but this column is internally used by SQL Server to calculate ROW END - meaning till what DATE we need to keep updating the UpdatedDate column. Hence all column values under ValidUnitl column is by default set as "9999-12-31 23:59:59.9999999". Hence, we cannot drop the ValidUnitl column.
(7) But we can hide the ValidUnit column as it is only to be used internally. After hiding the ValidUntil column, when we do SELECT * FROM dbo.Customer WITH (NOLOCK); the output will also not include ValidUntil column. To hide the ValidUntil column, execute - ALTER TABLE dbo.SomeTable ALTER COLUMN ValidUntil ADD HIDDEN;
(8) Con1: The time stored will be updated by the system and always be UTC.
(9) Con2: A second column (ValidUntil) is needed. But it can be marked as hidden making it easier to ignore.
