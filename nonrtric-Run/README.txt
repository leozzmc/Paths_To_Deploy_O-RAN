################################

application_configuration.json

################################

- it must run "with" A1 SDNC Controller
- if you want to run "without" A1 SDNC Controller，please use this json
```
{
   "config": {   
      "ric": [
         {
            "name": "ric1",
            "baseUrl": "http://ric1:8085/",
            "managedElementIds": [
               "kista_1",
               "kista_2"
            ]
         },
         {
            "name": "ric2",
            "baseUrl": "http://ric2:8085/",
            "managedElementIds": [
               "kista_3",
               "kista_4"
            ]
         },
         {
            "name": "ric3",
            "baseUrl": "http://ric3:8085/",
            "managedElementIds": [
               "kista_5",
               "kista_6"
            ]
         }
      ]
   }

```
- optional dmaap config in application_configuration.json

To enable the also the optional DMAAP interface, add the following config (same level as the "ric" entry) to application_configuration.json.

```
...
"streams_publishes": {
  "dmaap_publisher": {
    "type": "message-router",
    "dmaap_info": {
      "topic_url": "http://dmaap-mr:3904/events/A1-POLICY-AGENT-WRITE"
    }
  }
},
"streams_subscribes": {
  "dmaap_subscriber": {
    "type": "message-router",
    "dmaap_info": {
      "topic_url": "http://dmaap-mr:3904/events/A1-POLICY-AGENT-READ/users/policy-agent?timeout=15000&limit=100"
    }
  }
},
...
```