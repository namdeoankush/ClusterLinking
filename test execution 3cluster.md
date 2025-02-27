This is to create cluster link from left to right and left to center
create topics

```shell
docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic test --create --partitions 1 --replication-factor 1

docker compose exec centerKafka kafka-topics --bootstrap-server centerKafka:39092 --topic test --create --partitions 1 --replication-factor 1

docker compose exec rightKafka kafka-topics --bootstrap-server rightKafka:29092 --topic test --create --partitions 1 --replication-factor 1
```

### Create cluster linking from left to right

```shell
docker compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl.properties'

docker compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-linkAC \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
``` 

#Cluster link 'bidirectional-link' creation successfully completed.


### Create cluster linking from left to center


```shell
docker compose exec centerKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl.properties'

docker compose exec centerKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec centerKafka \
    kafka-cluster-links --bootstrap-server centerKafka:39092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
``` 

#Cluster link 'bidirectional-link' creation successfully completed.


### Create cluster linking from right to left
```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
cluster.link.prefix=right.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl1.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl1-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-linkAC \
    --config-file /home/appuser/cl1.properties \
    --consumer-group-filters-json-file /home/appuser/cl1-offset-groups.json
```
# check for link


### Create cluster linking from center to left
```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=centerKafka:39092
link.mode=BIDIRECTIONAL
cluster.link.prefix=center.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl2.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json
```
# check for link



docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --list

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --list

    docker compose exec centerKafka \
    kafka-cluster-links --bootstrap-server centerKafka:39092 --list

# creating mirror topics

```shell
docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic left.test \
    --link bidirectional-linkAC \
    --bootstrap-server rightKafka:29092        
``` 

```shell
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic right.test \
    --link bidirectional-linkAC \
    --bootstrap-server leftKafka:19092        
``` 

```shell
docker compose exec centerKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic left.test \
    --link bidirectional-linkAB \
    --bootstrap-server centerKafka:39092        
``` 

```shell
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic center.test \
    --link bidirectional-linkAB \
    --bootstrap-server leftKafka:19092        
``` 
```shell
docker compose exec leftKafka bash

for x in {1..1000}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server leftKafka:19092 --topic test

docker compose exec rightKafka bash

for x in {2000..3000}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server rightKafka:29092 --topic test

docker compose exec centerKafka bash

for x in {1000..2000}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server centerKafka:39092 --topic test
````



        disaster recover: 


        kafka-console-consumer --bootstrap-server leftKafka:19092 \
        --group disaster_test_group \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true

        kafka-console-consumer --bootstrap-server rightKafka:29092 \
        --group disaster_test_group \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true

        kafka-console-consumer --bootstrap-server centerKafka:39092 \
        --group disaster_test_group \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true



for x in {2..200}; do echo $x; sleep 2; done | kafka-console-producer --bootap-server rightKafka:29092 --topic test

kafka-consumer-groups --bootstrap-server leftKafka:19092 --group test_group --describe --offsets

kafka-consumer-groups --bootstrap-server rightKafka:29092 --group right_test_group --describe --offsets


kafka-consumer-groups --bootstrap-server leftKafka:19092 --group disaster_test_group --describe --offsets

kafka-consumer-groups --bootstrap-server rightKafka:29092 --group disaster_test_group --describe --offsets

kafka-consumer-groups --bootstrap-server centerKafka:29092 --group disaster_test_group --describe --offsets




kafka-console-consumer --bootstrap-server leftKafka:19092 \
        --group left_test_group \
        --from-beginning \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true



        kafka-console-consumer --bootstrap-server rightKafka:29092 \
        --group right_test_group \
        --from-beginning \
        --include ".*test" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true




