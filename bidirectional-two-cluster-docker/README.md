
# Kafka Cluster Linking (Left to Right & Right to Left)

## **1. Start the Clusters**
```sh
docker compose up -d
docker compose logs -f
```

## **2. Create Topics**
```sh
docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic clicks --create --partitions 1 --replication-factor 1
docker compose exec rightKafka kafka-topics --bootstrap-server rightKafka:29092 --topic clicks --create --partitions 1 --replication-factor 1
```

## **3. Create Cluster Linking (Left to Right)**
```sh
docker compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092\
link.mode=BIDIRECTIONAL\
cluster.link.prefix=left.\
consumer.offset.sync.enable=true\
consumer.offset.sync.ms=1000\
" > /home/appuser/cl.properties'

docker compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
```

### **Optional Extra Step for Modifying Filters**
```sh
docker cp newFilters.properties rightKafka:/tmp/newFilters.properties
docker exec rightKafka kafka-configs --bootstrap-server rightKafka:29092 --alter --cluster-link bidirectional-linkAB --add-config-file /tmp/newFilters.properties
```

## **4. Create Cluster Linking (Right to Left)**
```sh
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092\
link.mode=BIDIRECTIONAL\
cluster.link.prefix=right.\
consumer.offset.sync.enable=true\
consumer.offset.sync.ms=1000\
" > /home/appuser/cl1.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl1-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl1.properties \
    --consumer-group-filters-json-file /home/appuser/cl1-offset-groups.json
```

## **5. Check for Cluster Links**
```sh
docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 --list

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --list
```

## **6. Create Mirror Topics**

```sh
#Mirror topic for data mirror from left -> right
docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic clicks \
    --mirror-topic left.clicks \
    --link bidirectional-linkAB \
    --bootstrap-server rightKafka:29092
#Mirror topic for data mirror from right -> left
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic clicks \
    --mirror-topic right.clicks \
    --link bidirectional-linkAB \
    --bootstrap-server leftKafka:19092
```

## **7. Pause Cluster Link**
```sh
kafka-configs --bootstrap-server rightKafka:29092 --entity-type cluster-links --entity-name bidirectional-linkAB --alter --add-config cluster.link.paused=true
```

## **8. Check Mirror Topics State in Cluster**
```sh
docker compose exec leftKafka kafka-mirrors --describe --bootstrap-server leftKafka:19092 --links bidirectional-linkAB

docker compose exec rightKafka kafka-mirrors --describe --bootstrap-server rightKafka:29092 --links bidirectional-linkAB
```

## **9. Consumer Offset Testing**
```sh
docker compose exec leftKafka \
    kafka-console-consumer --bootstrap-server leftKafka:19092 \
    --from-beginning \
    --group disaster_test_group \
    --property print.value=true \
    --topic __consumer_offsets \
    --formatter "kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter"
```

## **10. Disaster Recovery Testing**
## Dont run the consumer simultenously, it will stop the offset mirroring.
```sh
docker compose exec leftKafka \
kafka-console-consumer --bootstrap-server leftKafka:19092 \
    --group disaster_test_group \
    --include ".*test" \
    --property print.timestamp=true \
    --property print.offset=true \
    --property print.partition=true \
    --property print.headers=true \
    --property print.key=true \
    --property print.value=true

docker compose exec rightKafka \
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

## **11. Offset Testing**
```sh
kafka-consumer-groups --bootstrap-server leftKafka:19092 --group disaster_test_group --describe --offsets
kafka-consumer-groups --bootstrap-server rightKafka:29092 --group disaster_test_group --describe --offsets
```


## **12. Promote Mirror Topic to Writable Topic (Graceful Shutdown)**
```sh
kafka-mirrors --promote --topics left.clicks --bootstrap-server rightKafka:29092
```

## **13. Failover Mirror Topic to Writable Topic (Force Shutdown)**
```sh
kafka-mirrors --failover -topics left.clicks --bootstrap-server rightKafka:29092
```

## **14. delete the environment**
```sh
docker compose down
```

