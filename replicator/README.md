## Replicator

### Prerequisits

* docker-compose


### start the stack with

```bash
# set the CP version
export CP_VERSION=7.5.3
docker-compose up -d
```

### create a topic demo and  produce some messages to our source cluster dc1

docker exec broker-dc1 kafka-topics --create --topic demo --bootstrap-server broker-dc1:29092 --replication-factor 1 --partitions 1

seq -f "demo_dc1_%g ${RANDOM}" 10 | docker container exec -i broker-dc1 kafka-console-producer --broker-list localhost:9092 --topic demo

### start replicator with 

```bash
# set the CP version
export CP_VERSION=7.5.3
docker-compose -f docker-compose_replicator.yml up -d
```

### check the data flowing to broker in dc2

via control center or via cli

**CLI**
```bash
docker container exec -i broker-dc2 kafka-console-consumer --bootstrap-server localhost:9096 --topic demo --from-beginning
```

```
demo_dc1_1 10547
demo_dc1_2 10547
demo_dc1_3 10547
demo_dc1_4 10547
demo_dc1_5 10547
demo_dc1_6 10547
demo_dc1_7 10547
demo_dc1_8 10547
demo_dc1_9 10547
demo_dc1_10 10547
```

keep the console consumer above runnong produce some new messages to DC1 and see the data flowing to DC2


seq -f "demo_dc1_%g ${RANDOM}" 100 120 | docker container exec -i broker-dc1 kafka-console-producer --broker-list localhost:9092 --topic demo


output should be 
```
demo_dc1_100 19687
demo_dc1_101 19687
demo_dc1_102 19687
demo_dc1_103 19687
demo_dc1_104 19687
demo_dc1_105 19687
demo_dc1_106 19687
demo_dc1_107 19687
demo_dc1_108 19687
demo_dc1_109 19687
demo_dc1_110 19687
demo_dc1_111 19687
```