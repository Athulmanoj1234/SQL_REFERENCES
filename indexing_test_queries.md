-- first copy the customers table data to the new tables ie the copied table

SELECT *
INTO [index_tuto_db].[dbo].[Customer_Index_Copy]
FROM [index_tuto_db].[dbo].[Customers]

SELECT * FROM [index_tuto_db].[dbo].[Customer_Index_Copy]

SELECT * FROM [index_tuto_db].[dbo].[Customer_Index_Copy] WHERE Name = 'Priya';

-- suppose we need to retrieve a row from the table customers ie in the row id 5
SELECT * FROM [index_tuto_db].[dbo].[Customer_Index_Copy]
     WHERE id = 4;   -- here full rows in the tables has been scanned because in here data pages ie new table is an heap structure

SELECT * FROM [index_tuto_db].[dbo].[Customer_Index_Copy]
     WHERE Age = 25;  

-- so this table shows heap table structured ie not in the sorted way for the data pages so we need to change from the heap structure to the clustered index structure
CREATE CLUSTERED INDEX idx_Customer_Index_Copy_Id ON [index_tuto_db].[dbo].[Customer_Index_Copy] (Id)
-- again running to create clustered index for the id 

-- now we are trying to create clustered index also for another columns also in the same table
CREATE CLUSTERED INDEX idx_Customer_Index_Copy_Name ON [index_tuto_db].[dbo].[Customer_Index_Copy] (Name) -- so we cannot create more than one clustered index for the table and the error shows like this - Cannot create more than one clustered index on table 'index_tuto_db.dbo.Customer_Index_Copy'. Drop the existing clustered index 'idx_Customer_Index_Copy_Id' before creating another.
-- after dropping the existing index on the id and when i again tried to create the clustered index it successfully created, but we are removing it because the name will not be unique but the id will be so dropping the named index and creating the id index

-- so we need to drop the esisting index to create clustered index for the other column
DROP INDEX idx_Customer_Index_Copy_Id ON [index_tuto_db].[dbo].[Customer_Index_Copy];

-- we are droppping named index and re-creating the id index on the customers copy table because name is not unique
DROP INDEX idx_Customer_Index_Copy_Name ON [index_tuto_db].[dbo].[Customer_Index_Copy];


-- we are going to create non-clustered index on the both name and age and refer from the heap structure
CREATE NONCLUSTERED INDEX idx_Customer_Index_Copy_Name_Age ON [index_tuto_db].[dbo].[Customer_Index_Copy] (Name, Age)
-- one thing sql automatically assigns ASC deafaultly to all the indexes while creating so it will be equalent to 
/* CREATE NONCLUSTERED INDEX idx_Customer_Index_Copy_Name_Age
ON [index_tuto_db].[dbo].[Customer_Index_Copy]
(
    Name ASC,
    Age ASC
); */