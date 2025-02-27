This is to create cluster link from left to right and right to left

## Start the clusters

```shell
    docker compose up -d

    docker compose logs -f
``` 
# create topics

```shell
docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic test --create --partitions 1 --replication-factor 1

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
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
``` 

# Cluster link 'bidirectional-link' creation successfully completed.

# extra step for modifying filters

```shell
docker cp newFilters.properties rightKafka:/tmp/newFilters.properties

docker exec rightKafka kafka-configs --bootstrap-server rightKafka:29092 --alter --cluster-link bidirectional-linkAB --add-config-file /tmp/newFilters.properties

```

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
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl1.properties \
    --consumer-group-filters-json-file /home/appuser/cl1-offset-groups.json
```
# check for link

```shell
docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --list

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --list
```

# creating mirror topics

```shell
docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic left.test \
    --link bidirectional-linkAB \
    --bootstrap-server rightKafka:29092        
``` 

```shell
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic test \
    --mirror-topic right.test \
    --link bidirectional-linkAB \
    --bootstrap-server leftKafka:19092        
``` 

```shell
docker compose exec leftKafka bash

for x in {1..1000}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server leftKafka:19092 --topic test

docker compose exec rightKafka bash

for x in {2000..3000}; do echo $x; sleep 2; done | kafka-console-producer --bootstrap-server rightKafka:29092 --topic test


```

 # disaster recover: 

```shell
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

```

# for x in {2..200}; do echo $x; sleep 2; done | kafka-console-producer --bootap-server rightKafka:29092 --topic test

# test for offset / can use C3 as well

```shell


kafka-consumer-groups --bootstrap-server leftKafka:19092 --group disaster_test_group --describe --offsets

kafka-consumer-groups --bootstrap-server rightKafka:29092 --group disaster_test_group --describe --offsets

kafka-consumer-groups --bootstrap-server centerKafka:29092 --group disaster_test_group --describe --offsets


```



