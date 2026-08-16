---
title: elasticsearch notes
toc: true
lang: en
description: Learnings and notes about elasticsearch
header-includes:
    <link rel="stylesheet" href="/style.css">
---

# About logging and data streams

If you only use elasticsearch for storing logs of applications, I highly recommend using a "Data Stream" with the LogsDB index mode instead of a normal index with default settings.

A normal index, if interrupted during lifecycling or with a wrong alias configuration, can create errors or problems further down the line that data streams do not have. A data stream takes care of handling the write and read indices automatically, no need to correctly name anything. You can create a data stream called "infra" and the writes automatically go to the write index called ".ds-infra-xxx".

With a data stream you can also automatically route unindexable data to a failure stream (.fs-infra-xxx) which easily shows if documents have been rejected due to mapping problems.  
This can be used to create another alert for unindexable data with for example elasticsearch_exporter.


# Recommendations for settings

Split logs with the mindset "separation of concerns".  
You can grant read/write permission to singular data streams or indices.

Have one data stream for the applications you host (e.g. named apps) and one data stream for infrastructure and platform/cloud logs (e.g. named infra).  
This way your developers look into one section and the infrastructure team looks into a different section to easier find problems and have less mapping conflicts.  

Elasticsearch can store millions of logs per hour, but splitting it up into test/prod is still good practice.  
Have atleast 2 clusters with as many nodes as needed so that no data node is more than 80% full if running for a month already (at full load).

The time to keep logs always depends on the use case, but as an example, you can keep all logs for 1 month and special short-lived ones for 2 weeks.  
On the test environments, where many different versions are built quickly, keeping logs only for 2 weeks or even 1 week is also valid.

After running and testing, one primary shard and one replica is enough to satisfy high availability. Even during updates, if a data node dies, another node still has the data. After nearly 2 years of running 4 clusters, no data loss was ever encountered with this setup.  
I would recommend always having 1 replica shard, because if a node with the only shard gets rebooted during an update, it could looes the data forever.

The rollover policy can be configured as a one-size-fits-all with a max primary shard size of 10GB and a max-age of 1day. This way you have atleast one index everyday and at max ~20GB of size per index. (20 because one primary and one replica shard, each being 10GB)  
This can create many indices per day in production environments, but one data node can have up to 1000 shards, so it shouldn't be a problem, even with hundreds of millions of log lines per day.  
Source: [https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/size-shards)

After rolling over into the warm tier it is acceptable to skip the cold tier and just delete the index after 30 days.  
Warm tier should use best-compression, read-only and 1primary/1replica settings.


# Recommendations for filebeat

Elasticsearch is a monolith, if you add overhead via pipelines to it you have a centralized bottleneck.  
It is easier and more efficient to add logic/pipelines to filebeat instead. This distributes CPU load more efficiently away from elasticsearch, enabling it to store and handle more logs.

If, for example, you have a field that you want in 2 different types, then you should convert it in Filebeat on the endpoint. This means elasticsearch does not have to do anything but save the logs it receives.

This also enables easier management of configuration because redeploying filebeat for configuration changes is easier than elasticsearch.

Filebeat has a javascript processor, which means it can do any and all custom logic on the logs it reads.  
Calling javascript code in filebeat (which was written in Golang) takes more CPU than the default processors it provides.  
Every "script" processor should have an "if" block infront of it so it only gets called for the logs it actually needs to process, using less CPU and memory.



# Debugging

If the cluster is experiencing problems it will be visible by the health endpoint.  
Green means everything is ok. Yellow means something is requiring attention, but nothing serious. A cluster can sometimes be yellow for a short duration. Red means shit just hit the fan and requires manual fixing.

It is a good practice to monitor the cluster health and alert if it stays on non-green for 5minutes.  
This can be easily done with, for example, Grafana, prometheus and elasticsearch_exporter.

If the cluster is yellow, an easy way to check what is wrong is to go to the dev tools and:

 - Check cluster health again to verify status
 - Show nodes and see if any is missing
 - Check shard status and see if any are unassigned
 - Check random unassigned shard to see why they are unassigned

This can be done with the following dev tools commands:

```
GET _cluster/health
GET _cat/nodes?v&h=ip,ram.percent,cpu,node.role,master,name
GET _cat/shards?v&h=index,prirep,state,docs,store.node&s=state,index
GET _cluster/allocation/explain?pretty
```

If any shard is unassigned and you think it is a mistake, you can retry the operation with:

```
POST /_cluster/reroute?retry_failed
```


# Mapping problems

If multiple applications log into the same index it can create mapping problems. Example:

 - Application 1 logs json with '{"host":{"name":"host1"}}'
 - Application 2 logs json with '{"host":"host2"}'

This creates a mapping problem because the field is both an object and a text field.  
This is not supported in elastic and one of the 2 log lines will be rejected and not saved.  
The rejected log line will end up in the failure stream if you are using data streams. The "error.message" field in the failure stream will show the errornous field in question.

To fix this you can update your filebeat configuration and correctly re-map the field.  
Example with a script/javascript processor in the filebeat config:

```
- if.has_fields: ["host"]
  then:
    - script:
        lang: javascript
        source: >
          function process(event) {
            var constTemp = event.Get("host");
            if (typeof(constTemp) == "string") {
              event.Delete("host");
              event.Put("host.name", constTemp);
            }
          }
```

This, ofcourse, puts unnecessary load onto filebeat. Such mapping errors should be fixed in the application's logging logic, but that is not always a quick or easy fix.  
If you are collecting logs from e.g. AWS or akamai then you can also create new data streams for them so that their logs do not interfere or create mapping problems with your application logs.


