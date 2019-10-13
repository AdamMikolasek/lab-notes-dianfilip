### 1. See how like with a leading pattern is slow
#### query with leading pattern on unindexed supplier_ico
- select * from documents where supplier_ico like '%3008'
> duration: ~1600ms

#### create index
- create index index_documents_on_supplier_ico on documents (supplier_ico text_pattern_ops)

#### query with leading pattern on indexed supplier_ico
- select * from documents where supplier_ico like '%3008'
>duration: ~1600ms

### 2. Try like with a trailing pattern
#### query with trailing pattern on unindexed supplier_ico
- select * from documents where supplier_ico like '45%'
> duration: ~1400ms

#### query with trailing pattern on indexed supplier_ico
- select * from documents where supplier_ico like '45%'
> duration: ~900ms

#### see index condition
- explain select * from documents where supplier_ico like '45%'
> plan:
"Bitmap Heap Scan on documents  (cost=524.18..44691.17 rows=23054 width=302)"
"  Filter: ((supplier_ico)::text ~~ '45%'::text)"
"  ->  Bitmap Index Scan on index_documents_on_supplier_ico  (cost=0.00..518.42 rows=22199 width=0)"
"        Index Cond: (((supplier_ico)::text ~>=~ '45'::text) AND ((supplier_ico)::text ~<~ '46'::text))"

#### query with nonsense input
- explain select * from documents where supplier_ico like 'ahoj%'
> plan:
"Index Scan using index_documents_on_supplier_ico on documents  (cost=0.43..8.45 rows=72 width=302)"
"  Index Cond: (((supplier_ico)::text ~>=~ 'ahoj'::text) AND ((supplier_ico)::text ~<~ 'ahok'::text))"
"  Filter: ((supplier_ico)::text ~~ 'ahoj%'::text)"

### 3. Make the suffix search fast
#### query with suffix input without index
- select * from documents where supplier_ico like '%3008'
> duration: ~340ms

#### create functional index with reversed supplier_ico column
- create index index_documents_on_supplier_ico on documents (reverse(supplier_ico) text_pattern_ops)

#### query with reversed suffix with index
- select * from documents where reverse(supplier_ico) like reverse('%3008')
> duration: ~210ms

### 5. Understand covering index
#### query list of customers on unindexed documents
- select customer from documents where department = 'Ministerstvo obrany SR'
> duration: ~280ms

#### create a simple index
- create index index_documents_on_department on documents (department)

#### query list of customers on indexed documents
- select customer from documents where department = 'Ministerstvo obrany SR'
> duration: ~160ms

- explain select customer from documents where department = 'Ministerstvo obrany SR'
> plan:
	"Bitmap Heap Scan on documents  (cost=271.32..11703.25 rows=7083 width=41)"
	"  Recheck Cond: ((department)::text = 'Ministerstvo obrany SR'::text)"
	"  ->  Bitmap Index Scan on index_documents_on_department  (cost=0.00..269.55 rows=7083 width=0)"
	"        Index Cond: ((department)::text = 'Ministerstvo obrany SR'::text)"

#### build a covering index
- create index index_documents_covering on documents(department, customer)
- set enable_bitmapscan = off;
- set enable_seqscan = off;
- select customer from documents where department = 'Ministerstvo obrany SR'
> duration: ~160ms (no change)

- explain select customer from documents where department = 'Ministerstvo obrany SR'
> plan:
	"Index Only Scan using index_documents_covering on documents  (cost=0.55..22783.49 rows=7083 width=41)"
	"  Index Cond: (department = 'Ministerstvo obrany SR'::text)"

