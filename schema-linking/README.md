## Schema Linking demo

### Prerequisits

* docker-compose


### start the stack with

```bash
# set the CP version
export CP_VERSION=7.5.3
docker-compose up -d
```


### create the schemas in dc1 

```bash
curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8081/subjects/product-value/versions
curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8081/subjects/other-topic-value/versions
```
check via curl

```bash
curl -v http://localhost:8081/subjects/
```

out put should be like
```
["other-topic-value","product-value"]%
```


### create SR config file on dc1-sr

 ```
docker exec schema-registry-dc1 bash -c '\echo "schema.registry.url=http://schema-registry-dc2:8081" > /home/appuser/config.txt'
```

### create the exporter

    docker exec schema-registry-dc1 bash -c '\
    schema-exporter --create --name dc1-to-dc2-sl --subjects "product-value" \
    --config-file ~/config.txt \
    --schema.registry.url http://schema-registry-dc1:8081 \
    --context-type NONE'

output should be like
```
Successfully created exporter dc1-to-dc2-sl
```

### validate the exporter
```bash
    docker exec schema-registry-dc1 bash -c '\
    schema-exporter --list \
    --schema.registry.url http://schema-registry-dc1:8081'    
```

output should be like 
```
[dc1-to-dc2-sl]
```

### check exporter is running

```bash
docker exec schema-registry-dc1 bash -c '\schema-exporter --get-status --name dc1-to-dc2-sl --schema.registry.url http://schema-registry-dc1:8081' | jq    
```

### check both sides for same schema

DC1
```bash
curl http://localhost:8081/subjects/product-value/versions/1 | jq
```

DC2
```bash
curl http://localhost:8082/subjects/product-value/versions/1 | jq    
```
