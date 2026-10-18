# SQL Server Indexes: Clustered vs Nonclustered

## 1. Introduction

An index in SQL Server is a data structure that helps retrieve records efficiently.

SQL Server table storage can be classified into two main types:

- **Heap:** A table without a clustered index.
- **Clustered table:** A table whose data rows are stored at the leaf level of a clustered index.

A table can also have nonclustered indexes, which maintain separate index structures to help retrieve data efficiently.

## 2. Heap

A heap is a table that does not have a clustered index.

### Example

```sql
CREATE TABLE Employees
(
    EmployeeId INT,
    Name VARCHAR(50),
    Salary INT
);
```

If no clustered index is created, SQL Server stores the table as a heap.

### Storage structure

```text
Heap
 |
 +-- Data Page 1
 |    +-- EmployeeId: 3, Athul, 40000
 |    +-- EmployeeId: 1, Arun, 30000
 |
 +-- Data Page 2
      +-- EmployeeId: 4, Anu, 60000
      +-- EmployeeId: 2, Rahul, 50000
```

*Note: This is an illustrative layout. SQL Server does not guarantee insertion order or a particular physical arrangement of rows.*

### Important points

- A heap does not have a clustered-index B-tree.
- Actual table rows are stored in heap data pages.
- Rows are not logically ordered by a clustered index key.
- A RID (Row Identifier) can identify a row by its file, page, and slot.
- Query results are not guaranteed to be in any particular order without `ORDER BY`.

## 3. Clustered Index

A clustered index organizes table rows by its index key. Its B-tree leaf level contains the actual table data.

### Example

```sql
CREATE CLUSTERED INDEX IX_Employees_EmployeeId
ON Employees (EmployeeId ASC);
```

### Clustered index structure

```text
           Root Page
               |
               v
       Intermediate Pages
               |
               v
        Leaf-Level Pages
        (Actual Table Data)
               |
       +-------+-------+
       |       |       |
       v       v       v
   Employee  Employee Employee
   ID: 1     ID: 2    ID: 3
```

### What is stored at each level?

| Level | Contents | Purpose |
|---|---|---|
| Root page | Navigation entries | Starts the search |
| Intermediate pages | Navigation entries | Directs the search |
| Leaf-level pages | Complete table rows | Stores the actual table data |

### Important points

- The leaf level of the clustered index IS the table data.
- There is no separate heap-like data structure underneath the clustered index.
- The leaf-level rows are logically ordered by the clustered index key.
- Pages do not have to occupy consecutive physical locations on disk.
- Query output order is not guaranteed without `ORDER BY`.

Example:

```sql
SELECT *
FROM Employees
ORDER BY EmployeeId ASC;
```

This query explicitly requests ascending order by `EmployeeId`.

## 4. Nonclustered Index

A nonclustered index is a separate B-tree structure that stores index key values and row locators pointing to the corresponding table rows.

### Example

```sql
CREATE NONCLUSTERED INDEX IX_Employees_Salary
ON Employees (Salary ASC);
```

### Nonclustered index structure

```text
          Root Page
              |
              v
      Intermediate Pages
              |
              v
     Leaf-Level Index Pages
       +------------------+
       | Salary: 30000    |
       | Row Locator      |
       +------------------+
       | Salary: 40000    |
       | Row Locator      |
       +------------------+
       | Salary: 50000    |
       | Row Locator      |
       +------------------+
              |
              | Row locators identify rows
              v
      Underlying Table Data
      (Separate Structure)
```

*Note: The underlying table data is not a child level of the nonclustered index. The row locators identify the rows in the separate table structure.*

### What is stored at each level?

| Level | Contents | Purpose |
|---|---|---|
| Root page | Navigation entries | Starts the search |
| Intermediate pages | Navigation entries | Guides the search |
| Leaf-level pages | Index keys, row locators and possibly included columns | Identifies matching rows |
| Underlying table | Actual table rows | Stores the complete row data |

### Row locators

The row locator depends on the table's storage type:

- **Heap:** The locator is a RID (Row Identifier).
- **Clustered table:** The locator uses the clustered index key to identify the row through the clustered index.

## 5. How a Nonclustered Index Retrieves Data

Consider the query:

```sql
SELECT *
FROM Employees
WHERE Salary = 40000;
```

SQL Server can perform these steps:

1. Search the nonclustered index B-tree.
2. Find the matching entry at the leaf level.
3. Read its row locator.
4. Retrieve the actual row if additional columns are needed.

### When the table is a heap

```text
Nonclustered Index
        |
        v
Matching Leaf Entry
        |
        v
       RID
        |
        v
   Heap Data Row
```

The operation that retrieves the heap row is commonly called a **RID Lookup**.

### When the table has a clustered index

```text
Nonclustered Index
        |
        v
Matching Leaf Entry
        |
        v
Clustered Index Key
        |
        v
Clustered Index B-tree
        |
        v
Actual Table Row
```

The operation that retrieves the clustered row is commonly called a **Key Lookup**.

SQL Server may avoid these lookups when the index already contains every column required by the query.

## 6. Covering Nonclustered Index

A covering index contains all the columns required by a particular query.

### Example

```sql
CREATE NONCLUSTERED INDEX IX_Employees_Salary
ON Employees (Salary)
INCLUDE (Name);
```

Query:

```sql
SELECT Salary, Name
FROM Employees
WHERE Salary = 40000;
```

Because both `Salary` and `Name` are available in the index, SQL Server can potentially return the result directly from the index without looking up the underlying table row.

### Key columns vs included columns

- **Key columns:** Determine the index's search and ordering keys.
- **Included columns:** Store additional values at the leaf level to help cover queries.

Included columns do not determine the index's key ordering.

## 7. Composite Index and Sort Direction

A composite index contains multiple key columns.

### Example

```sql
CREATE NONCLUSTERED INDEX IX_Employees_Salary_Name
ON Employees (Salary DESC, Name ASC);
```

This index orders entries by:

1. Salary in descending order.
2. Name in ascending order within each salary value.

Matching query:

```sql
SELECT *
FROM Employees
ORDER BY Salary DESC, Name ASC;
```

### Meaning of ASC and DESC

- `ASC`: Ascending order.
- `DESC`: Descending order.

These specify the sort direction of index key columns. They do not change whether an index is clustered or nonclustered.

SQL Server can scan an index backward when the requested ordering is compatible with a reverse scan. Mixed sort directions can be important when designing composite indexes.

## 8. Clustered vs Nonclustered Index

| Feature | Heap | Clustered Index | Nonclustered Index |
|---|---|---|---|
| B-tree for table storage | No | Yes | Has its own B-tree |
| Actual table rows | Heap data pages | Clustered index leaf pages | Stored separately |
| Leaf-level contents | Not applicable to the heap itself | Complete table rows | Index entries and row locators |
| Row locator used to reach table data | Not applicable to the heap itself | Not needed for its own leaf rows | Yes, when a separate table lookup is needed |
| Number allowed per table | One heap storage organization | At most one clustered index | Multiple nonclustered indexes |
| Guaranteed query output order | No | No | No |

## 9. Complete SQL Example

```sql
CREATE TABLE Employees
(
    EmployeeId INT NOT NULL,
    Name VARCHAR(50),
    Salary INT
);

-- Create a clustered index
CREATE CLUSTERED INDEX IX_Employees_EmployeeId
ON Employees (EmployeeId ASC);

-- Create a nonclustered index
CREATE NONCLUSTERED INDEX IX_Employees_Salary
ON Employees (Salary ASC);
```

Example queries:

```sql
-- Search using the clustered index key
SELECT *
FROM Employees
WHERE EmployeeId = 3;

-- Search using the nonclustered index key
SELECT *
FROM Employees
WHERE Salary = 40000;
```

SQL Server chooses an execution plan based on the query, available indexes, statistics, and estimated costs. It may use an index seek, index scan, lookup, or another access method.

## 10. Final Summary

1. **Heap:** Table rows are stored in data pages without a clustered-index B-tree.
2. **Clustered index:** Its leaf-level pages contain the actual table rows.
3. **Nonclustered index:** Its leaf-level pages contain index entries and row locators.
4. **Key Lookup:** Retrieves missing data from a clustered table.
5. **RID Lookup:** Retrieves missing data from a heap.
6. **Covering index:** Contains all columns needed by a query, potentially avoiding additional lookups.
7. **ASC/DESC:** Specify the sort direction of index key columns.
8. **ORDER BY:** Required to guarantee the order of query results.
