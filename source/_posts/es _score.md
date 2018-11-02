---
title: es _score
---

### lucene 的相关度

Lucene（或 Elasticsearch）使用 [_布尔模型（Boolean model）_](http://en.wikipedia.org/wiki/Standard_Boolean_model) 查找匹配文档， 并用一个名为 [_实用评分函数（practical scoring function）_](https://www.elastic.co/guide/cn/elasticsearch/guide/current/practical-scoring-function.html) 的公式来计算相关度。这个公式借鉴了 [_词频/逆向文档频率（term frequency/inverse document frequency）_](http://en.wikipedia.org/wiki/Tfidf) 和 [_向量空间模型（vector space model）_](http://en.wikipedia.org/wiki/Vector_space_model)，同时也加入了一些现代的新特性，如协调因子（coordination factor），字段长度归一化（field length normalization），以及词或查询语句权重提升

#### 1. 布尔模型（boolean model）

一个多词查询

```bash
{
  "query": {
    "match": {
      "text": "quick fox"
    }
  }
}
```

会在内部被重写为：

```bash
{
  "query": {
    "bool": {
      "should": [
        {"term": { "text": "quick" }},
        {"term": { "text": "fox"   }}
      ]
    }
  }
}
```

只要一个文档与查询匹配，Lucene 就会为查询计算评分，然后合并每个匹配词的评分结果，使用的评分计算公式叫做 _实用评分函数_

#### 2. TF/IDF 算法介绍

-   **TF**: 词频（term frequency） 词在文档中出现的频度
-   **IDF**：逆向文档频率（inversed document frequency)
-   field-length norm： 字段长度归一值

这三个因素是在索引时计算并存储的

#### 3. 向量空间模型(vector space model)

具体查看： https://www.elastic.co/guide/cn/elasticsearch/guide/current/scoring-theory.html

#### 4. 实用评分函数（ practical scoring function）

来计算一个 query 对一个 doc 的分数的公式，该函数会使用一个公式来计算

```bash
score(q,d)  =
            queryNorm(q)
          · coord(q,d)
          · ∑ (
                tf(t in d)
              · idf(t)2
              · t.getBoost()
              · norm(t,d)
            ) (t in q)
```

具体请看：https://www.elastic.co/guide/cn/elasticsearch/guide/current/practical-scoring-function.html

#### 5. 查询协调（query coodination）

奖励那些匹配更多字符的 doc 更多的分数

#### 6. field level boost

权重的提升会被应用到字段的每个词，而不是字段本身

```bash
{
    "match": {
        "title": {
            "query": "spark",
            "boost": 5
        }
    }
}
```

具体查看： https://www.elastic.co/guide/cn/elasticsearch/guide/current/query-time-boosting.html

Classic Lucene Similarity 已在 6.3.0 中弃用:

https://www.elastic.co/guide/en/elasticsearch/reference/6.4/index-modules-similarity.html#classic-similarity

### es 使用的 BM25 算法

\_search 加参数 explain ,是可以看到 \_score 如何计算的,比如

```bash
{
  "query": {
    "match": {
      "content": "到北京"
    }
  },
  "explain": true,
  "size": 12
}
```

截取一部分 hits:

```json
{
    "_shard": "[sentences][3]", // 索引为3的分片
    "_node": "sSoxWXsAToKidV9LMeYypg", // node_id
    "_index": "sentences",
    "_type": "_doc",
    "_id": "1",
    "_score": 3.7647452,
    "_source": {
        "content": "查到了快递，已经快到了"
    },
    "_explanation": {
        "value": 3.7647452,
        "description": "sum of:",
        "details": [
            {
                "value": 3.7647452,
                "description": "weight(content:到 in 0) [PerFieldSimilarity], result of:",
                "details": [
                    {
                        "value": 3.7647452,
                        "description": "score(doc=0,freq=2.0 = termFreq=2.0\n), product of:",
                        "details": [
                            {
                                "value": 2.5758784,
                                "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:", // idf(逆向文档频率) 的计算公式
                                "details": [
                                    {
                                        "value": 3, // “到”在该分片中，出现在了 3 个文档中
                                        "description": "docFreq",
                                        "details": []
                                    },
                                    {
                                        "value": 45, // 该分片中共有 45 个文档
                                        "description": "docCount",
                                        "details": []
                                    }
                                ]
                            },
                            {
                                "value": 1.4615384,
                                "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:", // tfNorm 计算公式
                                "details": [
                                    {
                                        "value": 2, // tf(词频) “到” 在该文档中出现了两次
                                        "description": "termFreq=2.0",
                                        "details": []
                                    },
                                    {
                                        "value": 1.2, // 默认值 1.2
                                        "description": "parameter k1",
                                        "details": []
                                    },
                                    {
                                        "value": 0.75, // 默认值 0.75
                                        "description": "parameter b",
                                        "details": []
                                    },
                                    {
                                        "value": 10.133333, // 平均文档长度
                                        "description": "avgFieldLength",
                                        "details": []
                                    },
                                    {
                                        "value": 8, // 文档长度
                                        "description": "fieldLength",
                                        "details": []
                                    }
                                ]
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
```

从上面的 hits 可以看出，\_score 计算公式：
$$
score = idf _　\frac{tf _ (k + 1) }{tf + k _ (1 - b +b _ \frac{d}{avgdl})}
$$

```bash
idf = log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5))
# docCount 某分片字段出现在了几个文档中
# docFreq  同一分片中文档总数
# k 、 b 的相关信息可以查看：
# https://www.elastic.co/guide/en/elasticsearch/reference/6.4/index-modules-similarity.html#bm25
```

-   如果不在意词在某个字段中出现的频次，而只在意是否出现过，则可以在字段映射中禁用词频统计:

    ```bash
    {
      "mappings": {
        "doc": {
          "properties": {
            "text": {
              "type":          "string",
              "index_options": "docs" // 设置成 docs 后，词频都为 1
            }
          }
        }
      }
    }
    ```

    index_options 查看： https://www.elastic.co/guide/en/elasticsearch/reference/current/index-options.html#index-options

-   字段长度的归一值对全文搜索非常重要， 许多其他字段不需要有归一值。无论文档是否包括这个字段，索引中每个文档的每个 `string` 字段都大约占用 1 个 byte 的空间。对于 `not_analyzed` 字符串字段的归一值默认是禁用的，而对于 `analyzed` 字段也可以通过修改字段映射禁用归一值：

    ```bash
    {
      "mappings": {
        "doc": {
          "properties": {
            "text": {
              "type": "string",
              "norms": { "enabled": false }
            }
          }
        }
      }
    }
    ```

    norms 查看： https://www.elastic.co/guide/en/elasticsearch/reference/current/norms.html

    禁用 norms 后 ，参数 b 为 0、 avgFieldLength、fieldLength 不会计算在内

    ```json
    {
        "value": 1.375,
        "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1) from:",
        "details": [
            {
                "value": 2,
                "description": "termFreq=2.0",
                "details": []
            },
            {
                "value": 1.2,
                "description": "parameter k1",
                "details": []
            },
            {
                "value": 0,
                "description": "parameter b (norms omitted for field)",
                "details": []
            }
        ]
    }
    ```

-   如果 词频 和 文档长度 都禁用，那么 \_score 是这样计算的
    $$
    score = idf
    $$

### 相关度分数优化方法

#### 1. boost（增加某个 term 的权重）

这是搜索时提升，索引时提升已在 5.0.0 中弃用，相关信息：
https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-boost.html

```bash
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "java spark",
              "boost": 2
            }
          }
        },
        {
          "match": {
            "content": "java spark"
          }
        }
      ]
    }
  }
}
```

或者

```bash
{
  "multi_match" : {
    "query": "some query terms here",
    "fields": [ "title^3", "content" ]
  }
}
```

boost 仅适用于 term queries，prefix, range and fuzzy queries 不会被提升

#### 2. 重构查询结构

A or B or (C or D) 结构

```bash
GET /forum/article/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "content": "java" // 权重 1/3
          }
        },
        {
          "match": {
            "content": "spark" // 权重 1/3
          }
        },
        {
          "bool": {
            "should": [
              {
                "match": {
                  "content": "solution" // 权重 1/6
                }
              },
              {
                "match": {
                  "content": "beginner" // 权重 1/6
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

#### 3. negative boost

https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-boosting-query.html

```bash
GET /forum/article/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "content": "java"
        }
      },
      "negative": {
        "match": {
          "content": "spark"
        }
      },
      "negative_boost": 0.2
    }
  }
}
```

#### 4. constant_score

https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html

如果你压根儿不需要相关度评分，直接走 constant_score 加 filter，所有的 doc 分数都是 1，没有评分的概念了

```bash
GET /forum/article/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "java"
              }
            }
          }
        },
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "spark"
              }
            }
          }
        }
      ]
    }
  }
}
```

如果某个字段更重要，可以使用 boost 增加权重

### 自定义相关度分数算法

在 Elasticsearch 中`function_score`是用于处理文档分值的 DSL，它会在查询结束后对每一个匹配的文档进行一系列的重打分操作，最后以生成的最终分数进行排序。它提供了几种默认的计算分值的函数：

#### weight：设置权重

weight 的用法最为简单，只需要设置一个数字作为权重，文档的分数就会乘以该权重。

他最大的用途应该就是和过滤器一起使用了，因为过滤器只会筛选出符合标准的文档，而不会去详细的计算每个文档的具体得分，所以只要满足条件的文档的分数都是 1，而 weight 可以将其更换为你想要的数值

#### field_value_factor：

通过文档中某个字段的值计算出一个分数

-   `field`：指定字段名
-   `factor`：对字段值进行预处理，乘以指定的数值（默认为 1）

-   `modifier`将字段值进行加工，有以下的几个选项：

    -   `none`：不处理
    -   `log`：计算对数
    -   `log1p`：先将字段值 +1，再计算对数
    -   `log2p`：先将字段值 +2，再计算对数
    -   `ln`：计算自然对数
    -   `ln1p`：先将字段值 +1，再计算自然对数
    -   `ln2p`：先将字段值 +2，再计算自然对数
    -   `square`：计算平方
    -   `sqrt`：计算平方根
    -   `reciprocal`：计算倒数

    举一个简单的例子，假设有一个商品索引，搜索时希望在相关度排序的基础上，销量（`sales`）更高的商品能排在靠前的位置，那么这条查询 DSL 可以是这样的：

    ```bash
    {
      "query": {
        "function_score": {
          "query": {
            "match": {
              "title": "雨伞"
            }
          },
          "field_value_factor": {
            "field": "sales",
            "modifier": "log1p",
            "factor": 0.1
          },
          "boost_mode": "sum"
        }
      }
    }
    ```

    这条查询会将标题中带有雨伞的商品检索出来，然后对这些文档计算一个与库存相关的分数，并与之前相关度的分数相加，对应的公式为：

    ```bash
    _score = _score + log (1 + 0.1 * sales)
    ```

#### random_score

这个函数的使用相当简单，只需要调用一下就可以返回一个 0 到 1 的分数。

它有一个非常有用的特性是可以通过`seed`属性设置一个随机种子，该函数保证在随机种子相同时返回值也相同，这点使得它可以轻松地实现对于用户的个性化推荐

#### 衰减函数：同样以某个字段的值为标准，距离某个值越近得分越高

衰减函数（Decay Function）提供了一个更为复杂的公式，它描述了这样一种情况：对于一个字段，它有一个理想的值，而字段实际的值越偏离这个理想值（无论是增大还是减小），就越不符合期望。这个函数可以很好的应用于数值、日期和地理位置类型，由以下属性组成：

-   原点（`origin`）：该字段最理想的值，这个值可以得到满分（1.0）
-   偏移量（`offset`）：与原点相差在偏移量之内的值也可以得到满分
-   衰减规模（`scale`）：当值超出了原点到偏移量这段范围，它所得的分数就开始进行衰减了，衰减规模决定了这个分数衰减速度的快慢
-   衰减值（`decay`）：该字段可以被接受的值（默认为 0.5），相当于一个分界点，具体的效果与衰减的模式有关

衰减函数还可以指定三种不同的模式：线性函数（linear）、以 e 为底的指数函数（Exp）和高斯函数（gauss），它们拥有不同的衰减曲线：

![](https://www.scienjus.com/uploads/2016/04/decay-function.png)

应用场景：租房

-   它的理想位置是公司附近
-   如果离公司在 5km 以内，是我们可以接受的范围，在这个范围内我们不去考虑距离，而是更偏向于其他信息
-   当距离超过 5km 时，我们对这套房的评价就越来越低了，直到超出了某个范围就再也不会考虑了

```bash
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "公寓"
        }
      },
      "gauss": {
        "location": {
          "origin": { "lat": 40, "lon": 116 },
          "offset": "5km",
          "scale": "10km"
           }
         },
         "boost_mode": "sum"
    }
  }
}
```

我们希望租房的位置在`40, 116`坐标附近，`5km`以内是满意的距离，`15km`以内是可以接受的距离

#### script_score`：通过自定义脚本计算分值

虽然强大的 field_value_factor 和衰减函数已经可以解决大部分问题了，但是也可以看出它们还有一定的局限性：

1. 这两种方式都只能针对一个字段计算分值

2. 这两种方式应用的字段类型有限，field_value_factor 一般只用于数字类型，而衰减函数一般只用于数字、位置和时间类型

    这时候就需要 script_score 了，它支持我们自己编写一个脚本运行，在该脚本中我们可以拿到当前文档的所有字段信息，并且只需要将计算的分数作为返回值传回 Elasticsearch 即可。

    注：使用脚本需要首先在配置文件中打开相关功能：

```
script.groovy.sandbox.enabled: true
script.inline: on
script.indexed: on
script.search: on
script.engine.groovy.inline.aggs: on
```

举一个之前做不到的例子，假如我们有一个位置索引，它有一个分类（`category`）属性，该属性是字符串枚举类型，例如商场、电影院或者餐厅等。现在由于我们有一个电影相关的活动，所以需要将电影院在搜索列表中的排位相对靠前。

之前的两种方式都无法给字符串打分，但是如果我们自己写脚本的话却很简单，使用 Groovy（Elasticsearch 的默认脚本语言）也就是一行的事：

```
return doc ['category'].value == '电影院' ? 1.1 : 1.0
```

接下来只要将这个脚本配置到查询语句中就可以了：

```
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "name": "天安门"
        }
      },
      "script_score": {
        "script": "return doc ['category'].value == '电影院' ? 1.1 : 1.0"
      }
    }
  }
}
```

或是将脚本放在`elasticsearch/config/scripts`下，然后在查询语句中引用它：

category-score.groovy：

```
return doc ['category'].value == '电影院' ? 1.1 : 1.0
```

```
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "name": "天安门"
        }
      },
      "script_score": {
        "script": {
         "file": "category-score"
        }
      }
    }
  }
}
```

在`script`中还可以通过`params`属性向脚本传值，所以为了解除耦合，上面的 DSL 还能接着改写为：

category-score.groovy：

```
return doc ['category'].value == recommend_category ? 1.1 : 1.0

{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "name": "天安门"
        }
      },
      "script_score": {
        "script": {
         "file": "category-score",
         "params": {
            "recommend_category": "电影院"
         }
        }
      }
    }
  }
}
```

这样就可以在不更改大部分查询语句和脚本的基础上动态修改推荐的位置类别了。

#### boost_mode

它还有一个属性`boost_mode`可以指定计算后的分数与原始的`_score`如何合并，有以下选项：

-   `multiply`：将结果乘以`_score`
-   `sum`：将结果加上`_score`
-   `min`：取结果与`_score`的较小值
-   `max`：取结果与`_score`的较大值
-   `replace`：使结果替换掉`_score`

#### 同时使用多个函数

上面的例子都只是调用某一个函数并与查询得到的`_score`进行合并处理，而在实际应用中肯定会出现在多个点上计算分值并合并，虽然脚本也许可以解决这个问题，但是应该没人愿意维护一个复杂的脚本吧。这时候通过多个函数将每个分值都计算出在合并才是更好的选择。在 function_score 中可以使用`functions`属性指定多个函数。它是一个数组，所以原有函数不需要发生改动。同时还可以通过`score_mode`指定各个函数分值之间的合并处理，值跟最开始提到的`boost_mode`相同。

下面举两个例子介绍一些多个函数混用的场景。

第一个例子是类似于大众点评的餐厅应用。该应用希望向用户推荐一些不错的餐馆，特征是：范围要在当前位置的 5km 以内，有停车位是最重要的，有 Wi-Fi 更好，餐厅的评分（1 分到 5 分）越高越好，并且对不同用户最好展示不同的结果以增加随机性。

那么它的查询语句应该是这样的：

```bash
{
  "query": {
    "function_score": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": $lat,
            "lon": $lng
          }
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "features": "wifi"
            }
          },
          "weight": 1
        },
        {
          "filter": {
            "term": {
              "features": "停车位"
            }
          },
          "weight": 2
        },
        {
            "field_value_factor": {
               "field": "score",
               "factor": 1.2
             }
        },
        {
          "random_score": {
            "seed": "$id"
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

注：其中所有以`$`开头的都是变量。这样一个饭馆的最高得分应该是 2 分（有停车位）+ 1 分（有 wifi）+ 6 分（评分 5 分 \* 1.2）+ 1 分（随机评分）。

另一个例子是类似于新浪微博的社交网站。现在要优化搜索功能，使其以文本相关度排序为主，但是越新的微博会排在相对靠前的位置，点赞（忽略相同计算方式的转发和评论）数较高的微博也会排在较前面。如果这篇微博购买了推广并且是创建不到 24 小时（同时满足），它的位置会非常靠前。

```bash
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "content": "$text"
        }
      },
      "functions": [
        {
          "gauss": {
            "createDate": {
              "origin": "$now",
              "scale": "6d",
              "offset": "1d"
            }
          }
        },
        {
          "field_value_factor": {
            "field": "like_count",
            "modifier": "log1p",
            "factor": 0.1
          }
        },
        {
          "script_score": {
            "script": "return doc ['is_recommend'].value && doc ['create_date'] > time ? 1.5 : 1.0",
            params: {
                "time": $time
            }
          }
        }
      ],
      "boost_mode": "multiply"
    }
  }
}
```

它的公式为：

```bash
_score * gauss (create_date, $now, "1d", "6d") * log (1 + 0.1 * like_count) * is_recommend ? 1.5 : 1.0
```

### 参考文档

-   [通过 Function Score Query 优化 Elasticsearch 搜索结果](https://www.scienjus.com/elasticsearch-function-score-query/)

-   [Elasticsearch 之（22） 自定义相关度分数算法 和 常见的相关度分数优化方法](https://blog.csdn.net/wuzhiwei549/article/details/80434603)

-   [ 相关度评分背后的理论](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scoring-theory.html)

-   [搜索引擎 ElasticSearch 的分数是怎么计算得出 (2.X & 5.X)](https://ruby-china.org/topics/node39)

-   [ElasticSearch 相关性打分机制](https://zhuanlan.zhihu.com/p/27951938)

-   [elasticsearch(九)](http://zh1cheung.com/zhi1cheung.github.io/elk/2018/09/19/elk/)

-   [干货 | Elasticsearch 通用优化建议](https://juejin.im/entry/5b7b7a23e51d453894001387)

-   [ Lucene 的实用评分函数](https://www.elastic.co/guide/cn/elasticsearch/guide/current/practical-scoring-function.html)

-   [BM25 The Next Generation of Lucene Relevance](https://opensourceconnections.com/blog/2015/10/16/bm25-the-next-generation-of-lucene-relevation/)
