---
layout: post
title: ILM Policies Elasticsearch
date: 2020-09-17
categories: linux tips
---

Basic howto on creating and understanding Elasticsearch ILM policies and index aliases

### Configuring indices and rollover

#### BEFORE

* Define index naming => tcpdump-%{YYYY.MM.dd}-01
 Logstash will rotate every day with a new index with date and ending with -01
 
 **This is not correct for ILM, it must be defined by hand on Elasticsearch cluster**
 
* Define index-pattern => tcpdump_template
* Define index aliases => tcpdump
* Define rollover policy name => tcpdump_policy

##### HOWTO

* Define ILM policy
* Create template related to ILM policy
* Create clean index on ES
* Configure Logstash
* Run logstash


#### Check current policies

`GET _ilm/policy`

`GET _alias`

`GET _cat/aliases`

#### create ILM policy

Here you define a name and rollover parameters.
This example will rotate every 200mb index size.

```
PUT _ilm/policy/tcpdump_policy   
{
  "policy": {                       
    "phases": {
      "hot": {                      
        "actions": {
          "rollover": {             
            "max_size": "200MB"
          }
        }
      }
    }
  }
}
```

#### Put template

Create a template pattern where you will link with the ILM policy.

```
PUT _template/tcpdump_template
{
  "order": 0,
  "index_patterns": [
    "tcpdump-*"
  ],
  "settings": {
      "number_of_shards": "1",
      "number_of_replicas": "0",
       "index.lifecycle.name": "tcpdump_policy",
       "index.lifecycle.rollover_alias": "tcpdump"
  },
  "mappings": {
      "properties": {
        "location": {
          "type": "geo_point"
        }
    }
  }
}
```

#### Create clean index with date-math in Elasticsearch

You need to create a blank index with date-math format through the elastic API in order to make use of dates in the index name.

##### Daily index

```
PUT /%3Ctcpdump-%7Bnow%2Fd%7D-1%3E
{
  "aliases": {
    "tcpdump": {
      "is_write_index":true
    }
  }
}
```

##### Monthly index

```
PUT /%3Ctcpdump-%7Bnow%2FM%7Byyyy.MM%7D%7D-1%3E
{
  "aliases": {
    "tcpdump": {
      "is_write_index":true
    }
  }
}
```

##### Yearly Index

```
PUT /%3Ctcpdump-%7Bnow%2FM%7Byyyy%7D%7D-1%3E
{
  "aliases": {
    "tcpdump": {
      "is_write_index":true
    }
  }
}
```


### Logstash config

```
22:11 $ cat conf.d/99-output.conf
output {
 stdout { codec => rubydebug }
 if "tcpdump" in [tags] {
  elasticsearch {
    hosts => "${HOSTS}"
    manage_template => false
    # we disable index option as it's created 
    # through ES API and not by Logstash
    # 
    # index => "tcpdump-%{+YYYY.MM.dd}-1" 
    #
    # we enable ILM and the rollover alias to be used
    ilm_enabled => true
    ilm_rollover_alias => "tcpdump"
  }
 }
}
```
### Check all

```
GET _ilm/policy/tcpdump_test
{
  "tcpdump_test" : {
    "version" : 1,
    "modified_date" : "2020-01-30T15:28:47.598Z",
    "policy" : {
      "phases" : {
        "hot" : {
          "min_age" : "0ms",
          "actions" : {
            "rollover" : {
              "max_size" : "200mb"
            }
          }
        }
      }
    }
  }
}
```

```
GET _alias/tcpdump
{
  "tcpdump-2020.01.30-1" : {
    "aliases" : {
      "tcpdump" : {
        "is_write_index" : false
      }
    }
  },
  "tcpdump-2020.01.30-000002" : {
    "aliases" : {
      "tcpdump" : {
        "is_write_index" : false
      }
    }
  },
  "tcpdump-2020.01.31-000003" : {
    "aliases" : {
      "tcpdump" : {
        "is_write_index" : true
      }
    }
  }
}

```

```
GET _cat/aliases

tcpdump tcpdump-2020.01.30-1      - - - false
tcpdump tcpdump-2020.01.30-000002 - - - false
tcpdump tcpdump-2020.01.31-000003 - - - true

```

### Modify writable alias

You can change the writable index by modifying the `is_write_index` field with `false` or `true` at any time.

Only one index from the alias group can be writable at a time.

```
POST /_aliases
{
 "actions": [
   {
     "add": {
       "index": "tcpdump-2020.01.30-1",
       "alias": "tcpdump",
       "is_write_index" : false
     }
   }
 ]
}
```

### References

[https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html]()
[https://www.elastic.co/guide/en/elasticsearch/reference/7.5/indices-rollover-index.html#_using_date_math_with_the_rollover_api
]()
[https://www.elastic.co/guide/en/elasticsearch/reference/7.5/indices-aliases.html#aliases-write-index
]()
[https://www.elastic.co/guide/en/elasticsearch/reference/7.5/date-math-index-names.html]()
tcpdump