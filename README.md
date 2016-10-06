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

The following are demonstrations of some spatial queries. The first one gets the number of tweets inside 
of Contra Costa County which has a FIPS code of 06013. We utilized `ST_Intersects`.
The result shows that there are about 85000 tweets in the county of interest.

```
query = "SELECT count(*) FROM counties, tweets \
         WHERE counties.geoid10='06013' \
         AND ST_Intersects(counties.geom, tweets.location);"

cur.execute(query)
cur.fetchall()
---
[(8502L,)]
```
The second spatial query gets the number of tweets that fall 100 miles outside 
Alameda County (FIPS code = 06001). We utilized `ST_Dwithin` and its negation to get number of features outside the buffer.
Depending on whether the unit of spatial reference system of the data is meter or not, `ST_Transform` may be used to convert unit to meter
for the purpose of distance calculation.

```
query = "SELECT count(*) FROM counties, tweets \
        WHERE counties.geoid10='06001' \
        AND NOT ST_Dwithin(ST_Transform(counties.geom, 3157), ST_Transform(tweets.location, 3157), 160934);"

cur.execute(query)
cur.fetchall()
---
[(14906L,)]
```

Following is an alternative way to get population data into database.

```
# Get the population data for California counties directly from the Census API 

c = Census(secrets.censuskey)

population = c.sf1.get('P0010001', geo={'for': 'county:*',
                                       'in': 'state:06'})
pop = pd.DataFrame(population)
pop.columns = ['population', 'county', 'state']
pop['geoid10'] = pop.state + pop.county
pop.head()
```

population | county | state | geoid10
--- | --- | --- | ---
1510271 | 001 | 06 | 06001
1175 | 003 | 06 | 06003
38091 | 005 | 06 | 06005
220000 | 007 | 06 | 06007
45578 | 009 | 06 | 06009

```
from sqlalchemy import create_engine
enginestring = 'postgresql://{}:{}@74.207.246.217:5432/tweets'.format(user, password)
engine = create_engine(enginestring)
pop.to_sql('population', engine)
conn.commit()
```

Population data is joined to counties and numbers of tweets by counties are summarized and joined to counties
as well. Based on common county name, two tables are further joined and then tweets per capital are calculated.

```
query = """
SELECT 

county_pop.name, 
county_tweets.count_tweets::float/county_pop.pop::float

FROM

    (SELECT counties.name10 as name, population.population as pop, counties.geom as geom 
    FROM population 
    INNER JOIN counties 
    ON counties.geoid10 = population.geoid10) county_pop
    
    INNER JOIN

    (SELECT counties.name10 as name, count(*) as count_tweets 
    FROM tweets, counties
    WHERE ST_Intersects(tweets.location, counties.geom)
    GROUP BY counties.name10) county_tweets

ON county_pop.name = county_tweets.name;

"""

cur.execute(query)
result = cur.fetchall()
```
The following tables shows the first several records of the query results. We also added counties without tweet
counts for visualization purpose.

NAME10 | tweets_per_capita
--- | ---
Orange|246.492629
Tehama|4222.933048
Colusa|NaN
Santa Barbara|179.289683
Mono|NaN
Monterey|1594.961656

County shapefile was first converted to geojson and joned with tweets_per_capita. Folium.Map is used 
to visualize the result, which is shown below:
![tweets_per_capita][figure1]

[figure1]:

Code is attached:
```
county_map = folium.Map(location=[38, -119], zoom_start=6, tiles="Mapbox Bright") 
county_map.choropleth(geo_path=geojson,
               data=pd.read_csv(os.path.join(os.getcwd(),'..','data','tweets_per_capita.csv')),
               columns= ['NAME10', 'tweets_per_capita'], 
               key_on='feature.properties.NAME10',
               fill_color='YlGn',
               fill_opacity=0.7,
               line_opacity=0.2,
               legend_name="Tweets per 1M people")

county_map.save('counties.html')
```