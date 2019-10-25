### 1. How far (air distance) is FIIT STU from the Bratislava main train station? The query should output the distance in meters without any further modification.
- with fiit_stu as (select way from public.planet_osm_polygon where name = 'Fakulta informatiky a informačných technológií STU'),
	stanica as(select way from public.planet_osm_polygon where name = 'Bratislava hlavná stanica')
	select st_distance((select way::geography from fiit_stu), (select way::geography from stanica))
> result:
			2527.990245823

### 2. Which other districts are direct neighbours with 'Karlova Ves'?
- with districts as (select * from planet_osm_polygon where admin_level = '9' and boundary = 'administrative')
	select districts.name from planet_osm_polygon as polygons cross join districts
	where polygons.admin_level = '9' and polygons.boundary = 'administrative' and polygons.name = 'Karlova Ves' 
	and st_touches(districts.way, polygons.way)
> result:
			"Devín"
			"Dúbravka"
			"Bratislava - mestská časť Staré Mesto"

### 3. Which bridges cross over the Danube river?
- with danube as (select * from planet_osm_line where name = 'Dunaj' and waterway = 'river'),
	bridges as (select * from planet_osm_line where bridge = 'yes' and name is not null)
	select distinct(bridges.name) from danube cross join bridges where st_intersects(danube.way, bridges.way)
> result:
			"Prístavný most"
			"Most SNP"
			"Petržalská električka - 1. časť"
			"Most Lafranconi"
			"Most Apollo"
			
### 4. Find the names of all streets in 'Dlhé diely' district.
- select distinct streets.name from planet_osm_line as streets cross join planet_osm_polygon as district
	where district.name = 'Dlhé diely' and streets.name is not null and st_contains(district.way, streets.way);
> result:
			"Dlhé diely II"
			"Vincenta Hložníka"
			"Beniakova"
			"Matejkova"
			"Majerníkova"
			"Tománkova"
			"Dlhé diely I"
			"Vyhliadka"
			"Albína Brunovského"
			"Jána Stanislava"
			"Kolískova"
			"Ľudovíta Fullu"
			"Jamnického"
			"Veternicová"
			"Kresánkova"
			"Hlaváčikova"
			"Na Kampárke"
			"Iskerníková"
			"Hany Meličkovej"
			"Cikkerova"
			"Pribišova"
			"Blyskáčová"
			
