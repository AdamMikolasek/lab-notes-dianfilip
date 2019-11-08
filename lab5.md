
### 1. Write a query which finds all restaurants (point, amenity = 'restaurant') within 1000 meters of 'Fakulta informatiky a informačných technológií STU' (polygon). Select the restaurant name and distance in meters. Sort the output by distance - closest restaurant first.
- with restaurants as (select * from planet_osm_point where amenity = 'restaurant')
	select restaurants.name, st_distance(fiit.way::geography, restaurants.way::geography) as dist from planet_osm_polygon as fiit 
	join restaurants on st_dwithin(fiit.way::geography, restaurants.way::geography, 1000)
	where fiit.name = 'Fakulta informatiky a informačných technológií STU'
	order by dist;
> result:
			"Bastion - Slovenská Koliba"	"35.742375541"
			"Drag"	"366.312228092"
			"Idyla"	"414.046695922"
			"Družba"	"622.942884737"
			"Venza (študentská jedáleň)"	"674.141796757"
			"Kamel"	"749.299621785"
			"Eat and Meet (študentská jedáleň)"	"771.717881845"
			"Riviera"	"825.566535564"
			"Seoul garden"	"833.397496062"

### 2. Check the query plan and measure how long the query takes. Now make it as fast as possible. Make sure to also use geo-indices, but don't expect large improvements. The dataset is small, and filtering in amenity='restaurant' will greatly limit the search space anyway.
- explain with restaurants as (select * from planet_osm_point where amenity = 'restaurant')
	select restaurants.name, st_distance(fiit.way::geography, restaurants.way::geography) as dist from planet_osm_polygon as fiit cross join restaurants where fiit.name = 'Fakulta informatiky a informačných technológií STU'
	and restaurants.name is not null and st_dwithin(fiit.way::geography, restaurants.way::geography, 1000) order by dist;
> plan:
		"Sort  (cost=3005.84..3005.84 rows=1 width=244) (actual time=29.020..29.029 rows=9 loops=1)"
		"  Sort Key: (_st_distance((fiit.way)::geography, (restaurants.way)::geography, 0::double precision, true))"
		"  Sort Method: quicksort  Memory: 25kB"
		"  CTE restaurants"
		"    ->  Seq Scan on planet_osm_point  (cost=0.00..620.78 rows=497 width=981) (actual time=0.022..5.573 rows=497 loops=1)"
		"          Filter: (amenity = 'restaurant'::text)"
		"          Rows Removed by Filter: 25965"
		"  ->  Nested Loop  (cost=0.00..2385.05 rows=1 width=244) (actual time=13.568..28.977 rows=9 loops=1)"
		"        Join Filter: (((fiit.way)::geography && _st_expand((restaurants.way)::geography, 1000::double precision)) AND ((restaurants.way)::geography && _st_expand((fiit.way)::geography, 1000::double precision)) AND _st_dwithin((fiit.way)::geography, (restaurants.way)::geography, 1000::double precision, true))"
		"        Rows Removed by Join Filter: 488"
		"        ->  CTE Scan on restaurants  (cost=0.00..9.94 rows=497 width=64) (actual time=0.027..6.738 rows=497 loops=1)"
		"        ->  Materialize  (cost=0.00..2089.09 rows=2 width=180) (actual time=0.005..0.026 rows=1 loops=497)"
		"              ->  Seq Scan on planet_osm_polygon fiit  (cost=0.00..2089.07 rows=2 width=180) (actual time=2.089..11.648 rows=1 loops=1)"
		"                    Filter: (name = 'Fakulta informatiky a informačných technológií STU'::text)"
		"                    Rows Removed by Filter: 49605"
		
> execution time: ~27ms
> 
#### create indexes
- create index index_planet_osm_point on planet_osm_point using gist((way::geography));
		create index index_planet_osm_polygon on planet_osm_polygon using gist((way::geography));
		create index index_planet_osm_polygon_name on planet_osm_polygon(name);
		create index index_planet_osm_point_amenity on planet_osm_point(amenity);
> plan:
		"Sort  (cost=625.54..625.55 rows=1 width=244) (actual time=12.259..12.268 rows=9 loops=1)"
		"  Sort Key: (_st_distance((fiit.way)::geography, (restaurants.way)::geography, 0::double precision, true))"
		"  Sort Method: quicksort  Memory: 25kB"
		"  CTE restaurants"
		"    ->  Bitmap Heap Scan on planet_osm_point  (cost=12.14..317.45 rows=497 width=981) (actual time=0.210..1.308 rows=497 loops=1)"
		"          Recheck Cond: (amenity = 'restaurant'::text)"
		"          Heap Blocks: exact=199"
		"          ->  Bitmap Index Scan on index_planet_osm_point_amenity  (cost=0.00..12.01 rows=497 width=0) (actual time=0.156..0.156 rows=497 loops=1)"
		"                Index Cond: (amenity = 'restaurant'::text)"
		"  ->  Nested Loop  (cost=4.31..308.08 rows=1 width=244) (actual time=1.155..12.224 rows=9 loops=1)"
		"        Join Filter: (((fiit.way)::geography && _st_expand((restaurants.way)::geography, 1000::double precision)) AND ((restaurants.way)::geography && _st_expand((fiit.way)::geography, 1000::double precision)) AND _st_dwithin((fiit.way)::geography, (restaurants.way)::geography, 1000::double precision, true))"
		"        Rows Removed by Join Filter: 488"
		"        ->  CTE Scan on restaurants  (cost=0.00..9.94 rows=497 width=64) (actual time=0.214..2.546 rows=497 loops=1)"
		"        ->  Materialize  (cost=4.31..12.12 rows=2 width=180) (actual time=0.001..0.003 rows=1 loops=497)"
		"              ->  Bitmap Heap Scan on planet_osm_polygon fiit  (cost=4.31..12.11 rows=2 width=180) (actual time=0.037..0.038 rows=1 loops=1)"
		"                    Recheck Cond: (name = 'Fakulta informatiky a informačných technológií STU'::text)"
		"                    Heap Blocks: exact=1"
		"                    ->  Bitmap Index Scan on index_planet_osm_polygon_name  (cost=0.00..4.30 rows=2 width=0) (actual time=0.029..0.029 rows=1 loops=1)"
		"                          Index Cond: (name = 'Fakulta informatiky a informačných technológií STU'::text)"
		
> execution time: ~12ms 
	
### 3. Update the query to generate a geojson. The output should be a single row containg a json array.
- select array_to_json(array_agg(row_to_json(r))) from (with restaurants as (select * from planet_osm_point where amenity = 'restaurant')
	select restaurants.name, st_distance(fiit.way::geography, restaurants.way::geography) as dist, st_asgeojson(restaurants.way)::json as way
	from planet_osm_polygon as fiit 
	join restaurants on st_dwithin(fiit.way::geography, restaurants.way::geography, 1000)
	where fiit.name = 'Fakulta informatiky a informačných technológií STU'
	order by dist) as r
> result:
		[
			{
				"name": "Bastion - Slovenská Koliba",
				"dist": 35.742375541,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0722601,
						48.1547198
					]
				}
			},
			{
				"name": "Drag",
				"dist": 366.312228092,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0672632,
						48.1509517
					]
				}
			},
			{
				"name": "Idyla",
				"dist": 414.046695922,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0779226,
						48.1538106
					]
				}
			},
			{
				"name": "Družba",
				"dist": 622.942884737,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0701073,
						48.147537
					]
				}
			},
			{
				"name": "Venza (študentská jedáleň)",
				"dist": 674.141796757,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0690275,
						48.1605604
					]
				}
			},
			{
				"name": "Kamel",
				"dist": 749.299621785,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0618877,
						48.150218
					]
				}
			},
			{
				"name": "Eat and Meet (študentská jedáleň)",
				"dist": 771.717881845,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0672253,
						48.1610779
					]
				}
			},
			{
				"name": "Riviera",
				"dist": 825.566535564,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0629032,
						48.1480213
					]
				}
			},
			{
				"name": "Seoul garden",
				"dist": 833.397496062,
				"way": {
					"type": "Point",
					"coordinates": [
						17.0772523,
						48.1610729
					]
				}
			}
		]
