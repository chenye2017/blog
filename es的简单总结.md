---
title: es的简单总结
date: 2020-12-17 14:07:42
tags: [es,go]
categories: elasticsearch
---

最近在做商品评论的相关内容，因为数据量比较大，而且后台需要根据内容进行评论匹配的功能，显然用mysql 的 like 不太合适了，我们用的是elasticsearch。

<!--more-->

需要注意的几个点:

1. es 每条记录都有 id , 这个id 不等同于我们结构体中的id， 需要我们手动指定，如果我们不指定，es 会默认生成一个字符充当id， 当我们用get 请求的时候拼接的id 也是 此id （如果id 重复后插入的会覆盖之前的）

```
{
        "_index": "shihuo_other_comment_dgscore_v1",
        "_type": "other_comment_dgscore",
        "_id": "25135190",
        "_score": null,
        "_source": {
          "id": "25135190",
          "channelType": 23,
          "createTime": "2020-12-14 16:38:01",
          "goodsId": "50021",
          "mixId": "o_39007587",
          "commentDisplay": "2",
          "dgScore": "0.00",
          "isPhoto": 0,
          "mixType": "o",
          "praise": "0",
          "tagIds": [],
          "innerScore": "0.00",
          "content": "好看，喜欢。"
        },
        "sort": [
          25135190
        ]
      }

// 比如我们常用的go包（github.com/olivere/elastic）
中通过 Id 去指定这条记录的id
```

2.当我们想获取es 的版本号的时候我们可以发送 get  / 请求，能获取到版本号

3.es 7 好像废弃了 type 的存在，之前我们可以理解 index 数据库，type 表

4.es 的那些搜索条件

```
sort 和 query 是同一个级别的 

sort 也可以传入多个条件
{
    "sort":[
        {
            "f1":{
                "order":"desc"
            }
        },
        {
            "f2":{
                "order":"desc"
            }
        }
    ]
}

go 代码：
res1, err := client.Search(indexName).Query(bo).Sort("innerScore", false).Sort("id", false).Do(context.TODO())
或者：
sortQuery1 := elastic.NewFieldSort("name")
sortQuery2 := elastic.NewFieldSort("age").Desc()

searchService := client.Search().
Index("students").
SortBy(sortQuery1, sortQuery2)

// 几种方法的对应 
/id   -> Get().Index()
/_search -> Search().Index()
/_mapping -> Index().Index() , 这个index方法和后面的index方法 不一样的结构体的方法
/_delete -> Delete().Index()
/_deleteByQuery()  传入query 就可以了
/_updatedByQuery() 
这个相比较之前的就是和query 级别多了个参数 script， lang 用的painless。 然后如果我们更新的级别比较频繁，建议用
source 代表更新的field 比如 ctx._source.{$k}=params.{$k};
我们传入一个map 赋值给一个params ，然后循环params用来构成source 字符串 
POST /shihuo_other_comment_dgscore/other_comment_dgscore/25117494/_update_by_query
{
    "query":{
        "bool":{
            "filter":[
                {
                    "term":{
                        "goodsId":"397"
                    }
                }
            ]
        }
    },
    "script":{
        "source":"ctx._source.commentDisplay=params.test;ctx._source.commentDisplay=params.display",
        "lang":"painless",
        "params": {
               "test": 11
               "display": "display"
          }
    }
}

// 大批量更新的时候一定要用params ， 否则会报错。



/_update -> id 更新最简单的方式就是
POST /shihuo_other_comment_dgscore/other_comment_dgscore/25117494/_update
{
  "doc": {
    "innerScore": "0"
  }
}

// bulk , bulk 我们提交的数据批量插入，但这个某个失败不会导致我们整体bulk的失败
go 里面大致流程 ：
1.构造一个 bulk client （感觉就是 bulk url）
2. 构造一个 插入的request 数组
3. bulk 多次 add 这个 request ，完事 do 请求一下
```

5.我们在构造query 参数中需要注意的几个点

```
query 第一层无可厚非
bool 第二层，我平时就没用用过非bool 的
第三层有两个 filter 和 must，怎么区分呢，filter 一般用来过滤条件，比如 id = ，terms in， term 这种完全匹配 。filter效率比较高，因为他不会根据sort 去排分，must 一般用在 内容匹配上，一般我会搭配match_parse ,感觉这样更精确。然后 must 和 filter 里面装的是数组，剩下的都是 obj， 这点一定要注意。还有个should 是同一级别的，should [] 中是 or 的关系，但是should 自身和must 还是and 关系,
exists 和 missing 匹配 类似sql 中 is_null.

go 中怎么使用呢，new boolquery， 然后把这些条件都加进去。
```

6.我们在用es 更新的时候不一定马上能刷新，比如删除，比如修改，我们可以用refresh = wait_for, 这个就相当于是同步的，默认是false 是异步的，如果用true 所有的都这样会影响性能。

[关于刷新](https://blog.csdn.net/u011228889/article/details/80855431), 这篇文章讲的挺好的，其实wait_for 也不是马上刷新，只是加了wait for 之后那个接口会在等待刷新完成之后才返回结果。

7.es 的聚合功能，我们在口碑中有使用到[这篇文章用法讲的很仔细](https://www.cnblogs.com/leeSmall/p/9215909.html)

```
// 必备的属性，不能缺少
{
  "aggs": { // 属性
    "test": { // 聚合名称 
      "terms": {  // 聚合类型 
        "field": "innerTagIds", // 字段
        "size" : 2 // 数量
        "order": {
  "_key": "asc"  // 按照key 排序
}
      }
    }
  }
}

// 聚合当中的filter 属性很重要，可以减少我们聚合的次数，比如我们口碑后台就可以两次聚合成一次，减少复杂度。

go 中 指标聚合的 https://www.tizi365.com/archives/868.html，蛮好用的

// 
做口碑后台统计的时候用到的功能，用下面这个举例
{
  "query": {
    "bool": {
        
    }
  },
  "size": 0,  // 这个地方生产模式可以改成 0， 因为其实我们不需要这部分查询出来的数据
  
  "aggs": {
    "term_no_quality": {  // 分组的名称 
      "terms": {  // 聚合的模式，其实还有 avg， max， value_count 等多种 
        "field": "shopId",
        "size": 10 , // 限制聚合的桶的个数，因为可能会比较多，我们没必要
         "collect_mode":"breadth_first" // 这个地方代表聚合的模式， 之前是广度优先，我们可以用深度优先
      }
    },
    
    
    "term_shopId": {  //另一个分组 
      "filter": {  // 我们可以加上分组的条件，比如我们口碑统计的时候，一次计算需要一个条件，另一次计算不需要这个条件，我们就可以加上这个选项，但加上这个选项之后我们就不能马上加用桶聚合，还是得加上 aggs 的key ，再继续往下写
        "term": {
          "problem": 1
        }
      }, 
      "aggs": {
        "term_level": {
          "terms": {
            "field": "shopId",
             "size": 10
          },
          "aggs": {
            "term_level_level": {
              "terms": {
                "field": "level"
               
              }
            }
          }
        }
      }
    }
  }
}
```



8.es 搜索到的内容json转换成我们需要的内容

```
if searchResult.TotalHits() > 0 {
		// 查询结果不为空，则遍历结果
		var b1 Article
		// 通过Each方法，将es结果的json结构转换成struct对象
		for _, item := range searchResult.Each(reflect.TypeOf(b1)) {
			// 转换成Article对象
			if t, ok := item.(Article); ok {
				fmt.Println(t.Title)
			}
		}
	}
之前代码中遇到totalhits > 0 ,拿到数据却是0， 主要就是这个each 方法执行失败的 err我不知道怎么抛出来，需要自己打印下，一般是结构体转换失败。
```

9.es 中经常遇到的mapping 问题 [一般都是字段名字写错了](https://www.cnblogs.com/Neeo/articles/10585035.html#%E4%B8%A5%E6%A0%BC%E6%A8%A1%E5%BC%8F%EF%BC%88dynamic%EF%BC%9Astrict%EF%BC%89), 而且 es 的mapping 设定好修改包括字段类型，就不方便修改了



10.es 中查询值不存在，试试

```
{
    "query":{
        "bool":{
            "must_not":[
                {
                    "exists":{
                        "field":"innerScore"
                    }
                }
            ]
        }
    }
}
```





11.es 的遍历 。类似redis 的遍历， https://www.cnblogs.com/WalkOnMars/p/12377313.html， 注意不要用search_type=scan， 直接用

```
GET /_search?scroll=1m  // 这个1m 代表1min， 我再次请求中间的时间间隔，快照只会保留着在 1min
{
  "sort": [
    "_doc"
  ]
}
```

search_type 这种方式早就下线了，上面的scroll 方式效率也很高，因为不排序。



12.es 想获取分片信息，直接  /shihuo_other_comment_dgscore?pretty， 能获取index 的详细信息，比如6个shard



13.桶聚合的数据其实是不准确的

原理看下这篇文章 ：https://segmentfault.com/a/1190000022025890， 为了让更加准确，用一个shard 显然不合适，所以调大聚合的数据，比如 聚合的时候，termes 里面的size 我使用的是200， 让聚合更加准确

```
这两个很重要的指标
doc_count_error_upper_bound：表示没有在这次聚合中返回，但是可能存在的潜在聚合结果。（和上面链接中的 d 元素对应下）
sum_other_doc_count：表示这次聚合中没有统计到的文档数。这个好理解，因为ES统计的时候默认只会根据count显示排名前十的分桶。如果分类（这里是目的地）比较多，自然会有文档没有被统计到。
```



14.关于es 的更新操作，好久之前在写php 的时候就发现 create 和 update 功能一样，这次特地查了下，es 其实还是分不分更新和整体更新



部分更新 ：https://www.elastic.co/guide/cn/elasticsearch/guide/current/partial-updates.html， _update 方法，如果没有查询到这个doc( 不同于update_by_query 用的id 索引)， 就更新失败。



整体更新：https://www.elastic.co/guide/cn/elasticsearch/guide/current/update-doc.html#update-doc， 这种更新是那种如果不存在就插入，其实还蛮好用的。



注意上面两种方法参数有一些不一致。



15.今天在并发更新es 的时候出现错误，归其原因就是并发时候隔离性的问题 （es 控制并发用的乐观锁，每条数据都有个version 记录 ，mysql 也有，mysql 用mvcc 的事务隔离级别完美避开了 并发问题）

比如同时读取一条数据，同时 update， 到底用哪个

https://www.jianshu.com/p/7a3652bae8a4 。 es 更新数据的时候会比对 version， 如果version 不一致，就不能更新。但其实如果version 较大，也是可以更新的，用 &version_type=external 比如

```
1. version 1 , 1 ->2
2. version 2 , 1 ->3
3. version 3,  1 ->4

虽然有3条，但其实只用执行最后一条就可以了。
```

es 如果对一个字段，重复更新，version 版本号不会改变。



一旦我们用了 &version_type=external, 我们doc 记录的version 就变成我们传入的version了，我没法控制每次version 的大小 递增（可不可以按照毫秒级别 ? 或者类似雪花算法的模式，一定要是递增模式，但这样还是可能存在覆盖问题，因为但凡带了机器的前缀 ，就会有个优先级）。 目前我最简单的方法就是重试。 这样看来，es 还不能当做数据库使用，同时修改的时候还是会存在并发问题。











