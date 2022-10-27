# Shell script & Makefile

## Parsing json using shell script
### Create dynamodb table
```
{
    "TableName": "Objects-shell",
    "AttributeDefinitions": [
        {
            "AttributeName": "ObjectId",
            "AttributeType": "S"
        }
    ],
    "KeySchema": [
        {
            "AttributeName": "ObjectId",
            "KeyType": "HASH"
        }
    ],
    "ProvisionedThroughput": {
        "ReadCapacityUnits": 1,
        "WriteCapacityUnits": 1
    }
}
```

### Create GSI
```
{
    "TableName": "Objects-shell",
    "AttributeDefinitions": [
        {
            "AttributeName": "NEObject",
            "AttributeType": "S"
        }, 
        {
            "AttributeName": "EpochMilli", 
            "AttributeType": "N"
        }
    ], 
    "GlobalSecondaryIndexUpdates": [
        {
            "Create": {
                "IndexName": "list-index", 
                "KeySchema": [
                    {
                        "AttributeName": "NEObject",
                        "KeyType": "HASH"
                    }, 
                    {
                        "AttributeName": "EpochMilli",
                        "KeyType": "RANGE"
                    }
                ], 
                "Projection": {
                    "ProjectionType": "ALL"
                },
                "ProvisionedThroughput": {
                    "ReadCapacityUnits": 1,
                    "WriteCapacityUnits": 1
                }
            }
        }
    ]
}
```

### set data
```
#!/bin/sh

len=$(jq '.element_count' $1)
num=0
objects=$(jq '[.near_earth_objects[][]]' $1)

while [ $num -lt $len ]
do
	object=$(echo ${objects} | jq --argjson i $num '.[$i] | {"ObjectId": .id, "ObjectName": .name, "EpochMilli": .close_approach_data[0].epoch_date_close_approach}')

	id=$(echo $object | jq .ObjectId)
	name=$(echo $object | jq .ObjectName)
	milli=$(echo $object | jq '.EpochMilli | .')	

	aws dynamodb put-item \
		--table-name Objects-shell \
		--item '{"ObjectId": {"S": '"${id}"'}, "ObjectName": {"S": '"${name}"'}, "EpochMilli": {"N": "'"${milli}"'"}, "NEObject": {"S": "obj"}}' \
		--endpoint-url http://localhost:8000
    
	num=`expr $num + 1`
done
```

## Makefile
### run app(local)
```
gen: design/design.go
	goa gen objects-dynamo/design

.PHONY:
build: gen objects_dynamo
objects_dynamo:
	go build -o $@ ./cmd/objects_dynamo

run: build
	./objects_dynamo
```

### run app
```
.PHONY:
build-linux: gen
	GOOS=linux GOARCH=amd64 go build -o objects_dynamo_linux ./cmd/objects_dynamo

.PHONY:
image: build-linux
	docker build -t light-object-app-dynamo .

.PHONY:
app: build-linux image
	docker compose up -d app
```

### init db
```
.PHONY:
table:
	aws dynamodb create-table \
		--cli-input-json file://jsonfile/create-table.json \
		--endpoint-url http://localhost:8000
	sleep 1
	
.PHONY:
index:
	aws dynamodb update-table \
		--cli-input-json file://jsonfile/create-gsi-list.json \
		--endpoint-url http://localhost:8000
	sleep 1
	
.PHONY:
set:
	./script.sh source-data/asteroids_2022_10_p1.json
	./script.sh source-data/asteroids_2022_10_p2.json
	./script.sh source-data/asteroids_2022_10_p3.json
	./script.sh source-data/asteroids_2022_10_p4.json

.PHONY:
db: table index set
```

### clean
```
.PHONY:
clean:
	rm -f objects_dynamo
	rm -f objects_dynamo_linux
	docker rm -f object-app
	docker rmi -f light-object-app-dynamo:latest
```
