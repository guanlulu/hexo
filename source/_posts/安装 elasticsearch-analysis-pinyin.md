#### 安装 elasticsearch-analysis-pinyin

直接进入版本库，下载对应版本编译后的压缩文件

地址： `https://github.com/medcl/elasticsearch-analysis-pinyin/releases`

解压到 `ES_HOME` 下的 `/plugins/pinyin/` ，重新启动 es

报错：

` Could not load plugin descriptor for plugin directory [pinyin]`

原因： 

应该直接解压到目录 `/plugins` 下，删除压缩包，将解压后的文件重命名为 `pinyin`

#### 测试遇到的问题

1. 报错：` No handler for type [multi_field] declared on field [name]`

   ```bash
   curl -XPOST http://localhost:9200/medcl/folks/_mapping -d'
   {
       "folks": {
           "properties": {
               "name": {
                   "type": "multi_field",
                   "fields": {
                       "name": {
                           "type": "string",
                           "store": "no",
                           "term_vector": "with_positions_offsets",
                           "analyzer": "pinyin_analyzer",
                           "boost": 10
                       },
                       "primitive": {
                           "type": "string",
                           "store": "yes",
                           "analyzer": "keyword"
                       }
                   }
               }
           }
       }
   }'
   ```

   原因： `multi-field` 在ES 1.x中已弃用，在ES 5.x中已完全删除

   解决： 现在支持多个字段[`fields`](https://www.elastic.co/guide/en/elasticsearch/reference/5.0/multi-fields.html)

   正确的写法： 

   ```json
   {
       "folks": {
           "properties": {
               "name": {
               	"type": "text", // string 已经弃用
                   "store": "false", // boolean 值只能是 true / false
                   "term_vector": "with_positions_offsets",
                   "analyzer": "pinyin_analyzer",
                   "boost": 10,
                   "fields": {
                       "primitive": {
                           "type": "text",
                           "store": "true",
                           "analyzer": "keyword"
                       }
                   }
               }
           }
       }
   }
   ```
