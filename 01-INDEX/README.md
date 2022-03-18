# Data Management

Define an index that satisfies a given set of requirements

## Create Index

Exercises from https://medium.com/kreuzwerker-gmbh/exercises-for-the-elastic-certified-engineer-exam-store-data-into-elasticsearch-cbce230bcc6

Create the index `hamlet-raw` with 1 primary shard and 3 replicas

```
PUT /hamlet-raw
{
  "settings": {
    "index": {
      "number_of_shards": 1,  
      "number_of_replicas": 3 
    }
  }
}
```

Add a document to `hamlet-raw`, so that the document:
- has id "1"
- has default type
- has one field named `line` with value "To be, or not to be: that is the question"

> You can add a single document with [index api](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-index_.html) *PUT /<target>/_doc/<id>*

```
# Setting the ID
PUT /hamlet-raw/_doc/1
{
  "line": "To be, or not to be: that is the question"
}

GET /hamlet-raw/_search
{
  "took" : 689,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "hamlet-raw",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "line" : "To be, or not to be: that is the question"
        }
      }
    ]
  }
}
```

Update the document with id "1" by adding a field named `line_number` with value "3.1.64"

```
POST /hamlet-raw/_update/1
{
  "script": "ctx._source.line_number = '3.1.64'"
}

GET /hamlet-raw/_search
....
    "hits" : [
      {
        "_index" : "hamlet-raw",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "line" : "To be, or not to be: that is the question",
          "line_number" : "3.1.64"
        }
      }
```

Add a new document to `hamlet-raw`, so that the document:
- has the id automatically assigned by Elasticsearch
- has default type
- has a field named `text_entry` with value "Whether tis nobler in the mind to suffer"
- has a field named `line_number` with value "3.1.66"

```
# Automatic ID
POST /hamlet-raw/_doc/
{
  "text_entry": "Whether tis nobler in the mind to suffer",
  "line_number": "3.1.66"
}

GET /hamlet-raw/_search
....
    "hits" : [
      {
        "_index" : "hamlet-raw",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "line" : "To be, or not to be: that is the question",
          "line_number" : "3.1.64"
        }
      },
      {
        "_index" : "hamlet-raw",
        "_type" : "_doc",
        "_id" : "zP-UnH8B4w_Bj9E7rgZ3",
        "_score" : 1.0,
        "_source" : {
          "text_entry" : "Whether tis nobler in the mind to suffer",
          "line_number" : "3.1.66"
        }
      }
```

Update the last document by setting the value of `line_number` to "3.1.65"

```
POST /hamlet-raw/_update/zP-UnH8B4w_Bj9E7rgZ3
{
  "script": "ctx._source.line_number = '3.1.65'"
}
```

In one request, update all documents in `hamlet-raw` by adding a new field named `speaker` with value "Hamlet"

```
POST _bulk
{ "update" : {"_id" : "1", "_index" : "hamlet-raw"} }
{ "script" : { "source": "ctx._source.speaker = 'Hamlet'", "lang": "painless"} }
{ "update" : {"_id" : "zP-UnH8B4w_Bj9E7rgZ3", "_index" : "hamlet-raw"} }
{ "script" : { "source": "ctx._source.speaker = 'Hamlet'", "lang": "painless"} }
```

Update the document with id "1" by renaming the field `line` into `text_entry`

```
POST /hamlet-raw/_update/1
{
  "script": "ctx._source.text_entry = ctx._source.remove(\"line\")"
}
```

Use the following to add documents in bulk mode (used by next questions)

```
PUT hamlet/_doc/_bulk
{"index":{"_index":"hamlet","_id":0}}
{"line_number":"1.1.1","speaker":"BERNARDO","text_entry":"Whos there?"}
{"index":{"_index":"hamlet","_id":1}}
{"line_number":"1.1.2","speaker":"FRANCISCO","text_entry":"Nay, answer me: stand, and unfold yourself."}
{"index":{"_index":"hamlet","_id":2}}
{"line_number":"1.1.3","speaker":"BERNARDO","text_entry":"Long live the king!"}
{"index":{"_index":"hamlet","_id":3}}
{"line_number":"1.2.1","speaker":"KING CLAUDIUS","text_entry":"Though yet of Hamlet our dear brothers death"}
{"index":{"_index":"hamlet","_id":4}}
{"line_number":"1.2.2","speaker":"KING CLAUDIUS","text_entry":"The memory be green, and that it us befitted"}
{"index":{"_index":"hamlet","_id":5}}
{"line_number":"1.3.1","speaker":"LAERTES","text_entry":"My necessaries are embarkd: farewell:"}
{"index":{"_index":"hamlet","_id":6}}
{"line_number":"1.3.4","speaker":"LAERTES","text_entry":"But let me hear from you."}
{"index":{"_index":"hamlet","_id":7}}
{"line_number":"1.3.5","speaker":"OPHELIA","text_entry":"Do you doubt that?"}
{"index":{"_index":"hamlet","_id":8}}
{"line_number":"1.4.1","speaker":"HAMLET","text_entry":"The air bites shrewdly; it is very cold."}
{"index":{"_index":"hamlet","_id":9}}
{"line_number":"1.4.2","speaker":"HORATIO","text_entry":"It is a nipping and an eager air."}
{"index":{"_index":"hamlet","_id":10}}
{"line_number":"1.4.3","speaker":"HAMLET","text_entry":"What hour now?"}
{"index":{"_index":"hamlet","_id":11}}
{"line_number":"1.5.2","speaker":"Ghost","text_entry":"Mark me."}
{"index":{"_index":"hamlet","_id":12}}
{"line_number":"1.5.3","speaker":"HAMLET","text_entry":"I will."}
```

Create a script named `set_is_hamlet` and save it into the cluster state. The script 
- adds a field named `is_hamlet` to each document, 
- sets the field to "true" if the document has `speaker` equals to "HAMLET", 
- sets the field to "false" otherwise

```
POST _scripts/set_is_hamlet
{
  "script": {
    "lang": "painless",
    "source": """
    if (ctx._source.speaker == 'HAMLET')
        {ctx._source.is_hamlet = true}
    else
        {ctx._source.is_hamlet = false}
    """
  }
}
```

Update all documents in `hamlet` by running the `set_is_hamlet` script

```
POST hamlet/_update_by_query
{
    "script": {
        "id": "set_is_hamlet"
    }
}
```

Remove from `hamlet` the documents that have either "KING CLAUDIUS" or "LAERTES" as the value of `speaker`

> 'OR' == should, 'AND' == must

```
POST hamlet/_delete_by_query
{
    "query": {
        "bool": {
            "should": [ 
                { "match":  { "speaker": "KING CLAUDIUS" }}, 
                { "match":  { "speaker": "LAERTES" }}
            ]
        }
    }
}
```

