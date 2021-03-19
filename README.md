# Recommender with Apache Mahout
Part 1 - Building a Recommender
1. Start up an EMR cluster
  ![image](https://user-images.githubusercontent.com/36765343/111746871-9c239300-88b4-11eb-8ae2-ba04bbe70bb7.png)
2. Get the MovieLens data\
```wget http://files.grouplens.org/datasets/movielens/ml-1m.zip```
unzip ml-1m.zip
3. Convert ratings.dat, trade “::” for “,”, and take only the first three columns:\
```cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv```
4. Put ratings file into HDFS:\
```hdfs dfs -put ratings.csv /user/hadoop/ratings.csv```
5. Run the recommender job:\
```mahout recommenditembased --input /user/hadoop/ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE```
6. Look for the results in the part-files containing the recommendations:\
```hdfs dfs -ls /user/hadoop/recommendations```
```hdfs dfs -cat /user/hadoop/recommendations/part-r-00000 | head```

Part 2 - Building a Recommender Service
1. Get Twisted, and Klein and Redis modules for Python\
```pip3 install --user twisted```\
```pip3 install --user klein```\
```pip3 install --user redis```
2. Install Redis and start up the server\
```wget http://download.redis.io/releases/redis-2.8.7.tar.gz```
```tar xzf redis-2.8.7.tar.gz```
```cd redis-2.8.7```
```make```
```./src/redis-server &```
3. Build a web service that pulls the recommendations into Redis and responds to queries.(Create directory called script and put server.py file which incuded following content)
```
from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hdfs dfs -cat /user/hadoop/recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id 
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  a = str(i).split()

  # Put key, value into Redis
  r.set(a[0],a[1])

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+id+' are '+v.decode("utf-8")+'\n'


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:4200/30'

# Start up a listener on port 4200
run("localhost", 4200)
```
4. Start the web service.\
```twistd -noy server.py &```
5. Test the web service with user id “5”:\
```curl localhost:4200/10```
