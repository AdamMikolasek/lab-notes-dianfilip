
### 1. Write a recursive query which returns a number and its factorial for all numbers up to 10.
	with recursive fact(n,fact) as (
		select 0, 1
		union all
		select f.n + 1, f.fact * (f.n + 1) from fact as f
	)
	select * from fact limit 10
	
>result:
		0	1
		1	1
		2	2
		3	6
		4	24
		5	120
		6	720
		7	5040
		8	40320
		9	362880
	
### 2. Write a recursive query which returns a number and the number in Fibonacci sequence at that position for the first 20 Fibonacci numbers.
	with recursive fibonacci(por, fib, n) as (
		select 1, 1, 1
		union all
		select  f.por + 1, f.n, f.fib + f.n from fibonacci as f
	)
	select por, n from fibonacci limit 20
	
>result:
		1	1
		2	2
		3	3
		4	5
		5	8
		6	13
		7	21
		8	34
		9	55
		10	89
		11	144
		12	233
		13	377
		14	610
		15	987
		16	1597
		17	2584
		18	4181
		19	6765
		20	10946
	
### 3. Table product_parts contains products and product parts which are needed to build them. A product part may be used to assemble another product part or product, this is stored in the part_of_id column. When this column contains NULL it means that it is the final product. List all parts and their components that are needed to build a 'chair'.
	with recursive product_r(name) as (
		select parts.name, parts.id from product_parts product cross join product_parts parts where parts.part_of_id = product.id and product.name = 'chair'
		union all
		select parts.name, parts.id from product_r product cross join product_parts parts where parts.part_of_id = product.id
	)
	select name from product_r

> result:
		"armrest"
		"metal leg"
		"metal rod"
		"cushions"
		"red dye"
		"cotton"

### 4. List all bus stops between 'Zochova' and 'Zoo' for line 39. Also include the hop number on that trip between the two stops.
	with recursive stops_r(name, id, hop) as (
		select ss.name, ss.id, 0 from stops ss
	 	where ss.name = 'Zochova'
	
		union
	
		select se.name, se.id, hop + 1 from stops_r ss
		join connections cc on ss.id = cc.end_stop_id
		join stops se on cc.start_stop_id = se.id
		where se.name != 'Zoo' and cc.line = '39'
	)
	select name, hop from stops_r where hop != 0
	
>result:
		"Chatam Sófer"	1
		"Park kultúry"	2
		"Lafranconi"	3



























	
