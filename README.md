# Eclipse-Ditto-MQTT-iWatch
This example presents how to configure Ditto to be able update things and receive notification about changes via MQTT. In this example we will create a iWatch (thing) in Ditto, it will be updated via MQTT, we will update the Digital Twin in Ditto via MQTT.

# Requirements
1. Clone Ditto
2. Clone Mosquitto
3. Clone Eclipse-Ditto-MQTT-iWatch

# Start Ditto and Mosquitto

### Ditto: 
```
cd ditto-3.0.0/deployment/docker

docker compose up -d
```

### Mosquitto: 
```
docker run -it --name mosquitto --network docker_default -p 1883:1883 -v $(pwd)/mosquitto:/mosquitto/ eclipse-mosquitto
```

# Create the Policy
```
curl -X PUT 'http://localhost:8080/api/2/policies/org.Iotp2c:policy' -u 'ditto:ditto' -H 'Content-Type: application/json' -d '{
    "entries": {
        "owner": {
            "subjects": {
                "nginx:ditto": {
                    "type": "nginx basic auth user"
                }
            },
            "resources": {
                "thing:/": {
                    "grant": [
                        "READ","WRITE"
                    ],
                    "revoke": []
                },
                "policy:/": {
                    "grant": [
                        "READ","WRITE"
                    ],
                    "revoke": []
                },
                "message:/": {
                    "grant": [
                        "READ","WRITE"
                    ],
                    "revoke": []
                }
            }
        }
    }
}'
```


# Create the Thing
```
curl -X PUT 'http://localhost:8080/api/2/things/org.Iotp2c:iwatch' -u 'ditto:ditto' -H 'Content-Type: application/json' -d '{
    "policyId": "org.Iotp2c:policy",
    "attributes": {
        "name": "iwatch",
        "type": "iwatch"
    },
    "features": {
        "vital_signs": {
            "properties": {
                "heart_rate": 0
            }
        },
        "timestamp": {
            "properties": {
                "value": "1970-01-01T00:00:00.000Z"
            }
        },
        "location": {
            "properties": {
                "longitude": 0,
                "latitude": 0
            }
        }
    }
}'
```

# Create a MQTT Connection
Before we can use MQTT, we have to open a MQTT connection in Eclipse Ditto. We can do this by using DevOps Commands. In this case we need the Piggyback Commands to open a new connection.
To use these commands we have to send a `POST Request` to the URL `http://localhost:8080/devops/piggyback/connectivity?timeout=10`.

## Create the connection:
```
curl -X POST \
  'http://localhost:8080/devops/piggyback/connectivity?timeout=10' \
  -H 'Content-Type: application/json' \
  -u 'devops:foobar' \
  -d '{
    "targetActorSelection": "/system/sharding/connection",
    "headers": {
        "aggregate": false
    },
    "piggybackCommand": {
        "type": "connectivity.commands:createConnection",
        "connection": {
            "id": "mqtt-connection-iwatch",
            "connectionType": "mqtt",
            "connectionStatus": "open",
            "failoverEnabled": true,
            "uri": "tcp://ditto:ditto@172.22.0.10:1883",
            "sources": [{
                "addresses": ["org.Iotp2c:iwatch/things/twin/commands/modify"],
                "authorizationContext": ["nginx:ditto"],
                "qos": 0,
                "filters": []
            }],
            "targets": [{
                "address": "org.Iotp2c:iwatch/things/twin/events/modified",
                "topics": [
                "_/_/things/twin/events",
                "_/_/things/live/messages"
                ],
                "authorizationContext": ["nginx:ditto"],
                "qos": 0
            }]
        }
    }
}'
```

## If you need to delete the connection:
```
curl -X POST \
  'http://localhost:8080/devops/piggyback/connectivity?timeout=10' \
  -H 'Content-Type: application/json' \
  -u 'devops:foobar' \
  -d '{
    "targetActorSelection": "/system/sharding/connection",
    "headers": {
        "aggregate": false
    },
    "piggybackCommand": {
        "type": "connectivity.commands:deleteConnection",
        "connectionId": "mqtt-connection-iwatch"
    }
}'
```

# Payload mapping
Depending on your IoT-Device, you may have to map the payload that you send to Eclipse Ditto. Because IoT-Devices are often limited due to their memory, it's reasonable not to send fully qualified Ditto-Protocol messages from the IoT-Device. 
In this case, the function that simulates the data generated from an iWatch sends a dictionary with the data from iWatch.
After that we will map this payload so it is according to the Ditto-Protocol format.

Ditto-Protocol format (in the send_data_iwatch.py):
```
    ditto_data = {
        "topic": "org.Iotp2c/iwatch/things/twin/commands/modify",
        "path": "/",
        "value": {
            "thingId": "org.Iotp2c:iwatch",
            "policyId": "org.Iotp2c:policy",
            "attributes": {
                "name": "iwatch",
                "type": "iwatch"
            },
            "features": {
                "vital_signs": {
                    "properties": {
                        "heart_rate": iwatch_data['heart_rate']
                    }
                },
                "timestamp": {
                    "properties": {
                        "value": iwatch_data['timestamp']
                    }
                },
                "location": {
                    "properties": {
                        "longitude": iwatch_data['longitude'],
                        "latitude": iwatch_data['latitude']
                    }
                }
            }
        }
    }
```


`topic`: This is the topic to which the message will be published. In this case, the topic is "org.Iotp2c/iwatch/things/twin/commands/modify", which suggests that the message is intended to modify a twin (digital representation) of an iWatch device in an IoT platform.

`path`: This is the path within the twin where the value will be updated. In this case, the path is "/", indicating that the value should be updated at the root level of the iWatch twin.

`value`: This is the data payload that will be updated in the twin.

`thingId`: This is the unique identifier of the iWatch device within the IoT platform. In this example, the thingId is "org.Iotp2c:iwatch".

`policyId`: This is the identifier of the policy that governs the access control of the iWatch device. In this example, the policyId is "org.Iotp2c:policy".

`attributes`: This is a dictionary of key-value pairs that represent metadata about the iWatch device. In this example, the attributes include the device name ("iwatch") and type ("iwatch").

`features`: This is a dictionary that represents the features of the iWatch device. Each feature contains properties that describe its current state. In this example, the features include "vital_signs", "timestamp", and "location".

- `vital_signs`: This feature describes the vital signs data of the iWatch device. In this example, the feature contains a "properties" dictionary that includes the heart rate data retrieved from the iwatch_data variable.

- `timestamp`: This feature describes the timestamp of the data retrieved from the iWatch device. In this example, the feature contains a "properties" dictionary that includes the timestamp data retrieved from the iwatch_data variable.

- `location`: This feature describes the location data of the iWatch device. In this example, the feature contains a "properties" dictionary that includes the longitude and latitude data retrieved from the iwatch_data variable.

# Test if the digital twin is being updated
To see if the twin is being updated with the data send by script we can run the following:
```
curl -u ditto:ditto -X GET 'http://localhost:8080/api/2/things/org.Iotp2c:iwatch'
```
