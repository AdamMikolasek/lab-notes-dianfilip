### 1. Work with your GIS project data. Choose a simple but meaningful subset of the data and import it from PostgreSQL into Elasticsearch. Do not import it manually, but write a code which will do the import for you. Don't import all columns from the database, select 3-4 fields that are interesting.
### Write an Elasticsearch query which will allow you to search the imported objects by name - don't worry about search quality, a basic match query should be enough for this lab. Write some interesting (meaningful) aggregations (1-2 are enough, don't worry about nested aggregations).

#### Implemented search:

Find roads within districts and road codes.

Query:

```
with district as (select pol.name, pol.way from planet_osm_polygon pol
				  where (pol.admin_level = '8' or pol.admin_level = '6') and pol.name <> 'London'),
road as (select distinct pon.name as road_name, pon.ref, d.name as district_name from planet_osm_line pon
		 cross join district d
		 where st_contains(d.way, pon.way) and pon.highway = 'primary'
	  	 and pon.name is not null and pon.ref is not null)
select row_to_json(results)
from (select road_name, ref as code, district_name as district from road order by road_name) as results
```
#### Create index in ES:
(via postman)
put http://localhost:9200/roads

#### Mapping for index:
put http://localhost:9200/roads/_mapping
```
{
  "properties": {
    "road_name": {
      "type": "keyword"
    },
    "code" : {
     "type" : "keyword"
    },
    "district" : {
     "type" : "keyword"
    }
  }
}
```

#### Load data to ES.
Python script:
```from sqlalchemy import create_engine  
import requests  
  
  
engine = create_engine('postgresql://postgres:K1m2o3t4r5@localhost:5432/london_gis')  
  
with engine.connect() as con:  
  select = """with district as (select pol.name, pol.way from planet_osm_polygon pol where (pol.admin_level = '8' or pol.admin_level = '6') and pol.name <> 'London'),  
 road as (select distinct pon.name as road_name, pon.ref, d.name as district_name from planet_osm_line pon cross join district d where st_contains(d.way, pon.way) and pon.highway = 'primary' and pon.name is not null and pon.ref is not null) select row_to_json(results) from (select road_name, ref as code, district_name as district from road order by road_name) as results"""  
  result = con.execute(select)  
    for row in result:  
        line = row[0]  
        print(line)  
  
        r = requests.post('http://localhost:9200/roads/_doc', json=line)  
        print(r.text)
```

#### Check data:

get http://localhost:9200/roads/_search

output:

```
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1454,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "562a224BawiZRXeb1kP7",
                "_score": 1.0,
                "_source": {
                    "road_name": "Abbey Road",
                    "code": "A123",
                    "district": "London Borough of Barking and Dagenham"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "6K2a224BawiZRXeb10MT",
                "_score": 1.0,
                "_source": {
                    "road_name": "Abingdon Street",
                    "code": "A3212",
                    "district": "City of Westminster"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "6a2a224BawiZRXeb10Mv",
                "_score": 1.0,
                "_source": {
                    "road_name": "Acre Lane",
                    "code": "A2217",
                    "district": "London Borough of Lambeth"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "6q2a224BawiZRXeb10NA",
                "_score": 1.0,
                "_source": {
                    "road_name": "Acton Lane",
                    "code": "A4000",
                    "district": "London Borough of Brent"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "662a224BawiZRXeb10NU",
                "_score": 1.0,
                "_source": {
                    "road_name": "Acton Street",
                    "code": "A201",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "7K2a224BawiZRXeb10Nl",
                "_score": 1.0,
                "_source": {
                    "road_name": "Acton Street",
                    "code": "A5200",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "7a2a224BawiZRXeb10N8",
                "_score": 1.0,
                "_source": {
                    "road_name": "Addington Road",
                    "code": "A2022",
                    "district": "London Borough of Bromley"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "7q2a224BawiZRXeb2UML",
                "_score": 1.0,
                "_source": {
                    "road_name": "Addington Road",
                    "code": "A2022",
                    "district": "London Borough of Croydon"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "762a224BawiZRXeb2UND",
                "_score": 1.0,
                "_source": {
                    "road_name": "Addington Street",
                    "code": "A3200",
                    "district": "London Borough of Lambeth"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "8K2a224BawiZRXeb2UNV",
                "_score": 1.0,
                "_source": {
                    "road_name": "Airport Roundabout",
                    "code": "A1020",
                    "district": "London Borough of Newham"
                }
            }
        ]
    }
}
```

#### 1. aggregated search
Find which road corridors cross searched district and how many streets/roads belong to them.

```
{
  "query": {
  	"match": {
  		"district": "City of London"
  	}
  },
  "aggregations": {
  	"by_code": {
  		"filter" : { "term": { "district": "City of London"} },
            "aggregations" : {
                "codes" : { "terms": {"field": "code"} }
            }
  	}
  }
}
```

result:
```
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 42,
            "relation": "eq"
        },
        "max_score": 3.533257,
        "hits": [
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "B62a224BawiZRXeb2kTm",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Anchor and Hope Lane",
                    "code": "A2052",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "Lq2a224BawiZRXeb3USl",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Beresford Street",
                    "code": "A206",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "O62a224BawiZRXeb3kSI",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Bexley Road",
                    "code": "A210",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "Ta2a224BawiZRXeb30TV",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Blackwall Lane",
                    "code": "A2203",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "Tq2a224BawiZRXeb30Tm",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Blackwall Lane link",
                    "code": "A2203",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "Wa2a224BawiZRXeb4ESf",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Bostall Hill",
                    "code": "A206",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "j62a224BawiZRXeb5UR3",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Bugsby's Way",
                    "code": "A2052",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "r62a224BawiZRXeb6ER7",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Carlyle Road",
                    "code": "A2041",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "sK2a224BawiZRXeb6ESS",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Carlyle Way",
                    "code": "A2041",
                    "district": "Royal Borough of Greenwich"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "v62a224BawiZRXeb6UTM",
                "_score": 3.533257,
                "_source": {
                    "road_name": "Central Way",
                    "code": "A2041",
                    "district": "Royal Borough of Greenwich"
                }
            }
        ]
    },
    "aggregations": {
        "by_code": {
            "doc_count": 42,
            "codes": {
                "doc_count_error_upper_bound": 0,
                "sum_other_doc_count": 3,
                "buckets": [
                    {
                        "key": "A206",
                        "doc_count": 15
                    },
                    {
                        "key": "A208",
                        "doc_count": 5
                    },
                    {
                        "key": "A2041",
                        "doc_count": 4
                    },
                    {
                        "key": "A210",
                        "doc_count": 3
                    },
                    {
                        "key": "A200",
                        "doc_count": 2
                    },
                    {
                        "key": "A2016",
                        "doc_count": 2
                    },
                    {
                        "key": "A2052",
                        "doc_count": 2
                    },
                    {
                        "key": "A207",
                        "doc_count": 2
                    },
                    {
                        "key": "A211",
                        "doc_count": 2
                    },
                    {
                        "key": "A2203",
                        "doc_count": 2
                    }
                ]
            }
        }
    }
}
```

#### 2. aggregated search
Find districts which are crossed by searched road corridor and how many road/streets belong to the corridor inside them.
```
{
  "query": {
  	"match": {
  		"code": "A40"
  	}
  },
  "aggregations": {
  	"by_code": {
  		"filter" : { "term": { "code": "A40"} },
            "aggregations" : {
                "codes" : { "terms": {"field": "district"} }
            }
  	}
  }
}
```

result:
```
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 22,
            "relation": "eq"
        },
        "max_score": 4.1692457,
        "hits": [
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "_62a224BawiZRXeb2kNU",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Aldersgate Street",
                    "code": "A40",
                    "district": "City of London"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "U62a224BawiZRXeb4EQ4",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Bloomsbury Way",
                    "code": "A40",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "S62a224BawiZRXeb90Ux",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Drake Street",
                    "code": "A40",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "WK2a224BawiZRXeb-EWs",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Earnshaw Street",
                    "code": "A40",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "Kq2b224BawiZRXebCUa3",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "High Holborn",
                    "code": "A40",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "cq2b224BawiZRXebDkbR",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Holborn",
                    "code": "A40",
                    "district": "City of London"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "c62b224BawiZRXebDkbh",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Holborn",
                    "code": "A40",
                    "district": "London Borough of Camden"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "dK2b224BawiZRXebDkby",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Holborn Circus",
                    "code": "A40",
                    "district": "City of London"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "da2b224BawiZRXebD0YE",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "Holborn Viaduct",
                    "code": "A40",
                    "district": "City of London"
                }
            },
            {
                "_index": "roads",
                "_type": "_doc",
                "_id": "sK2b224BawiZRXebE0aE",
                "_score": 4.1692457,
                "_source": {
                    "road_name": "King Edward Street",
                    "code": "A40",
                    "district": "City of London"
                }
            }
        ]
    },
    "aggregations": {
        "by_code": {
            "doc_count": 22,
            "codes": {
                "doc_count_error_upper_bound": 0,
                "sum_other_doc_count": 0,
                "buckets": [
                    {
                        "key": "London Borough of Camden",
                        "doc_count": 11
                    },
                    {
                        "key": "City of London",
                        "doc_count": 9
                    },
                    {
                        "key": "City of Westminster",
                        "doc_count": 1
                    },
                    {
                        "key": "London Borough of Hammersmith and Fulham",
                        "doc_count": 1
                    }
                ]
            }
        }
    }
}
```
