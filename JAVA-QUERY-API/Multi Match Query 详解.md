## Multi Match Query

>  国内对于Elasticsearch深入的人并不多,或者大多数大牛不屑于分享关于Elasticsearch的知识,这里讲讲 Elasticsearch中的Multi Match Query


### 关于

- 官方文档:[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)
- ApacheCN : [中文文档](http://cwiki.apachecn.org/pages/viewpage.action?pageId=4883323)


### 概念

    multi_match查询基于匹配查询且允许多字段查询构建的,就是允许我们在输入一个搜索内容的时候,支持在某个索引类型下的多个
    字段中进行搜索.

### 语法

~~~
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "this is a test",
      "fields": [ "subject", "message" ]
    }
  }
}
~~~


### 具体的例子

我们就拿程序员的需求来说,一般来说程序员在遇到某个自己不懂的问题都会Baidu,Google,比如某程序员A想深入了解Elasticsearch关于match的博客文章,那么他在输入
" elasticsearch  match query " 的时候 是希望 是标题中含有" elasticsearch  match query",还是希望正文中含有"elasticsearch  match query",或许A真正希望的结果是这样的:
如果有博客的标题中含有"elasticsearch  match query"则优先返回标题相关的文章,如果没有标题中含有"elasticsearch  match query",则A希望返回正文中含有"elasticsearch  match query"
的文章,那么问题来了,你是不是这样想的呢?

我们先来造几条数据做测试:
~~~
POST /blogs/blog/1
{
  "title":"elasticsearch match query",
  "descrption":"elasticsearch match query"
}
POST /blogs/blog/2
{
  "title":"elasticsearch match query",
  "descrption":"i am a dog"
}
POST /blogs/blog/3
{
  "title":"i am a dog",
  "descrption":"elasticsearch match query"
}
POST /blogs/blog/4
{
  "title":"i am a dog",
  "descrption":"i am a dog"
}
~~~


我们构造自己的需求,我们想搜索一些关于"elasticsearch match query"的博客文章来看:

~~~
POST /blogs/blog/_search
{
  "query": {
   "multi_match": {
     "query": "elasticsearch match query",
     "fields": ["title","descrption"]
   }
  }
}
~~~

我们查看下返回结果:

~~~
{
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1.9646256,
    "hits": [
      {
        "_index": "blogs",
        "_type": "blog",
        "_id": "2",
        "_score": 1.9646256,
        "_source": {
          "title": "elasticsearch match query",
          "descrption": "i am a dog"
        }
      },
      {
        "_index": "blogs",
        "_type": "blog",
        "_id": "1",
        "_score": 0.7594807,
        "_source": {
          "title": "elasticsearch match query",
          "descrption": "elasticsearch match query"
        }
      },
      {
        "_index": "blogs",
        "_type": "blog",
        "_id": "3",
        "_score": 0.7594807,
        "_source": {
          "title": "i am a dog",
          "descrption": "elasticsearch match query"
        }
      }
    ]
  }
}
~~~

从结果中看出,搜索结果在大体上是符合我们的预期的,越与"elasticsearch match query"相关的文章,好像都返回了,但是仔细一点就会发现,结果并不是很符合,
因为id为1的博客的"descrption"内容中也包含了"elasticsearch match query",为什么会排在id为2的文章的后面呢?

这个问题可以参考：[multi_match question in ES 5.5.0 ](https://discuss.elastic.co/t/query-multi-match-question-in-es-5-5-0/96709)

官方给出的介绍是: If you only have a handful of docs use one shard. 意思是我只在单机环境下并且只有一个分片下切分片下的文档数量太少.

真正的原因,还有待商榷......

但是问题还是需要解决,我们是可以设置不同的参数来达到想要的结果:

~~~
POST /blogs/blog/_search
{
  "query": {
   "multi_match": {
     "query": "elasticsearch match query",
     "fields": ["title","descrption"],
     "tie_breaker": 20
   }
  }
}
~~~

通过设置tie_breaker参数来控制匹配的权重缓冲值,意思是每个字段都会在得到匹配之后都会对该值进行计算.


### 参数理解

multi_match查询收到许多参数控制,下面我们来看下这些参数对查询结果有哪些影响

**type**
- best_fields
        (默认) 查找与任何字段匹配的文档，使用最佳字段中的权重。 详情参见：best_fields
- most_fields
        查找与任何字段匹配的文档，并组合每个字段的权重。详情参见：most_fields.
- cross_fields
        使用相同的分析仪处理字段，就像它们是一个大字段。 在任何字段中查找每个字词，详情参见：cross_fields.
- phrase
        对每个字段运行match_phrase查询，并合并每个字段的权重，详情参见：phrase and phrase_prefix.
- phrase_prefix
        对每个字段运行match_phrase_prefix查询，并合并每个字段的权重，详情参见：phrase and phrase_prefix
**operator**

**analyzer**
**slop**
**fuzziness**
**prefixLength**
**maxExpansions**
**minimumShouldMatch**
**fuzzyRewrite**
**useDisMax**
**tieBreaker**
**lenient**
**cutoffFrequency**
**zeroTermsQuery**

### Java API

~~~
  @Test
      public void testForClient() throws Exception {
          MultiMatchQueryBuilder multiMatchQueryBuilder = QueryBuilders.multiMatchQuery("elasticsearch match query","title", "descrption");

          multiMatchQueryBuilder.analyzer("standard");
          multiMatchQueryBuilder.cutoffFrequency(0.001f);
          multiMatchQueryBuilder.field("title",20);
          multiMatchQueryBuilder.fuzziness(Fuzziness.TWO);
          multiMatchQueryBuilder.maxExpansions(100);
          multiMatchQueryBuilder.prefixLength(10);
          multiMatchQueryBuilder.tieBreaker(20);
          multiMatchQueryBuilder.type(MultiMatchQueryBuilder.Type.BEST_FIELDS);
          multiMatchQueryBuilder.boost(20);



         SearchResponse searchResponse =  client.prepareSearch()
                  .setIndices("blogs")
                  .setTypes("blog")
                  .setQuery(multiMatchQueryBuilder)
                  .execute()
                  .actionGet();

         System.out.println(ResponseUtil.parse(searchResponse));
      }
~~~

更多关于Java API,请参考:[MultiMatchQueryDemo](https://github.com/felayman/elasticsearch-java-api/blob/master/src/test/java/org/visualchina/elasticsearch/api/demo/query/MultiMatchQueryDemo.java)

## 参考
- [Match Phrase Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html)
- [Common Terms Query](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/java-full-text-queries.html#java-query-dsl-common-terms-query)



