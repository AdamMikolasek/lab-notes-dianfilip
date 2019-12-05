### 1. Try benchmarking PostgreSQL write performance with and without synchronous_commit option set. The metric that you should measure is inserts per seconds - i.e., how many inserts can the database process in 1 second.

Script:
```
from sqlalchemy import create_engine  
from datetime import datetime, timedelta  
import os, binascii, random  
  
  
engine = create_engine('postgresql://postgres@localhost:5433/oz')  
  
  
def insert():  
  end_time = datetime.now() + timedelta(seconds=1)  
  count = 0  
  while datetime.now() < end_time:  
        rand1 = str(binascii.b2a_hex(os.urandom(5)))  
        rand2 = str(binascii.b2a_hex(os.urandom(5)))  
        rand3 = str(random.random() * 100000000)  
  
        sql = "insert into documents(name, type, created_at, department, contracted_amount) values ('Generated doc " + rand1[2:10] + \  
                        "', 'MyType', now(), 'Department " + rand2[2:10] + "', " + rand3 + ")"  
	    con.execute(sql)  
        count += 1  
  return count  
  
  
with engine.connect() as con:  
  total_count = 0  
  
  for x in range(10):  
        insert_count = insert()  
        total_count += insert_count  
  
  print('total count in 10s: ' + str(total_count))  
  print('average count in 1s: ' + str(total_count / 10))
```

Now, run the benchmark against the stock docker image. How many inserts per second did it handle?

Result:

>total count in 10s: 2677
average count in 1s: 267.7

Stop the docker container and run it again, but with different options.

Result:

>total count in 10s: 5944
average count in 1s: 594.4
