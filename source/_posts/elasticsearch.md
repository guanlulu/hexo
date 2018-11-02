---
title: es_approch
---
    使用 elasticsearch.js 遇到一些问题，记录一下 ^_^

#### 坑1：`refresh`  参数

问题： 导入数据之后，直接 search ，hits 总是返回 [ ]

解决方法： 添加 refresh 参数，create 、bulk 都有这个参数

```bash
var elasticsearch = require('elasticsearch')
var esClient = new elasticsearch.Client({
    host: 'localhost:9200',
    log: 'error'
})
//
await esClient.create({
    index: 'index_001',
    type: 'doc',
    id: 1,
    body: {
    	content: '1'
    },
    refresh: true
})
// 批量
await esClient.bulk({ body: index_bulk, refresh: true })
```

https://www.elastic.co/guide/en/elasticsearch/reference/6.4/docs-refresh.html

#### 坑2：`Error: [version_conflict_engine_exception] [doc][1]: version conflict, document already exists`

问题： 使用 js_api create 两次 id 相同的数据时，会发生这样的问题

解决方式： 避免 id 相同

拓展： 使用 curl 并不会， 多次 put 的时候，返回结果只有 version 会变

```bash
curl -X PUT "localhost:9200/twitter/_doc/1" -H 'Content-Type: application/json' -d'
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
'
// 第一次
{
    "_index": "my_index",
    "_type": "doc",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
// 第二次
{
    "_index": "my_index",
    "_type": "doc",
    "_id": "1",
    "_version": 2,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
```







