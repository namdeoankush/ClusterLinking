
## Start the clusters

```shell
    docker compose up -d

    docker compose logs -f
``` 

Tail the logs expecting everything to start.

Two CP clusters (ZK+Broker+SR+C3) are running:

*  Left Control Center available at [http://localhost:19021](http://localhost:19021/)
*  Right Control Center available at [http://localhost:29021](http://localhost:29021/)
*  Left Schema Register available at [http://localhost:8085](http://localhost:8085/)
*  Right Schema Register available at [http://localhost:8086](http://localhost:8086/)


# Below execution is for disaster recover example or can be considered for active active cluster on different regions

```shell
# creating topics topics

docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic test --create --partitions 1 --replication-factor 1

docker compose exec rightKafka kafka-topics --bootstrap-server rightKafka:29092 --topic test --create --partitions 1 --replication-factor 1


### Create cluster linking from left to right


docker compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
" > /home/appuser/cl.properties'

docker compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
 

#Cluster link 'bidirectional-link' creation successfully completed.


### Create cluster linking from right to left

docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
cluster.link.prefix=right.
consumer.offset.sync.enable=true
" > /home/appuser/cl2.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json

# check config for the created link

docker compose exec leftKafka \
    kafka-configs --bootstrap-server leftKafka:19092 \
                  --describe \
                  --cluster-link bidirectional-link

#modifying cluster link add consumer offset sync for 1000 mili seconds
docker compose exec leftKafka \
    kafka-configs --bootstrap-server leftKafka:19092 \
                  --alter \
                  --cluster-link bidirectional-link \
                  --add-config consumer.offset.sync.ms=1000


docker compose exec rightKafka \
    kafka-configs --bootstrap-server rightKafka:29092 \
                  --alter \
                  --cluster-link bidirectional-link \
                  --add-config consumer.offset.sync.ms=1000

# check for the created cluster link

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --link bidirectional-link --list

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list

# creating mirror topics

docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic left.test \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092        
 

docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic right.test \
    --link bidirectional-link \
    --bootstrap-server leftKafka:19092        

# from here onward we were work directing on the kafka cluster bash. 
# working on left cluster now:

    docker compose exec leftKafka bash

# producing 1-1000 number on leftKafka

    for x in {1..1000}; do echo $x; sleep 1; done | kafka-console-producer --bootstrap-server leftKafka:19092 --topic test

# working on right cluster now:
 
    docker compose exec rightKafka bash

# producing 2000-3000 number on rightKafka

    for x in {2000..3000}; do echo $x; sleep 1; done | kafka-console-producer --bootstrap-server rightKafka:29092 --topic test

# disaster recovery: 
# consuming message on left cluster with a consumer group test_group
  
    kafka-console-consumer --bootstrap-server leftKafka:19092 \
        --group test_group \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true

# stop consumer on left 
# consuming message on right cluster with same consumer group 

        kafka-console-consumer --bootstrap-server rightKafka:29092 \
        --group test_group \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true

# here if you notice, when you stop consuming from left and start on right, as offsets are synced every seconds the right consumer start from the offset after the offset from left consumer.

# for another executing if you dont consumer from the right cluster and try to use the same consumer group to describe the offset in both the cluster you can see the offsets being synced in right cluster as well even thought the consumer never connected to right cluster

# below are the commands to check for offset

kafka-consumer-groups --bootstrap-server leftKafka:19092 --group test_group --describe --offsets

kafka-consumer-groups --bootstrap-server rightKafka:29092 --group test_group --describe --offsets


```


