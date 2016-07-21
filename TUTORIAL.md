
# kaae <img src="https://camo.githubusercontent.com/15f26c4f603cac9bf415c841a8a60077f6db5102/687474703a2f2f696d6775722e636f6d2f654c446f4f4b592e706e67">

## Proof-of-Concept Tutorial
This tutorial will illustrate a working example of **kaae** for alerting 

_WARNING: This guide is a work-in-progress and should not be used in production!_

### Requirements

* Elasticsearch 2.3.x
* Kibana 4.5+
* shell + curl to execute commands

### Setup
Before starting, download and install the latest dev version of the plugin:
```
git clone https://github.com/chenryn/kaae
cd kaae && npm install && npm run package
/opt/kibana/bin/kibana plugin --install kaae -u file://`pwd`/kaae-latest.tar.gz
```

### Dataset
To illustrate the logic and elements involved with *kaae* we will generate some random data and insert it to Elasticsearch.
Our sample JSON object will report a UTC ```@timestamp``` and ```mos``` value per each interval:
<pre>
{"mos":4,"@timestamp":"2016-07-17T15:56:02.890"}
</pre>

The following BASH script will produce our entries for a realistic example:

```
#!/bin/bash
INDEX=`date +"%Y.%m.%d"`
SERVER="http://127.0.0.1:9200/mos-$INDEX/mos/"

echo "Press [CTRL+C] to stop.."
while :
do
	header="Content-Type: application/json"
	timestamp=`TZ=UTC date +"%Y-%m-%dT%T.%3N"`
	mos=$(( ( RANDOM % 5 )  + 1 ))
	mystring="{\"mos\":${mos},\"@timestamp\":\"${timestamp}\"}"
	echo $mystring;
	curl -sS -i -XPOST -H "$header" -d "$mystring" "$SERVER"
	sleep 5
done
```

* Save the file as ```elasticgen.sh``` and execute it for a few minutes

### Watcher rule
To illustrate the trigger logic, we will create an alert for an aggregation against the data we just created. 

The example will use simple parameters: 
* Run each 60 seconds
* Target the daily mos-* index with query aggregation
* Trip condition when aggregations.avg.value < 3
* Email action with details

```
curl -XPUT http://127.0.0.1:9200/watcher/watch/mos -d'
{
  "trigger": {
    "schedule" : { "interval" : "60"  }
  },
  "input" : {
    "search" : {
      "request" : {
        "indices" : [ "<mos-{now/d}>", "<mos-{now/d-1d}>"  ],
        "body" : {
          "query" : {
            "filtered" : {
              "query": {
		"query_string": {
		  "query": "mos:*",
		  "analyze_wildcard": true
		}
	      },
              "filter" : { "range" : { "@timestamp" : { "from" : "now-5m"  } } }
            }
          },
           "aggs": {
		"avg": {
		  "avg": {
		    "field": "mos"
		  }
		}
	  }
        }
      }
    }
  },
  "condition" : {
    "script" : {
      "script" : "payload.aggregations.avg.value < 3"
    }
  },
  "transform" : {
    "search" : {
      "request" : {
        "indices" : [ "<mos-{now/d}>", "<mos-{now/d-1d}>"  ],
        "body" : {
          "query" : {
            "filtered" : {
              "query": {
		"query_string": {
		  "query": "mos:*",
		  "analyze_wildcard": true
		}
	      },
              "filter" : { "range" : { "@timestamp" : { "from" : "now-5m"  } } }
            }
          },
           "aggs": {
		"avg": {
		  "avg": {
		    "field": "mos"
		  }
		}
	  }
        }
      }
    }
  },
  "actions" : {
    "email_admin" : {
    "throttle_period" : "15m",
    "email" : {
      "to" : "mos@qxip.net",
      "subject" : "Low MOS Detected: {{payload.aggregations.avg.value}} ",
      "priority" : "high",
      "body" : "Low MOS Detected:\n {{payload.aggregations.avg.value}} average with {{payload.aggregations.count.value}} measurements in 5 minutes"
    }
    }
  }
}'
```

### Alarm Triggering
Kibana/Kaae will automatically fetch and schedule jobs each 10 minutes, and execute the query according to the ```trigger.schedule``` parameter, validate its results according to the provided ```condition.script``` 

### Check output
Assuming all data and scripts are correctly executed, you should start seeing output in your kibana logs.

##### Positive Match
<pre>
Jul 17 17:55:00 es2pcap kibana[44702]: KAAE Payload: { took: 21,
Jul 17 17:55:00 es2pcap kibana[44702]: timed_out: false,
Jul 17 17:55:00 es2pcap kibana[44702]: _shards: { total: 186, successful: 186, failed: 0 },
Jul 17 17:55:00 es2pcap kibana[44702]: aggregations: { avg: { value: 2.9069767441860463 } } }
Jul 17 17:55:00 es2pcap kibana[44702]: KAAE Condition: payload.aggregations.avg.value < 3
<b>Jul 17 17:55:00 es2pcap kibana[44702]: Low MOS Detected: 2.9069767441860463  Low MOS Detected:
Jul 17 17:55:00 es2pcap kibana[44702]: 2.9069767441860463 average with  measurements in 5 minutes</b>
</pre>

##### Negative Match
<pre>
Jul 17 17:54:00 es2pcap kibana[44702]: KAAE Payload: { took: 30,
Jul 17 17:54:00 es2pcap kibana[44702]: timed_out: false,
Jul 17 17:54:00 es2pcap kibana[44702]: _shards: { total: 186, successful: 186, failed: 0 },
Jul 17 17:54:00 es2pcap kibana[44702]: aggregations: { avg: { value: 3.225806451612903 } } }
Jul 17 17:54:00 es2pcap kibana[44702]: KAAE Condition: payload.aggregations.avg.value < 3
</pre>


#### To be continued ... 