## Cluster Linking

### Prerequisits

* docker-compose


### start the stack with

```bash
# set the CP version
export CP_VERSION=7.5.3
docker-compose up -d
```


### create the link and so on


#### create the topic 
```bash
docker exec broker-west kafka-topics --create --topic demo --bootstrap-server broker-west:29092 --replication-factor 1 --partitions 1
```

#### produce some data to the source broker-west

- Topic demo
- random numbers

```bash
seq -f "demo-msg-west_%g ${RANDOM}" 20 | docker container exec -i broker-west bash -c "kafka-console-producer --broker-list broker-west:29092 --topic demo"
```

#### check that we have received the data in source cluster using consumer group id demo, read only 5 messages
```bash
docker container exec -i broker-west bash -c "kafka-console-consumer --bootstrap-server broker-west:29092 --topic demo --from-beginning --max-messages 5 --consumer-property group.id=demo"
```

#### copy the offset file to broker-east for further usage while creating the cluster link
```bash
docker cp consumer.offset.sync.json broker-east:/tmp/consumer.offset.sync.json
```

<details>

<summary>consumer.offset.sync.json</summary>

the file defines list of consumer groups to be migrated.
in our case we migrate all consumer groups 

```json
{
    "groupFilters": [{
        "name": "*",
        "patternType": "LITERAL",
        "filterType": "INCLUDE"
    }]
}

```
</details>



#### Create the cluster link on the destination cluster (with metadata.max.age.ms=5 seconds + consumer.offset.sync.enable=true + consumer.offset.sync.ms=3000 + consumer.offset.sync.json set to all consumer groups + consumer.group.prefix disabled)"
```bash
docker exec broker-east kafka-cluster-links --bootstrap-server broker-east:29096 --create --link demo-link --config bootstrap.servers=broker-west:29092,consumer.offset.sync.enable=true,consumer.group.prefix.enable=false --consumer-group-filters-json-file /tmp/consumer.offset.sync.json
```

#### Initialize the topic mirror for our demo topic
```bash
docker exec broker-east kafka-mirrors --create --mirror-topic demo --link demo-link --bootstrap-server broker-east:29096
```

Auto topic creation on target cluster could also be enabled with ```auto.create.mirror.topics.enable``` in conjunction with ```auto.create.mirror.topics.filters``

#### Check the replica status on the destination
```bash
docker exec broker-east kafka-replica-status --topics demo --include-linked --bootstrap-server broker-east:29096

Topic Partition Replica ClusterLink IsLeader IsObserver IsIsrEligible IsInIsr IsCaughtUp LastCaughtUpLagMs LastFetchLagMs LogStartOffset LogEndOffset LeaderEpoch
demo  0         1       -           true     false      true          true    true       0                 0              0              20           0
demo  0         1       demo-link   true     false      true          true    true       -6                -6             0              20           0
```

#### Wait 6 seconds for consumer.offset sync to happen (2 times consumer.offset.sync.ms=3000)"
sleep 6

#### check consumer group on both sides

```bash
docker exec broker-east kafka-consumer-groups --bootstrap-server broker-east:29096 --describe --group demo


Consumer group 'demo' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
demo            demo            0          5               20              15              -               -               -


docker exec broker-east kafka-consumer-groups --bootstrap-server broker-west:29092 --describe --group demo

Consumer group 'demo' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
demo            demo            0          5               20              15              -               -               -
```

#### Consume from the mirror topic on the destination cluster and verify consumer offset is working, it should start at 6
```
docker container exec -i broker-east kafka-console-consumer --bootstrap-server broker-east:29096 --topic demo --max-messages 5 --consumer-property group.id=demo


demo-msg-west_6 10547
demo-msg-west_7 10547
demo-msg-west_8 10547
demo-msg-west_9 10547
demo-msg-west_10 10547
Processed a total of 5 messages
```