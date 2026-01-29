# crunchengine

Query engines are the invisible workhorses powering modern data infrastructure. Every time you run a SQL query against a database, execute a Spark job, or query a data lake, a query engine is transforming your high-level request into an efficient execution plan.

A query engine is simply software that retrieves and processes data based on some criteria. The difference between your for loop and a production query engine is scale, generality, and optimization, but the core idea is the same.

```sql
SELECT name, gpa
FROM students
WHERE gpa > 3.5;
```

This query expresses the same logic as our Python loop, but the query engine decides how to execute it efficiently. This separation of "what" from "how" is powerful. The same query can run against a small file or a distributed cluster of servers.

## Anatomy of a Query Engine
A query engine transforms a query (like the SQL above) into actual results through several stages:

Parsing: Convert the query text into a structured representation (like an abstract syntax tree)
Planning: Determine which operations are needed (scan, filter, join, aggregate)
Optimization: Reorder and transform operations for efficiency
Execution: Actually process the data and produce results

## Why Apache Columnar?

Traditional databases and programming languages typically store data row by row. If you have a table of employees, each employee record sits together in memory.

`SELECT AVG(salary) FROM employees WHERE department = 'Engineering'`

This query only needs the department and salary columns. With row-based storage, we would load entire rows into memory just to access two fields.

Columnar storage flips this around. Each column is stored contiguously:

```rust
ids:         [1, 2, 3]
names:       ["Alice", "Bob", "Carol"]
departments: ["Engineering", "Sales", "Engineering"]
salaries:    [95000, 87000, 102000]
```

Now reading salary means reading one contiguous block of memory, with no jumping around to skip unwanted fields. This matters enormously for performance:

- Modern CPUs load memory in cache lines (typically 64 bytes). Columnar data packs more useful values per cache line.
- Similar values grouped together compress much better. A column of departments might compress to just a few distinct values.
- CPUs can apply the same operation to multiple values simultaneously (Single Instruction, Multiple Data). Columnar layout enables this.

## Arrow memory layout
An Arrow column (called a "vector" or "array") consists of:

A data buffer containing the actual values, packed contiguously
A validity buffer, which is a bitmap indicating which values are null
An optional offset buffer for variable-length types like strings

For a column of 32-bit integers `[1, null, 3, 4]`:

```rust
Validity bitmap: [1, 0, 1, 1]  (bit per value: 1=valid, 0=null)
Data buffer:     [1, ?, 3, 4]  (? = undefined, since null)
```

For strings, we need offsets because strings vary in length:

```rust
Values: ["hello", "world", "!"]
Offsets: [0, 5, 10, 11]  (start position of each string, plus end)
Data:    "helloworld!"   (all strings concatenated)
```

> Fixed-width types like integers require just one memory access per value. The validity bitmap uses just one bit per value, so checking for nulls is cheap.

