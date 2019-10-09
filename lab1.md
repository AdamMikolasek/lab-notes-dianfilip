### Write a simple query filtering on a single column (e.g. supplier = '', or department = ''). Measure how long the query takes.
- select * from documents where supplier='Fyzická osoba'
> **duration:** ~255ms

### Add an index for the column and measure how fast is the query now. What plan did the database use before & after index?
- create index index_documents_on_supplier on documents(supplier)
> **plan before:**
> "Seq Scan on documents  (cost=0.00..17398.30 rows=2236 width=303)"
			"  Filter: ((supplier)::text = 'Fyzická osoba'::text)"
			
> **plan after:**
> "Bitmap Heap Scan on documents  (cost=73.75..5882.37 rows=2236 width=303)"
			"  Recheck Cond: ((supplier)::text = 'Fyzická osoba'::text)"
			"  ->  Bitmap Index Scan on index_documents_on_supplier  (cost=0.00..73.19 rows=2236 width=0)"
			"        Index Cond: ((supplier)::text = 'Fyzická osoba'::text)"

> **duration after:** ~180ms

### Write a simple range query on a single column (e.g., total_amount > 100000 and total_amount <= 999999999). Measure how long the query takes.
- select * from documents where pocet_stran > 0 and pocet_stran <= 5
> **duration:** ~1850 ms

### Add an index for the column and measure how fast is the query now. What plan did the database use before & after index?
- create index index_documents_pocet_stran on documents(pocet_stran)
> **plan before:**
> "Seq Scan on documents  (cost=0.00..18276.36 rows=245459 width=303)"
			"  Filter: ((pocet_stran > 0) AND (pocet_stran <= 5))"

> **plan after:**
> "Seq Scan on documents  (cost=0.00..18276.36 rows=245522 width=303)"
			"  Filter: ((pocet_stran > 0) AND (pocet_stran <= 5))"

> **duration after:** ~1850ms

### Drop all indexes on the documents table
- *dropping all indexes*

### Benchmark how long does it take to insert a batch of N rows into the documents table.
- insert into documents (select * from documents limit 100000)
> **duration:** ~600ms

### Create index on a single column in the documents table. Choose an arbitrary column.
- create index index_documents_on_customer on documents(customer)

### Benchmark how long does it take to insert a batch of N rows now. How much slower is the insert now?
- insert into documents (select * from documents limit 100000)
> **duration:** ~9s

### Repeat the benchmark with 2 indices and 3 indices.
#### 2 indexes: customer, department(+)
- insert into documents (select * from documents limit 100000)
> **duration:** ~19s
#### 3 indexes: customer, department, total_amount(+)
- insert into documents (select * from documents limit 100000)
> **duration:** ~22s

### Now drop all indices and try if there's a difference in insert performance when you have a single index over low cardinality column vs. high cardinality column.
#### low cardinality: column "type"
- create index index_documents_on_type on documents(type)
- insert into documents (select * from documents limit 100000)
> **duration:** ~5s
#### high cardinality: "column supplier_ico"
- create index index_documents_on_supplier_ico on documents(supplier_ico)
- insert into documents (select * from documents limit 100000)
> **duration:** ~2.5s

