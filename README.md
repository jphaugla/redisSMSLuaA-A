# redisSMSLuaA-A
Provides a quick-start example of using Redis Streams for parsing incoming messages in an active/active redis enterprise.
## Overview
A java producer will produce messages on a redis stream across two redis clusters in an active/active database.
A java consumer will consume from the stream and create redis structures from the streams messages.
Both a simple consumer and a consumer using LUA to eliminate round trips to the client java application are available.

## Redis Advantages for message partition streaming
 * Redis easily handles high write transaction volume
 * Redis enterprise scales vertically (large nodes)  and horizontally (many nodes)
 * Redis enterprise active/active allows adding to a stream from either "region"
 * Redis allows putting TTL on the hash

## Requirements
* Docker installed on your local system, see [Docker Installation Instructions](https://docs.docker.com/engine/installation/).

* When using Docker for Mac or Docker for Windows, the default resources allocated to the linux VM running docker are 2GB RAM and 2 CPU's. Make sure to adjust these resources to meet the resource requirements for the containers you will be running. More information can be found here on adjusting the resources allocated to docker.

[Docker for mac](https://docs.docker.com/docker-for-mac/#advanced)

[Docker for windows](https://docs.docker.com/docker-for-windows/#advanced)

* Another option available here is to run on redis enterprise on AWS using CloudFormation

## Links that help!
 * [Active/Active docker under crdt-application](https://github.com/RedisLabs/redis-for-dummies)
 * [Getting Started with Redis Streams and java github](https://github.com/tgrall/redis-streams-101-java)
 * [Stackoverflow LUA with DICT](https://stackoverflow.com/questions/58999662/redis-how-to-hmset-a-dictionary-from-a-lua-script)
 * [Redis Active/Active CLI reference](https://docs.redis.com/latest/rs/references/crdb-cli-reference/)
## Create Redis Enterprise Active/Active database (Option 1)
 * Prepare Docker environment-see the Prerequisites section above...
 * Clone the github 
```bash
git clone https://github.com/jphaugla/redisSMSLuaA-A
```
 * Refer to the notes for redis enterprise Docker image used but don't get too bogged down as docker compose handles everything except for a few admin steps on tomcat.
   * [https://hub.docker.com/r/redislabs/redis/](https://hub.docker.com/r/redislabs/redis/)
 * Open terminal and change to the github home to create the clusters and the clustered database
```bash
./create_redis_enterprise_clusters.sh 
#  need to wait for cluster creation can use following command
docker logs rp1
#  wait for log to say "INFO __main__ MainThread: Done"
./setup_redis_enterprise_clusters.sh
# should return "Creating a new cluster... ok" from each cluster
./crdcreate.sh
```
* to stop docker use the following commands
```bash
docker stop rp1
docker stop rp2
docker rm rp1
docker rm rp2
docker network rm network1
docker network rm network2
```
To access the databases using redis-cli, leverage the two different port numbers
```bash
# this is first redis cluster
redis-cli -p 12000
# this is second redis cluster
redis-cli -p 12002
```
## To execute the java code with simple consumer
(Alternatively, this can be run through intellij)
 * Compile the code
```bash
mvn clean verify
```
 *  run the consumer   
```bash
export REDIS_URI=redis://localhost:12000
./runconsumer.sh
```
 * run the producer
```bash
# will produce to localhost 12000 and 12002.   (a little bit odd maybe)
./runproducer.sh
```
### Verify the results
```bash
redis-cli -p 12000
> keys *
```
#### Should see something similar to this![redis keys](images/keys.png)
* There is a Redis set for each message containing the hash key for each message part belonging to this message
```bash
type weather_sensor:wind:message:MSG0
```
returns "set"
```bash
smembers  weather_sensor:wind:message:MSG0
```
returns all the message parts for this message
Use one of the returned message parts in the hgetall below...
```bash
hgetall weather_sensor:wind:hash:1640798739595-2
```
return all the hash key value pairs from the message part
Also is a hash with 2 fields floatIncr and totalParts doing an HINCRBYFLOAT and HINCRBY with each stream
```bash
hgetall weather_sensor:wind:message:MSG2:float
```

## run with LUA consumer
(Alternatively, this can be run through intellij)
* Compile the code.  These will use the REDIS_URI environment variable from setEnvironment.sh
```bash
mvn clean verify
```
* run the consumer
* These scripts use REDIS_URI environment variable for connectivity
*NOTE:* this use of one hash key for all the keys is needed because LUA only can function on redis objects in the exact same hash slot. Using this type of technique can severely skew the data so is for demo purpose only.
```bash
export REDIS_URI=redis://localhost:12000
./runconsumerLUA.sh
```
*   run the producer.  
```bash
export REDIS_URI=redis://localhost:12000
./runproducerSingle.sh
```
### Verify the results
```bash
redis-cli -p 12000
> keys *
```
#### Should see something similar to this![redis LUA keys](images/keysLUA.png)
The individual redis sets and hashes are the same as above in the simple consumer example
## Additional tooling
 * scripts are provided to support typical [crdb-cli](https://docs.redis.com/latest/rs/references/crdb-cli-reference/) commands
    * The crdpurge.sh is very handy to efficiently do a "purgeall" of the database contents
 * Can run a consumer against each "region" and both will receive all the messages.  The crdb logic will handle de-duplication of the redis objects.
```bash
export REDIS_URI=redis://localhost:12000
 ./runconsumer.sh
 ```
```bash
export REDIS_URI=redis://localhost:12002
./runconsumer.sh
```
## run with open source single redis container
 * stop docker if running from above
```bash
docker stop rp1
docker stop rp2
docker rm rp1
docker rm rp2
```
 * start docker with docker compose
```bash
docker-compose up
```
```bash
# edit port number in script to 6379
export REDIS_URI=redis://localhost:12000
 ./runconsumerLUA.sh
 ```
```bash
export REDIS_URI=redis://localhost:12000
./runproducerSingle.sh
```
## Run using AWS Cloudformation
Script is aware of selected region.  So use desired method to set the region.  One easy way is the set the region in aws config file

* Set the environment for subsequent steps
```bash
source setEnvironment.sh 
```
* Create S3 bucket needed for manipulating CloudFormation scripts
```bash
./cfns3deploy.sh
```
* Add keypair for the region
```bash
./createKeyPair.sh
```
* Deploy and run script
```bash
./cfnpackage.sh
./cfndeploy.sh
```

