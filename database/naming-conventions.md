
DB Objects Naming Conventions

| Object | Convention    | Example    |
| :---:   | :---: | :---: |
| Table | Nouns   | DimCustomer, FactTestAttempt   |
| View | vw   | vwMedCustomerRecord   |
| Function | fn_ / udf_ / ufn_   | fn_FormatData, udf_GetTestScore, ufn_BuildPlan   |
| Stored Procedure | usp_   | usp_GetTestScore   |
| Primary Key | PK_TableName_FieldName   | PK_DimCustomer_CustomerId   |
| Foreign Key | FK_ReferencingTable_ReferencedTable_ReferencingColumn_ReferencedColumn   | FK_DimCustomer_DimTestScore_CustomerId_Cid   |
| Default Constraint | DF_TableName_FieldName   | DF_DimCustomer_CreatedDate, DF_DimCustomer_UpdatedDate   |
| Clustered Index | IXC_TableName_FieldNames   | IXC_DimCustomer_CustomerId_Record_Id   |
| Non Clustered Index | IXNC_TableName_FieldNames   | IXNC_DimCustomer_CustomerId_Record_Id   |
| * | *   | *   |
