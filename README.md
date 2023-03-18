# Eclipse-Ditto-MQTT-iWatch
This example presents how to configure Ditto to be able update things and receive notification about changes via MQTT. In this example we will create a iWatch (thing) in Ditto, it will be updated via MQTT, we will update the Digital Twin in Ditto via MQTT.

# Requirements
1. Clone Ditto
2. Clone Mosquitto

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

