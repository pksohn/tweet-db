# Analyzing Tweet Data with PostGIS 

*Using PostGIS to load and analyze tweet data  
Civil & Environmental Engineering 263n: Scalable Spatial Analytics at UC-Berkeley, Fall 2016  
By Yiyan Ge, Paul Sohn, Stephen Wong, Ruoying Xu, October 4, 2016*

## Part 2: Spatial Queries

For this assignment, we set up a PostgreSQL 9.3 installation with PostGIS 2.2 on a Linode server.
To load the Twitter data into PostGIS, we will:

* Load the data in a pandas DataFrame in Python
* Use the pandas `to_sql` method to load the DataFrame into a PostgreSQL table
* Create a PostGIS geometry column in the new table
* Update the geometry column using the `lat` and `lng` columns in the tweets table 

First, we will connect to the PostgreSQL database using sqlalchemy to take advantage of 
the pandas built in "to_sql" method.

```
from sqlalchemy import create_engine
enginestring = 'postgresql://{}:{}@74.207.246.217:5432/tweets'.format(user, password)
engine = create_engine(enginestring)

with pd.HDFStore(os.path.join(os.getcwd(), '..', 'data', 'tweets_1M.h5'), mode='r') as store:
    df = store.tweets_subset

df.to_sql('tweets', engine)
```

From now on, we will use the psycopg2 Python package to connect to the online database:

```
import psycopg2
conn = psycopg2.connect(database=db_name, user=user, password=password, host=host)
cur = conn.cursor()
```

We can check the columns in the loaded table to see if the DataFrame was loaded:

```
cur.execute("SELECT column_name, data_type FROM information_schema.columns \
            WHERE table_name = 'tweets';")
cur.fetchall()
```

Results of the query are as follows:

Column | Data Type
--- | ---
index | bigint
 id | bigint
 lat | double precision
 lng | double precision
 text | text
 timeStamp | timestamp without time zone
 user_id | bigint

The next step is to add a geometry column to turn this regular table into a 
PostGIS spatial table. We can use the `AddGeometryColumn` function for this.
Then we will update the column to generate spatial objects using the 
`ST_SetSRID` and `ST_Point` functions.

```
cur.execute("SELECT AddGeometryColumn ('tweets', 'location', 4326, 'POINT', 2);")
cur.execute("UPDATE tweets SET location = ST_SetSRID(ST_Point(lng, lat), 4326);")
```

A repeated query shows a new column:

Column | Data Type
--- | ---
location | USER-DEFINED

Now that the tweets are loaded into PostGIS as a spatial database, we can insert the county shapefile
downloaded from the Census website. 

```
shp2pgsql -I -W 'latin1' -s 4326 tl_2010_06_county10.shp counties | psql -h 74.207.246.217 -d tweets -U paul
```

We can check a few values of the FIPS codes of counties that were loaded. In addition to below,
we can load the table in QGIS to view the points.

```
cur.execute("SELECT geoid10 FROM counties LIMIT 5;")
cur.fetchall()
---
[('06059',), ('06103',), ('06011',), ('06083',), ('06051',)]
```

```
# Calculate the number of tweets inside of Contra Costa County.

query = "SELECT count(*) FROM counties, tweets \
         WHERE counties.geoid10='06013' \
         AND ST_Intersects(counties.geom, tweets.location);"

cur.execute(query)
cur.fetchall()
```