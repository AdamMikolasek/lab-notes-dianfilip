
### 1. Implement fulltext search in contracts. The search should be case insensitive and accents insensitive. It should support searching by

contract name
department
customer
supplier
by supplier ICO
and by contract code (anywhere inside contract code, such as OIaMIS)
Boost newer contracts, so they have a higher chance of being at top positions.

Try to find some limitations of your solutions - find a few queries (2 cases are enough) that are not showing expected results, or worse, are not showing any results. Think about why it's happening and propose a solution (no need to implement it).

- create text search configuration sk(copy = simple);
	create extension unaccent;
	alter text search configuration sk alter mapping for word with unaccent, simple;

- alter table contracts add fulltext_vector tsvector;
	update contracts set fulltext_vector = to_tsvector('sk', name || ' ' || department || ' ' || customer || ' ' || supplier);

- create extension pg_trgm;
	create index index_contracts_on_ico on contracts using gin((supplier_ico::text) gin_trgm_ops);
	create index index_contracts_on_identifier on contracts using gin(identifier gin_trgm_ops);
	create index index_contracts_on_fulltext on contracts using gin(fulltext_vector);

- select * from contracts
	where fulltext_vector @@ to_tsquery('sk', 'Úrad');
	
>plan:
		"Bitmap Heap Scan on contracts  (cost=34.41..8363.73 rows=2375 width=304) (actual time=44.730..330.911 rows=74821 loops=1)"
		"  Recheck Cond: (fulltext_vector @@ '''urad'''::tsquery)"
		"  Heap Blocks: exact=26090"
		"  ->  Bitmap Index Scan on index_contracts_on_fulltext  (cost=0.00..33.81 rows=2375 width=0) (actual time=27.532..27.532 rows=74821 loops=1)"
		"        Index Cond: (fulltext_vector @@ '''urad'''::tsquery)"
		"Planning time: 0.252 ms"
		"Execution time: 397.979 ms"

- select ts_rank(to_tsvector('sk', name || ' ' || department || ' ' || customer || ' ' || supplier || ' ' || identifier || ' '|| supplier_ico),
			   to_tsquery('sk', 'Úrad')) * (1 / (extract(days from (now() - published_on)))) as rank, * from contracts
	where fulltext_vector @@ to_tsquery('sk', 'Úrad')
	order by rank desc
	
> plan:
		"Sort  (cost=8615.65..8621.58 rows=2375 width=304) (actual time=6094.607..6237.189 rows=74821 loops=1)"
		"  Sort Key: ((ts_rank(to_tsvector('sk'::regconfig, ((((((((((name || ' '::text) || (department)::text) || ' '::text) || (customer)::text) || ' '::text) || (supplier)::text) || ' '::text) || (identifier)::text) || ' '::text) || (supplier_ico)::text)), '''urad'''::tsquery) * (1::double precision / date_part('days'::text, (now() - (published_on)::timestamp with time zone)))))"
		"  Sort Method: external merge  Disk: 49176kB"
		"  ->  Bitmap Heap Scan on contracts  (cost=34.41..8482.48 rows=2375 width=304) (actual time=23.625..5595.864 rows=74821 loops=1)"
		"        Recheck Cond: (fulltext_vector @@ '''urad'''::tsquery)"
		"        Heap Blocks: exact=26090"
		"        ->  Bitmap Index Scan on index_contracts_on_fulltext  (cost=0.00..33.81 rows=2375 width=0) (actual time=17.015..17.015 rows=74821 loops=1)"
		"              Index Cond: (fulltext_vector @@ '''urad'''::tsquery)"
		"Planning time: 0.476 ms"
		"Execution time: 6325.900 ms"
