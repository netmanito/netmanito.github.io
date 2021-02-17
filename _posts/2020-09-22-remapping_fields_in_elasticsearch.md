---
layout: post
title: Remapping fields in Elasticsearch
date: 2020-09-22
categories: linux elk elasticsearch
---


* update the mapping fields with the new configuration
* create new index with aliases and writable false

```
PUT /%3Cmyindex-%7Bnow%2FM%7Byyyy%7D%7D-000001%3E
{
  "aliases": {
    "myindex": {
      "is_write_index":false
    }
  }
}
```

* review the mapping from the new index

```
GET myindex-2020-000001
```

* set writable to false on current writable index

```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myindex-2020-1",
        "alias": "myindex",
        "is_write_index": false
      }
    }
  ]
}
```

* set writable to true on current non writable index

```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myindex-2020-000001",
        "alias": "myindex",
        "is_write_index": true
      }
    }
  ]
}
```

* reindex old index on new writable index

```
POST /_reindex?wait_for_completion=false
{
  "source": {
    "index": "myindex-2020-1"
  },
    "dest": {
      "index": "myindex"
    }
}
```
