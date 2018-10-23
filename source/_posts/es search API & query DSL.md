---
title: es search API & query DSL
---

### search API

#### 1. 多索引

```bash
curl -XPOST "localhost:9200/twitter,elasticsearch/_search" -H 'Content-Type: application/json' -d'
{    
	"query" : {
        "match": { 
        	"content": "把手"
         }
    }
}'
// 搜索所有可用的索引
curl -XPOST "localhost:9200/_all/_search" ... 
```

常用的参数：

* explain:  默认 false , 设为 true 时，返回结果包含如何计算命中得分的解释

* _source： 设置为 false 禁用 _source 字段检索，还可以使用 _source_include 和 _source_exclude ，检索部      分文档

* from： 从命中的索引开始返回。默认为 0

* size： 要返回的点击次数。默认为 10

  其他参数，查看： [uri search](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-uri-request.html)

#### 2. source filter

```js
POST /_search
{
    "_source": false,
    // "_source": "obj.*",  // 通配符模式
    // "_source": [ "obj1.*", "obj2.*" ],
    // "_source": {
    //  	"includes": [ "obj1.*", "obj2.*" ],
    //   	"excludes": [ "*.description" ]
    },
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```



#### 3. highlight

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match": { "content": "kimchy" }
    },
    "highlight" : {
    	"pre_tags" : ["<tag1>"],
        "post_tags" : ["</tag1>"],
        "fields" : {
            "content" : {}
        }
    }
}
'
```

更多 highlight 参数 ，查看：[Highlighting](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-request-highlighting.html#configure-tags)

#### 4. Multi Search API

```bash
curl -X POST "localhost:9200/twitter/_msearch" -H 'Content-Type: application/json' -d'
{}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
{}
{"query" : {"match_all" : {}}}
{"index" : "twitter2"}
{"query" : {"match_all" : {}}}
'
```

支持模板，查看： [Multi Search API](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-multi-search.html)

### Query DSL

ES提供了基于JSON的query DSL查询语言，有两种类型子句： 
1. Leaf query。查找特定字段的特定值。如match、term、range查询。 
2. Compound query。wrap other leaf or compound queries。以一种含有逻辑（如bool、dis_max查询）的方式组合多个查询，或改变查询行为(如constant_score查询)
Query clauses behave differently depending on whether they are used in query context or filter context.

#### 1. Query and filter context

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }}, 
        { "match": { "content": "Elasticsearch" }}  
      ],
      "filter": [ 
        { "term":  { "status": "published" }}, 
        { "range": { "publish_date": { "gte": "2015-01-01" }}} 
      ]
    }
  }
}
'
```

* `query`参数表示查询上下文

* `filter`参数表示过滤器上下文,他们会过滤掉不匹配的文件，但不会影响匹配文件的分数

* `tip`:  在查询上下文中使用查询子句，以了解应该影响匹配文档的分数的条件（即文档的匹配程度），并在过滤器上下文中使用所有其他查询子句

#### 2. Match All Query

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_all": {}
    }
}
'
```

#### 3. Full text queries

##### 3.1  [Match Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-match-query.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match" : {
            "message" : "this is a test"
        }
    }
}
'
```

* `message`是字段的名称，可以替换任何字段的名称

###### 3.1.1 boolean match

match会将搜索输入进行分析，分析结果就是得到一个布尔查询。对此可简单理解为：“this is a test”，被分析为[“this”, “test”]，然后检索就变成了对于“this”和“test”的两个词的联合查询，默认的布尔连接操作符是`or`。不过，可以通过参数来改变布尔操作

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test",
                "operator" : "and" 
            }
        }
    }
}
'
```

* 注意结构上的细小变化

##### 3.2  [Match Phrase Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-match-query-phrase.html#query-dsl-match-query-phrase)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_phrase" : {
            "message" : {
                "query" : "this is a test",
                "analyzer" : "my_analyzer"
            }
        }
    }
}
'
```

* `analyzer`  默认为字段显式映射定义或默认搜索分析器
* `match_phrase` 查询首先将查询字符串解析成一个词项列表，然后对这些词项进行搜索，但只保留那些包含 *全部* 搜索词项，且 *位置* 与搜索词项相同的文档

* 一个小例子，助于理解：

  一个被认定为和短语 `quick brown fox` 匹配的文档，必须满足以下这些要求：

  * `quick` 、 `brown` 和 `fox` 需要全部出现在域中。
  * `brown` 的位置应该比 `quick` 的位置大 `1` 。
  * `fox` 的位置应该比 `quick` 的位置大 `2` 。

  如果以上任何一个选项不成立，则该文档不能认定为匹配。

##### 3.3  [Match Phrase Prefix Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-match-query-phrase-prefix.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_phrase_prefix" : {
            "message" : "quick brown f",
            "max_expansions" : 10,
            "slop": 10
        }
    }
}
'
```

* 将查询字符串的最后一个词作为前缀使用

* `max_expansions`   控制着可以与前缀匹配的词的数量
* `slop`   让相对词序位置不那么严格
* 可以实现即时查询
* 官方文档建议： For better solutions for *search-as-you-type* see the [completion suggester](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-suggesters-completion.html) and [Index-Time Search-as-You-Type](https://www.elastic.co/guide/en/elasticsearch/guide/master/_index_time_search_as_you_type.html).

##### 3.4   [Multi Match Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-multi-match-query.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "multi_match" : {
      "query":    "Will Smith",
      "fields": [ "title", "*_name" ] 
    }
  }
}
'
```

* 可以使用通配符指定字段

* 一次查询的字段数不得超过1024个
* `multi_match`  有3种查询类型,查看： [Multi Match Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-multi-match-query.html)

##### 3.5  [Common Terms Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-common-terms-query.html)

`common`查询的应用场景在于检索里包括的词项中有stopword，但是这里的stopword由于其语境不同所以不再是通常意义的stopword，相反，必须将该词项考虑到检索和计分中去。使用`common`查询将可以完成该项工作，并且不会牺牲掉检索性能

##### 3.6  [Query String Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-query-string-query.html)

可以用一些运算符 AND OR

#### 4. Term level queries

##### 4.1  [Term Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-term-query.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term" : { "user" : "Kimchy" } 
  }
}
'
```

* `term` 查询被用于精确值匹配，这些精确值可能是数字、时间、布尔或者那些 `not_analyzed` 的字符串

##### 4.2  [Terms Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-terms-query.html)

```BASH
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "terms" : { "user" : ["kimchy", "elasticsearch"]}
    }
}
'
```

* `terms`   指定多值进行精确值匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件

##### 4.3  [Range Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-range-query.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "range" : {
            "age" : {
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}
'
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "range" : {
            "date" : {
                "gte" : "now-1d/d",
                "lt" :  "now/d"
            }
        }
    }
}
'
```

* 在针对日期类型字段使用`range`查询时，可以使用[Date Match](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math)表示方法。使用时也可以灵活运用Date Format

##### 4.4  [Exists Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-exists-query.html#query-dsl-exists-query)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "exists" : { "field" : "user" }
    }
}
'
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "user"
                }
            }
        }
    }
}
'
```

* `exists` 查询被用于查找那些指定字段中有值 (`exists`) 或无值 (`missing`) 的文档

##### 4.5  [Prefix Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-prefix-query.html#query-dsl-prefix-query)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{ "query": {
    "prefix" : { "user" : "ki" }
  }
}
'
```

* 对中文好像用处不大

#### 5. Compound queries

##### 5.1 [Bool Query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-bool-query.html)

```bash
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
'
```

* **minimum_should_match** 控制文档必须满足should子句中的子查询条件的数量
* 布尔查询支持的子查询类型共有四种，分别是：must，should，must_not和filter：
  * **must **: 文档必须匹配must查询条件
  * **should**：文档应该匹配should子句查询的一个或多个
  * **must_not**:  文档不能匹配该查询条件
  * **filter**:  过滤器，文档必须匹配该过滤条件，跟must子句的唯一区别是，filter不影响查询的score

* 布尔查询的四个子句，都可以是数组字段，因此，支持嵌套逻辑操作的查询
* 布尔查询的各个子句之间的逻辑关系是与（and），这意味着，一个文档只有同时满足所有的查询子句时，该文档才匹配查询条件，作为结果返回
* 在布尔查询中，对查询结果的过滤，建议使用过滤（filter）子句和must_not子句，这两个子句属于过滤上下文（Filter Context），经常使用filter子句，使得ElasticSearch引擎自动缓存数据，当再次搜索已经被缓存的数据时，能够提高查询性能；由于过滤上下文不影响查询的评分，而评分计算让搜索变得复杂，消耗更多CPU资源，因此，filter和must_not查询减轻搜索的工作负载

###### 5.1.1 **使用布尔查询实现复杂的分组查询**

复杂的分组查询，例如：(A and B) or (C and D) or (E and F) ，把布尔查询作为should子句的一个子查询

```bash
{
  "_source": "topics",
  "from": 0,
  "size": 100,
  "query": {
    "bool": {
      "should": [
       {
          "bool": {
            "must": [
              { "term": { "topics": 1}  },
              { "term": { "topics": 2}  }
            ]
          }
        },
        {
          "bool": {
            "must": [
              {"term": { "topics": 3 } },
              {"term": { "topics": 4}}
            ]
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```



### 参考文档：

1. [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

2. [ElasticSearch查询 第五篇：布尔查询](https://www.cnblogs.com/ljhdo/p/5040252.html)

3. [Elasticseach Query DSL](https://my.oschina.net/yumg/blog/637409)

