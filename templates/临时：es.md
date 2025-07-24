# 一、理论

## 1. 思维导图

![](templates/临时：es/Map：ElasticSearch基础.xmind)

---


## 2. Elasticsearch 概述

Elasticsearch 是 ELK 三件套中的 “E”：
- **E**lasticsearch：负责数据的存储、计算与搜索
- **L**ogstash（或 Filebeat）：负责数据抓取
- **K**ibana：负责数据的可视化与分析

> [!NOTE] 注意事项
> 1. ELK 的核心是 ElasticSearch，其他组件都可替代，但都建立在 ElasticSearch 的基础之上

---


## 3. ElasticSerach 基础语法

### 3.1. 系统命令

<span style="background:#fff88f">1. 启动 ES</span>
```
# 1. 前台启动
/mystudy/es/elasticsearch/bin/elasticsearch


# 2. 后台启动
/mystudy/es/elasticsearch/bin/elasticsearch -d
```


<span style="background:#fff88f">2.停止 ES</span>
```
ps aux | grep elasticsearch

kill -9 123456
```


<span style="background:#fff88f">3. 重置 ES（删除 ES 数据即可）</span>
```
sudo rm -rf /mystudy/es/elasticsearch/data/*
```

---


### 3.2. 集群命令

<span style="background:#fff88f">1.查看集群状态</span>
```
GET /_cluster/health
"""
---------------------------------------------------------------------------------
{
  "cluster_name": "my-es-cluster",
  "status": "green",
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  ...
}

1. statues
	1. green
		1. 一切正常，所有主分片和副本分片都正常。
	2. yellow
		1. 主分片正常，但有副本分片未分配（可能是单节点，没有地方放副本）
	3. red
		1. 有主分片未分配，可能导致数据丢失或部分数据不可用
---------------------------------------------------------------------------------
"""
```


<span style="background:#fff88f">2.查看集群内所有节点信息</span>
```
GET /_cat/nodes?v
```
![](image-20250418190141210.png)


<span style="background:#fff88f">3.查看集群详细信息（包括分片、分配、任务等，数据超多，动辄上万行）</span>
```
GET /_cluster/state?pretty
```

----


### 3.3. 用户命令

<span style="background:#fff88f">1. 重设 elastic 密码</span>
```
# 1. 随机密码（适合未记录密码情况下）
/mystudy/es/elasticsearch/bin/elasticsearch-reset-password -u elastic


# 2. 自定义密码（适合记录密码情况下）
curl -k -u elastic:<当前密码> -X POST "https://<本 ES 节点IP>:9200/_security/user/elastic/_password" -H 'Content-Type: application/json' -d '{
  "password" : "<新密码>"
}'
```

---


### 3.4. 分词器命令

<span style="background:#fff88f">1. 分析某公共分词器的分词情况</span>
```
POST /_analyze
{
    "analyzer": "standard",                                                      // 分词器的名称
    "text": "霸天长得太帅了啊，我去，爱了！！！"
}
```


<span style="background:#fff88f">2.分析某索引库自定义分词器的分词情况</span>
```
POST /<索引库名>/_analyze
{
    "analyzer": "my_analyzer",
    "text": "霸天长得太帅了啊，我去，爱了！！！"
}
```

----


### 3.5. 索引命令

#### 3.5.1. 创建索引


#### 3.5.2. 删除索引

```
DELETE /<索引库名>
```

----


#### 3.5.3. 修改索引

索引库和 Mapping 一旦创建就无法修改，只能添加**新的字段**：
```
PUT /<索引库名>/_mapping
{
    "properties": {
        "新字段名": {
            "type": "integer"
        }
    }
}
```

----


#### 3.5.4. 查询索引

<span style="background:#fff88f">1. 查看某个索引库的基本信息</span>
```
GET /<索引库名称>
```


<span style="background:#fff88f">2. 查看所有索引的分片状态</span>
```
GET /_cat/shards?v
```


<span style="background:#fff88f">3. 查看所有索引的详细信息</span>
```
GET /_cat/indices?v
```
![](image-20250426150231933.png)


<span style="background:#fff88f">4. 只显示所有索引的名称</span>
```
GET /_cat/indices?h=index
```

----


### 3.6. 文档命令

#### 3.6.1. 创建文档

```
POST /user_index/_doc/1
{
  "username": "张三",
  "email": "zhangsan@example.com",
  "profile": {
    "gender": "male",
    "age": 28
  }
}
"""
---------------------------------------------------------------------------------（模板写法）
POST /<索引库名>/_doc/<文档 ID>
{
    "字段1": <值 1>
    "字段2": <值 2>
    "字段3": {
        "子属性1": <值 3>
        "子属性2": <值 4>
    }
}
---------------------------------------------------------------------------------
"""
```

> [!NOTE] 注意事项
> 1. 如果没有指定文档 ID，ES 会自动生成一个 ID，但这显然不是我们想要的，我们一般将 table_id 作为 index_id

----


#### 3.6.2. 删除文档

```
DELETE /<索引库名>/_doc/<文档 ID>
```

----


#### 3.6.3. 修改文档

<span style="background:#fff88f">1. 全量修改</span>
```
PUT /<索引库名>/_doc/<文档 ID>
{
    "字段1": <值 1>
    "字段2": <值 2>
    "字段3": {
        "子属性1": <值 3>
        "子属性2": <值 4>
    }
}
```


<span style="background:#fff88f">2. 增量修改</span>
```
# 1. 假设假设你有如下文档
{
  "username": "张三",
  "email": "zhangsan@example.com",
  "profile": {
    "gender": "male",
    "age": 28
  }
}

# 2. 进行增量修改
POST /user_index/_update/1
{
  "script": {
    "source": """
      ctx._source.username = params.username;
      ctx._source.profile.age = params.age;
    """,
    "params": {
      "username": "李四",
      "age": 30
    }
  }
}
```

----


#### 3.6.4. 查询文档

##### 3.6.4.1. 基础查询

###### 3.6.4.1.1. 根据 ID 查询文档

```
GET/<索引库名>/_doc/<文档 ID>
```
![](image-20250422184124423.png)
1. index：
	1. 文档所在的索引库名称，这里是 `"user_index"`
2. id：
	1. 文档 ID，这里是 `"1"`，你在插入时指定的
	2. 如果没有指定，ES 会自动生成
3. version：
	1. 当前文档的版本号，每次更新文档时这个数字会加1（用于乐观锁）
4. seq_no：
	1. 序列号，表示这个操作在主分片上的顺序编号（用于内部事务一致性处理）
5. primary_term：
	1. 主分片当前的任期编号，用于实现高可用性和冲突检测
6. found：
	1. 布尔值，表示是否找到了该文档。`true` 就是找到了，`false` 就是没找到
7. source：
	1. 你当初插入的原始数据（也就是文档内容）

---


###### 3.6.4.1.2. 查询所有文档

虽然名称是“查询所有文档”，但实际上默认一次只返回大约 10 条数据（前 10 条），如果需要查询后续的更多文档，需要进行分页措施，详情可见下方：分页
```
GET /<索引库名>/_search
{
  "query": {
    "match_all": {}
  }
}
```

> [!NOTE] 注意事项
> 1. 查询所有文档一次只返回 10 条数据能理解，但是 value 最多只是 10000，这是因为 ES 最多只精确统计前 10000 条结果，还是详情可见下方：分页
```
"hits": {
  "total": {
    "value": 10000,
    "relation": "gte"
  }
}
```

---


###### 3.6.4.1.3. 全文查询（查询内容走 search_analyzer）

查询某个字段时，会使用该字段配置的查询分词器（`search_analyzer`）对输入的查询内容进行分词。

分词后，系统根据分词结果在倒排索引中检索匹配的文档，并按相关性打分，从高到低返回对应的文档 ID，最终返回文档内容。

如果该字段未配置分词器，虽然仍会走分词器流程，但由于无分词器，实际不会对查询内容进行分词，整体作为一个完整的词项进行匹配。
```
# 1. 查询单字段（推荐）
GET /<索引库名>/_search
{
  "query": {
    "match": {
      "匹配字段": "YOUR_SEARCH_TEXT"
    }
  }
}


# 2. 查询多字段
GET /<索引库名>/_search
{
  "query": {
    "multi_match": {
      "query": "YOUR_SEARCH_TEXT",
      "fields": ["匹配字段1","匹配字段2","匹配字段3","匹配字段4"]
    }
  }
}
```

> [!NOTE] 注意事项
> 1. 查询多个字段时，要确保这些字段都已建立倒排索引，即设置了 `index: true`。如果某个字段未建立索引，将无法在该字段上进行搜索
> 2. 查询字段数量越多，检索和打分过程涉及的数据也越多，系统需要分别检索每个字段的倒排索引，并综合打分，增加了计算量和响应时间，因此整体查询开销会增加，导致查询效率下降。

---


###### 3.6.4.1.4. 精确查询（查询内容不走 search_analyzer）

如果希望查询内容作为一个整体进行匹配，而不被分词处理，就需要使用精确查询。这样查询时，系统会将输入内容完整地与索引中的词项进行匹配，而不会进行分词拆分。

<span style="background:#fff88f">1.term（精确值查询）</span>
一个萝卜一个坑，输入值与倒排索引中的完整词**必须完全匹配**才能命中，比如电商网站中选择具体的品牌（海澜之家、360）等等
```
GET /<索引库名>/_search
{
  "query": {
    "term": {
      "匹配字段": {
        "value": "VALUE"
      }
    }
  }
}
```


<span style="background:#fff88f">2.range（值范围查询）</span>
用于按照**数值、日期、字符串范围**进行检索，比如电商网站中按价格区间（100-200）筛选商品
```
GET /<索引库名>/_search
{
  "query": {
    "range": {
      "匹配字段": {
        "gte": 10,                   // greater than equal，即大于等于 10（可以使用 gt，大于不等于）
        "lte": 20                    // less than equal，即小于等于 20（可以使用 lt，小于不等于）
      }
    }
  }
}
```

----


###### 3.6.4.1.5. 地理查询

<span style="background:#fff88f">1.geo_bounding_box（查矩形内的 geo_point）</span>
检索所有 `geo_point` 坐标位于指定矩形范围内的文档。
```
GET /<索引名称>/_search
{
  "query": {
    "geo_bounding_box": {
      "匹配字段": {
        "top_left": {
          "lat": 40.73,
          "lon": -74.1
        },
        "bottom_right": {
          "lat": 40.717,
          "lon": -73.99
        }
      }
    }
  }
}
```
![](source/_posts/笔记：ElasticSearch 基础/PixPin_2025-04-26_17-10-28.png)


<span style="background:#fff88f">2.geo_distance（查圆形内的 geo_point）</span>
查询所有 `geo_point` 坐标在指定中心点一定距离范围内的文档。
```
GET /<索引库名>/_search
{
  "query": {
    "geo_distance": {
      "distance": 100,                 // 单位默认是米，其他单位见补充部分
      "匹配字段": {
        "lat": 40.73,                  // 纬度
        "lon": -74.1                   // 经度
      }
    }
  }
}
```
![](source/_posts/笔记：ElasticSearch 基础/PixPin_2025-04-26_17-15-36.png)


<span style="background:#fff88f">3.geo_polygon（查任意形状范围内的 geo_point）</span>
查询位于自定义多边形范围内的 `geo_point` 文档。注意：自定义形状至少需要三个点才能构成有效的多边形
```
GET /user_index/_search
{
  "query": {
    "geo_polygon": {
      "FIELD": {
        "points": [
          { "lat": 40.73, "lon": -74.1 },   // 第一个点
          { "lat": 40.83, "lon": -75.1 },   // 第二个点
          { "lat": 40.70, "lon": -75.0 },   // 第三个点，闭合成面
        ]
      }
    }
  }
}

```


<span style="background:#fff88f">4.geo_shape（查任意形状范围内 / 外 / 相交的 geo_shape）</span>
```
GET /user_index/_search
{
  "query": {
    "geo_shape": {
      "FIELD": {
        "shape": {
          "type": "envelope",
          "coordinates": [
            [-45, 45],
            [45, -45]
          ]
        },
        "relation": "within"
      }
    }
  }
}
```

> [!NOTE] 注意事项
> 1. `shape` 属性支持以下几种类型：`coordinates`、`envelope`、`circle`、`polygon`。这些定义方式与我们在创建索引时为地理字段设置属性时是一致的。详细内容可参考：[笔记：数据类型和传参](https://blog.wangjia.xin/2025/04/12/笔记：数据类型和传参/#MySQL-ES)。
> 2. `relation` 参数用于指定查询时地理图形之间的空间关系，常见取值包括：
> 	- <font color="#00b0f0">disjoint</font>：
> 		- 文档中的图形（`geo_shape`）与查询的图形（`geo_shape`）**完全不相交**，即两者没有任何接触或重叠
> 	- <font color="#00b0f0">intersects（默认值）</font>：
> 		- 只要文档的图形与查询图形**有任意交集**，即认为匹配成功
> 	- <font color="#00b0f0">within</font>：
> 		- 文档中的图形必须完整地包含在查询图形的内部

---


###### 3.6.4.1.6. 布尔查询

布尔查询的工作原理可以类比为**多个过滤层次**，文档会依次经过这些条件的筛选。例如：在查询文档时，首先会通过 `must` 条件筛选出符合要求的文档，然后再根据 `should` 条件对这些文档进行进一步的过滤和加权
```
GET /articles/_search
{
  "query": {
    "bool": {
      "must": [
        { "term":  { "status":  "published" } },
        { "term":  { "author":  "Alice"     } }
      ],
      "must_not": [
        { "term":  { "status":   "draft"   } },
        { "term":  { "category": "private" } }
      ],
      "filter": [
        {
          "range": {
            "publish_date": {
              "gte": "2020-01-01",
              "lte": "2020-12-31"
            }
          }
        },
        {
          "range": {
            "views": {
              "gte": 100
            }
          }
        }
      ],
      "should": [
        { "match": { "tags": "elasticsearch" } },
        { "match": { "tags": "search"        } }
      ],
      "minimum_should_match": 1
    }
  }
}
“”“
---------------------------------------------------------------------------------（模板写法）
GET /<索引库名>/_search
{
  "query": {
    "bool": {
      "must": [
        {<查询语句1>}，
        {<查询语句2>}
      ],
      "must_not": [
        {<查询语句1>},
        {<查询语句2>}
      ],
      "filter": [
        {<查询语句1>},
        {<查询语句2>}
      ]，
      "should": [
        {<查询语句1>},
        {<查询语句2>}
      ],
      "minimum_should_match": 1
    }
  }
}

1. must：
	1. 必须匹配（相当于“与”关系）
2. filter：
	1. 必须匹配，但不参与得分计算
3. must_not：
	1. 必须不匹配，且不参与得分计算（相当于“非”关系）
4. should：
	1. 选择性匹配（相当于“或”关系），匹配的条件越多，得分越高
	2. 可以使用 `minimum_should_match` 指定至少匹配多少个条件，如果没有满足这个数量的条件，文档就不会被包含在结果中
5. 注意事项：
	1. 可以用一个或多个
---------------------------------------------------------------------------------
”“”
```
> [!NOTE] 注意事项
> 1. 参与打分的字段越多，查询性能可能会下降。具体如何优化，建议根据实际需求和场景进行权衡

---


###### 3.6.4.1.7. 自定义评分查询（复合查询）

![](source/_posts/笔记：ElasticSearch 基础/PixPin_2025-04-27_11-32-26_PhotoGrid.png)

---

##### 3.6.4.2. 高级玩法

###### 3.6.4.2.1. 排序

Elasticsearch 默认根据文档的打分从高到低进行排序。然而，通常我们也会根据数值类型、地理空间类型、日期和时间类型进行排序，不使用打分高低进行排序，甚至还可以将这些字段结合起来进行多字段排序。

需要注意的是，Elasticsearch 仍会对文档进行打分，即便这个分数最终并不影响排序结果，除非我们禁用打分（禁用打分可增快效率）

<span style="background:#fff88f">1. 按数值类型排序</span>
```
GET /<索引库名>/_search
{
  "query": {
    ......
  },
  "sort": [
    {
      "<排序字段>": {
        "order": "asc"
      }
    }
  ]
}
```

> [!NOTE] 注意事项
> 1. 升序：`asc`
> 2. 降序：`desc`


<span style="background:#fff88f">2. 按地理空间类型排序</span>
```
GET /locations/_search
{
  "query": {
    ......
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {                         // 从这个地理坐标开始计算
          "lat": 40.73,
          "lon": -73.93
        },
        "order": "asc",                       // 升序，离坐标近的排前面
        "unit": "km"                          // 指定举例单位
      }
    }
  ]
}
```


<span style="background:#fff88f">3. 按日期与时间类型排序</span>
```
GET /<索引库名>/_search
{
  "query": {
    ......
  },
  "sort": [
    {
      "<排序字段>": {
        "order": "asc"
      }
    }
  ]
}
```


<span style="background:#fff88f">4. 禁用打分</span>
```
# 1. constant_score + filter
GET /<索引库名>/_search
{
  "query": {
    "constant_score": {
      "filter": {
        ... // 原来的 query 条件
      }
    }
  },
  "sort": [
    {
      "<排序字段>": {
        "order": "asc"
      }
    }
  ]
}


# 2. 查询全文文档本身就不进行打分
GET /<索引库名>/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "<排序字段>": {
        "order": "asc"
      }
    }
  ]
}
```

> [!NOTE] 注意事项
> 1. constant_score 把原本 “会打分” 的查询，变成 “只判断是否匹配” 的查询，只能包裹 filter 不能包裹 query

----


###### 3.6.4.2.2. 分页

在使用 `match_all` 查询所有时，我们常说 “名称是‘查询所有文档’，但实际上默认只返回大约 **10** 条数据”。需要注意的是，不仅仅是 `match_all`，**所有**查询方法默认都只返回约 **10** 条数据。要获取更多结果，就必须采用分页策略。

常见的分页策略有多种，其中 **`from + size`** 最常用，默认查询上限是 **10000** 条（支持任意页码的 “随机翻页”，但最多只能翻到 10000 条文档），此外还有 **`search_after`**（仅支持向后翻页）和 **`scroll`** 等方式。

本节只聚焦 **`from + size`**，但我们需要先了解以下几点：
1. 倒排索引结构的局限性：
	1. 与 MySQL 不同，Elasticsearch 用倒排索引来存储数据，这种结构天生并不擅长深度分页。
	2. 举个例子，如果查询第 990 到第 1000 条数据，ES 实际上会先获取前 1000 条数据，然后截取其中的 990 到 1000 条。如果是集群模式，每个分片的数据都不一样，ES 会在每个分片上分别查出前 1000 条数据，再聚合并重新排序，从而取得最终结果。
	3. 假设百度的 Elasticsearch 集群有 5000 台服务器，每台服务器都要查询前 1000 条数据，那就是 5000 × 1000 = 500 万条中间结果。之后还要在协调节点进行一次全局排序，再从中截取你想要的那 10 条。注意，这只是查询前 1000 条时的代价；如果你要的是前 10000 条呢？那中间结果就是 5000 万条；如果你要的是前 100000 条，那就意味着协调节点需要处理 5 亿条数据。
	4. 而协调节点需要为这些庞大的中间数据再次排序、截取。我的天，这 CPU 和内存消耗你敢想吗？为了防止系统资源被拖垮，ES 对可返回的数据量做了硬限制：最多只能返回 10000 条数据，也就是 `from + size` 最大不能超过 10000（例如 `from=9990&size=10`）。即便如此，每个分片节点仍需要先对本地所有命中的数据进行排序，再从中截取最多的一万条结果返回给协调节点。而协调节点则要汇总所有分片返回的结果集——可能是上百万甚至上千万条数据，再进行全局排序和截取，最终返回你请求的那 10000 条。哪怕只是这样，系统资源的开销也已经非常庞大了。
2. 查询限制十分合理：
	1. 虽然理论上可以查询更多数据，只需要简简单单改一下配置即可，但实际上我们通常不需要查询如此大量的数据。大多数业务场景都会限制查询的数量，通常只查询几百条数据。例如，百度的搜索最多只能查到第 76 页，每页大约 10 条记录，也就是最多能查 760 条记录。你可以亲自去百度搜索，看看是否符合这个规则。

```
# 1. 模版
GET /<索引库名>/_search
{
  "query": {
  
  },
  "from": <分页开始位置>,
  "size": <期望获取的文档数>,
}


# 2. 举例
GET /<索引库名>/_search
{
  "query": {
    "match_all": {}
  },
  "from": 990,
  "size": 10,
}
```

![](image-20250427122257665.png)

----


###### 3.6.4.2.3. 高亮

当词条与查询语句通过 search_analyzer 分词得到的关键词有交集时，系统会在该词条的前后添加标签，默认使用 `<em>` 标签。
```
POST /my_index/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "content": "搜索 引擎" } },
        { "term": { "status": "published" } }
      ],
      "filter": [
        { "range": { "publish_date": { "gte": "now-1M/M" } } }
      ]
    }
  },
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fragment_size": 50,
    "number_of_fragments": 3,
    "require_field_match": false, 
    "fields": {
      "content": {},
      "status": {}
    }
  }
}
"""
---------------------------------------------------------------------------------（模板写法）
POST /<索引库名>/_search
{
    "query": {
      ......
    },
    "highlight": {
        "pre_tags": ["<em>"],                                  //  设置高亮内容的包裹标签
        "post_tags": ["</em>"],  
        "fragment_size": 150,                                   // 每个高亮片段的最大字符数，默认是 100
        "number_of_fragments": 5,                          // 一个文档中最多标注几个高亮片段
        "require_field_match": true,                         // 是否要求高亮字段必须在查询字段中，默认是 true ，即要求在查询字段中，详情见下文
        "fields": {
            "<高亮字段1>": {},
            "<高亮字段2>": {}
        }
    }
}
---------------------------------------------------------------------------------
"""
```

<span style="background:#fff88f">1. "require_field_match": true</span>
![](image-20250616102927998.png)


<span style="background:#fff88f">2."require_field_match": false</span>
![](image-20250616103159416.png)

----


###### 3.6.4.2.4. 聚合

----


###### 3.6.4.2.5. 补全

```
GET /test/_search
{
  "suggest": {
    "title_suggest": {
      "text": "s",                                                  // 表示用户输入的前缀，也就是关键字
      "completion": {
        "field": "title",                                      // 指定用来做补全的字段             
        "skip_duplicates": true,                       // 避免返回重复的建议项
        "size": 10                                             // 返回最多 10 条补全结果，默认是 5 条，打破 10 条规律    
      }
    }
  }
}
```

----


## 4. Elasticsearch 分词器

### 4.1. 常用分词器

#### 4.1.1. ES 内置分词器

##### 4.1.1.1. standard 分词器

`standard` 分词器是 ES 默认的、内置的分词器，使用 `Unicode` 文本分割规则，适合大多数英文文本，但对中文分词支持效果不好。

the quick brown fox jumps over the lazy dog -----> "the", "quick", "brown", "fox", "jumps", "over", "the", "lazy", "dog

中华人民共和国国歌 -----> "中", "华", "人", "民", "共", "和", "国", “国”, "歌"

----


#### 4.1.2. IK 分词器

##### 4.1.2.1. IK 分词器概述

ES 内置的分词器对中文支持不好，我们下载 IK 分词器，这个对中文支持好

---


##### 4.1.2.2. IK 分词器安装与使用方法

<span style="background:#fff88f">1. 下载 IK 分词器</span>
从 [IK 分词器下载地址](https://release.infinilabs.com/analysis-ik/stable/)下载 IK 分词器安装包（zip 包），注意要和 ES 版本一致
![](image-20250616163426285.png)


<span style="background:#fff88f">2. 解压缩到 ES 的 plugins 目录</span>
将安装包解压缩到 ES 安装目录下的 plugins 目录中，注意所有 ES 节点都要进行这个操作：
![](image-20250418191526611.png)


<span style="background:#fff88f">3. 重启 ES</span>
```
# 1. 查询 ES 查看进程号
ps aux | grep elasticsearch


# 2. 杀死进程号
kill -9 119946


# 3. 启动 ES
su es

cd /mystudy/es/elasticsearch

bin/elasticsearch -d
```
![](image-20250418192258101.png)


<span style="background:#fff88f">4. 使用 IK 分词器</span>
```
POST /_analyze
{
	"analyzer": "ik_smart",                                         // ik_smart 或 ik_max_word
	"text": "霸天长得太帅了啊，我去，爱了！！！"
}
```


<span style="background:#fff88f">5. 扩展字典 / 停止词字典</span>
要扩展 IK 分词器的字典 / 停止词字典，需要修改其 `config` 目录下的配置文件 **`IKAnalyzer.cfg.xml`**，在其中添加自定义扩展词典的路径，以便在分词时加载新的词条。注意所有 ES 节点都要进行这个操作：
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">ext.dic</entry>
	
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords">stopwords.dic</entry>
</properties>
```

> [!NOTE] 注意事项：词典的写法
```
传智播客
奥利给
```

-----


##### 4.1.2.3. ik_smart 分词器

粗粒度分词，尽可能少分词（分词快，但是命中率低），如：中华人民共和国国歌 -----> "中华人民共和国“， ”国歌"

----


##### 4.1.2.4. ik_max_word 分词器

细粒度分词，所有可能词都拆分出来（命中率更大，但分词量大），如：中华人民共和国国歌 -----> "中华人民共和国", "中华人民", "中华", "华人", "人民共和国", "人民“, ”共和国", "共和", "国", "国歌"

----


#### 4.1.3. 拼音分词器

##### 4.1.3.1. 拼音分词器概述

![](image-20250427201720187.png)

----


##### 4.1.3.2. 拼音分词器安装与使用方法

<span style="background:#fff88f">1. 下载拼音分词器</span>
从 [拼音分词器下载地址](https://release.infinilabs.com/analysis-pinyin/stable/)下载拼音分词器安装包（zip 包），注意要和 ES 版本一致
![](image-20250616164438024.png)


<span style="background:#fff88f">2. 解压缩到 ES 的 plugins 目录</span>
将安装包解压缩到 ES 安装目录下的 plugins 目录中，注意所有 ES 节点都要进行这个操作：
![](image-20250418191526611.png)


<span style="background:#fff88f">3. 重启 ES</span>
```
# 1. 查询 ES 查看进程号
ps aux | grep elasticsearch


# 2. 杀死进程号
kill -9 119946


# 3. 启动 ES
su es

cd /mystudy/es/elasticsearch

bin/elasticsearch -d
```
![](image-20250418192258101.png)


<span style="background:#fff88f">4. 使用拼音分词器</span>
```
POST /_analyze
{
	"analyzer": "pinyin",                                                        // 只有 pinyin
	"text": "霸天长得太帅了啊，我去，爱了！！！"
}
```

----


##### 4.1.3.3. pinyin 分词器

pinyin 分词器支持多种配置选项，实际使用中我们通常会对其进行自定义，很少采用默认配置，详见下文：自定义分词器。

其默认行为如下：“如家酒店还不错” → "ru", "jia", "jiu", "dian", "hai", "bu", "cuo", "rjjdhbc"。而通过合理配置后，其功能将更加强大，比如支持全拼、姓名拼音、首字母等多种模式。

-----


#### 4.1.4. 自定义分词器

##### 4.1.4.1. 自定义分词器概述

上文提到的拼音分词器，其默认配置存在不少问题，比如“如家酒店还不错”会被分为 "ru", "jia", "jiu", "dian", "hai", "bu", "cuo", "rjjdhbc"。你看像 "rjjdhbc" 这种整体拼音串，我们一般是用不到的；而 "ru"、"jia" 这些全拼也不太理想。

有没有那种像 "rj"、"r家" 这样的组合呢？很可惜，pinyin 分词器在默认配置下并不支持这种方式。我们只能通过自定义分词器来实现，因为只有自定义分词器时，才能自定义指定 pinyin 分词器的相关配置。

自定义分词器可以由一个或多个组件（分词器）组成，通常包括两个或三个，按照顺序依次通过 character filter、tokenizer 和 filter。
![](image-20250428112050442.png)

----


##### 4.1.4.2. 自定义分词器使用方法

我们可以在**创建索引库**时定义自己的自定义分词器，但需要注意的是，该分词器仅对当前索引库生效。
```
PUT /<索引库名称>
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": { 
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": {                           // 自定义 tokenizer filter
        "py": { 
          "type": "pinyin", 
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  }
}
```



























# 二、实操

## 1. 环境搭建

### 1.1. 对等分布式集群

#### 1.1.1. 集群概述

对等分布式集群，是 Elasticsearch 内置的一种集群搭建方式。首先我们先来了解一下对等集群，对等集群没有固定主从角色，所有节点地位平等、功能对等，彼此协作完成任务。也就是说，每个节点都具备相同能力。例如在 ES 中，每一个节点都能够存储数据分片、执行数据相关操作、文档预处理和作为协调节点等。

但是，Elasticsearch 内置的集群并不是严格意义上的对等集群，而是一种 “对等分布式集群”。虽然集群中的每个节点都具备相同功能，但为了保证元数据一致性和高效管理，集群会通过选举产生一个主节点。此主节点不仅负责上述功能，还负责维护集群状态、管理节点的增删、以及协调分片的分配等。

> [!NOTE] 注意事项
> 1. 你可能会疑问：ES 拥有四种节点角色，为什么还说功能相同？这里不必过于纠结，关键在于——每个节点都具备相同的能力与功能，只是看你用不用罢了

----


#### 1.1.2. 集群特性

##### 1.1.2.1. 读写分离

首先我们先了解 ES 的查询过程：
1. 如果执行的是 `GET/<索引库名>/_doc/<文档 ID>`，即根据 ID 查询文档， 请求会先发送到任意一个节点，该节点作为协调节点（Coordinating Node），负责本次整个查询流程的管理，它会根据查询类型获取索引元数据，定位到具体的**主分片或副本分片**（写操作只会被路由到主分片，详情见下文：数据同步），并将请求路由至相应节点。在目标分片节点上执行查询，分片返回本地查询结果（如文档 ID、评分、聚合桶等）给协调节点。协调节点负责将所有分片返回的结果进行排序、聚合和分页等全局合并，最终将合并后的结果封装成单一响应返回给客户端。
2. 如果执行的是其他查询语句，流程与 ID 查询类似，但请求会路由到所有相关的主分片和副本分片（一个分片组中（一个主分片和其副本分片）只需路由到其中一个）。每个分片根据查询条件执行本地计算，并将结果（如匹配文档、评分、聚合数据）返回给协调节点。协调节点汇总所有分片结果，进行全局排序、合并和分页处理，最后将统一响应封装返回给客户端。

Elasticsearch 各节点天然支持读写共担，ES 中真正需要专门做读写分离的场景非常少见。ES 内部拥有一套完善的负载均衡和路由机制（协调节点），因此想要人为地实现读写分离既复杂又不必要。毕竟大多数情况下，我们是通过 MySQL 直接同步数据到 ES，很少让用户直接向 ES 插入数据。

你或许会疑问：MySQL 同步到 ES，难道不应该减轻主分片节点的读压力，腾出更多资源用于写入吗？实际上，这里有两个关键点需要考虑：
1. 主分片是会迁移的，比如主分片所在服务器短暂故障，主分片会在副本分片节点上重新选举。如果人为固定主分片节点只写、副本节点只读，出现故障后角色逆转，反而会带来更多复杂操作和不稳定风险
2. ES 查询时，协调节点默认优先路由到副本分片（只要副本健康且响应良好），虽不是纯粹的读写分离，但也减轻主分片节点很多压力，我称为 “伪读写分离”，通常完全够用。

---


##### 1.1.2.2. 负载均衡

Elasticsearch 内置了一套完整的负载均衡策略，由于每个节点都具备协调（Coordinating）能力，因此无论请求落到哪个节点，处理流程都是一致的：
1. 当请求为写入操作时，协调节点会将其路由到对应的主分片节点；  
2. 当请求为查询操作时，则会路由到主分片或副本分片，其中优先选择副本分片（前提是副本健康可用），以分担主分片的压力。

----


##### 1.1.2.3. 故障转移

###### 1.1.2.3.1. 主节点故障转移

当主节点挂掉后，剩余节点会自动通过选举产生新的主节点。具体的选举过程是怎样的，使用的是 Raft 算法还是其他方法，你可以以后慢慢去了解。

> [!NOTE] 注意事项
> 1. 在停掉几个节点后，发现 Elasticsearch 和 Kibana 都无法使用，这很可能是由于 Elasticsearch 集群不可用，可能原因是无法选出主节点。
> 	1. 因为 Master 节点必须获得 超过 N / 2 的投票（不含“等于”）才能成为 Leader，N 为历史上加入过集群的节点总数，即使节点已下线，其记录仍计入 N。因此我们必须保证集群中大多数节点存活时才可完成选举，例如 3 节点必须保证 2 节点存活。
> 	2. 所以，通过启动几台拥有 Master 角色的服务器，问题应该能得到解决。
> 2. 我们通常会保证用于选举 Leader 的节点数量为奇数个，这主要是因为：在相同的容灾能力下，奇数节点能以更少的资源成本提供相同的高可用性，资源利用效率更高。
> 	1. 常见的说法是使用奇数个节点是为了防止脑裂，其实并非如此。只要我们采用多数选举机制，就能够确保不会出现多个 Leader 的问题，也就是不会脑裂。但偶数个节点确实有可能出现投票平局的情况
> 	2. 那为什么还是推荐使用奇数个节点呢？这是因为选举 Leader 必须依赖多数节点存活。如前面提到的，Elasticsearch 集群不可用的常见原因之一就是无法选出主节点。比如我们使用 5 个节点，最多允许下线 2 个，至少要有 3 个节点存活；而即便我们使用了 6 个节点，最多也只能下线 2 个，仍然要求至少 4 个节点存活。你看，我们多加了一个节点，能承受的故障数并没有变，反而要求更多节点在线，变得更 “严格” 了。也就是说，我们多花了一台机器的成本，但并没有获得更高的可用性。如果加一台机器不是为了提升性能，那又何必多加？反之，如果确实需要提升性能，为何不干脆多加两台，凑成奇数呢？
> 	3. 综上，使用偶数个节点既可能出现投票平局，也无法提升资源利用率，反而增加了对节点存活数的要求，因此并不是更优的选择
> 3. 上面所说的节点，都是具有 Master 角色的节点

----


###### 1.1.2.3.2. 主分片故障转移

当某个索引的主分片所在节点挂掉时，ES 会自动从副本中选举出一个新的主分片。具体选举过程是怎样的，使用的是 Raft 算法还是其他方法，你可以以后慢慢去了解。

----


##### 1.1.2.4. 故障恢复

---





##### 1.1.2.5. 数据同步

首先我们先了解 ES 的写入过程：当客户端发起写入请求后，请求会发送到集群中的任意一个节点，该节点会作为协调节点，负责管理本次写入的整个流程。它会根据请求类型获取目标索引的元数据，并使用 `murmur3` 算法计算出该文档应落在哪个主分片上，然后将请求路由到对应主分片所在的节点。该节点会首先将操作写入 translog（追加日志），这是一种快速且持久的写入方式，确保即使发生断电或崩溃，数据操作也不会丢失，同时也可以通过该日志进行操作恢复。与此同时，数据还会被写入该节点内存中的索引缓冲区（Segment Buffer），这部分数据尚未持久化到磁盘。当内存缓冲区被写满或达到刷新周期（默认 1 秒）时，内存中的数据会被刷新为 Lucene segment 文件并写入磁盘，此时 translog 中对应的操作会被标记为已提交并可安全丢弃，从而避免日志无限膨胀。

需要注意的是，在这里，我们的 ES 并不是直接返回响应的（ES 默认使用的半同步方式）。这一点与 MySQL 和 Redis 的单体一致性机制不同（他们使用的是异步方式，最多只能实现最终一致性）。在 ES 中，主分片在完成自身写入操作后，会将该写请求并发地发送给所有副本分片。副本分片接收到写入请求后，会执行与主分片相同的操作，并将执行结果（成功或失败）反馈给主分片。默认情况下，主分片只需要接收到任意一个副本分片的成功响应即可将操作结果通知协调节点，协调节点随后才会向客户端返回响应（我们可以通过写请求中的 `wait_for_active_shards` 参数来指定需等待多少个副本分片成功，例如设为 `all` 表示所有副本分片都必须返回成功）

这种同步机制，我称之为 “近似强一致性”。原因在于，即使某个副本分片已经写入成功，由于查询时 ES 内部会进行负载均衡（并不确定查询命中的是哪个分片），所以无法保证查询必然能命中包含最新数据的那个分片；

但如果我们将 `wait_for_active_shards` 设置为 `all`，那么就实现了真正的强一致性，因为只有在所有副本分片写入成功之后，主分片才会确认写入成功。只要有任一副本写入失败，主分片就立刻中断当前请求，向协调节点报告 “写入失败”，协调节点再把 “失败” 返回给客户端。你可能会疑问，已经刷入主分片磁盘的数据会怎样？实际上，这些数据有可能在下一轮磁盘刷新周期中被删除。这就可能导致一种现象：你刚插入的数据，起初还能查询到，但过一段时间后却查不到了。

你可能还会有疑问：如果我对强一致性没有太高要求，只要求一个副本分片返回成功，就立即返回写入成功，那其他副本分片如果没同步成功怎么办？我已经给客户端返回成功，但这些副本分片仍未完成同步。实际上，Elasticsearch 有后台恢复机制，会在后台把这次操作补发给尚未同步成功的副本分片。

> [!NOTE] 注意事项：异步和半同步
> 1. 异步：
> 	1. 写操作提交后，立即返回成功给客户端
> 2. 半同步：
> 	1. 写操作必须等待至少部分确认成功后，才返回成功给客户端

---


##### 1.1.2.6. 数据分布

ES 的数据分布，简单来说可以理解为 Redis 那种的数据分区。简单来说就是，一个索引表，如果我们在创建的时候指定了多个主分片（shards），那么当我们往这个索引表插入数据，他就会自动根据 `murmur3` 算法计算出这个数据要插入到那个主分片中，然后我我们的副本分片（replicas）就会对每个主分片进行备份，存储在不同的电脑上
![](image-20250615155650857.png)

> [!NOTE] 注意事项
> 1. 注意这些分片是会动态迁移的，包括主分片和副本分片，详情见下文：数据迁移；

----


##### 1.1.2.7. 数据迁移






---


##### 1.1.2.8. 事务支持

Elasticsearch 本身并不支持传统关系型数据库里的事务（transaction）概念，但 Elasticsearch 对单个文档的写入操作是原子的

----


#### 1.1.3. 环境搭建

##### 1.1.3.1. 架构说明

集群共有三个节点，每个节点同时具备 Master、Data、Ingest 和 Coordinating 角色。首先启动第一个节点用于初始化集群信息，之后其余两个节点通过该节点作为引导节点（中间人）加入集群。

----


##### 1.1.3.2. 节点列表

Elasticsearch 最低只需要 1 个节点就可以构成一个“集群”，这里的集群指的是一个伪集群；但从容错与高可用的角度来看，至少需要 3 个节点才算是一个真正健壮的集群环境。

这里的节点是指具备 Master 角色的节点，并且 Master 角色的节点数量必须为奇数，以避免脑裂的发生。在这种配置下，通常采用 3 个节点，每个节点同时具备四种角色。

| IP             | 主机名（不必要） | 集群内名称    | 说明                                       |
| -------------- | -------- | -------- | ---------------------------------------- |
| 192.168.136.8  | es-node1 | es-node1 | master-eligible、data、ingest、coordinating |
| 192.168.136.9  | es-node2 | es-node2 | master-eligible、data、ingest、coordinating |
| 192.168.136.10 | es-node3 | es-node3 | master-eligible、data、ingest、coordinating |

----


##### 1.1.3.3. 环境准备

###### 1.1.3.3.1. 查看 Ubuntu 版本

```
lsb_release -a
```

---


###### 1.1.3.3.2. 下载软件

从 [ES 下载地址](https://www.elastic.co/cn/downloads/past-releases#elasticsearch)下载 ES 安装包：
![](image-20250416201625864.png)

----


###### 1.1.3.3.3. 安装相关工具

```
# 1. openssl（导出 CA 公钥）
command -v openssl >/dev/null 2>&1 || sudo apt-get install -y openssl


# 2. dos2unix（转 Unix，若需要使用脚本）
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix


# 3. chrony（时间同步）
command -v chrony >/dev/null 2>&1 || sudo apt-get install -y chrony


# 4. nmap（节点互通）
command -v nmap >/dev/null 2>&1 || sudo apt-get install -y nmap
            |                                                                             |
```

> [!NOTE] 注意事项：设置代理
```
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy
```

---


###### 1.1.3.3.4. 时间同步

```
sudo systemctl enable chrony && sudo systemctl start chrony
```

----


###### 1.1.3.3.5. 创建 es 用户（软件要求）

由于 ES 不允许以 root 用户身份运行（我们仅在运行 ES 时使用 es 用户），因此我们需要创建一个名为 `es` 的用户，并使用该用户进行后续的操作和配置。
```
# 1. 删除已存在 es 用户（包括其主目录）
userdel -r es


# 2. 新增 es 用户
useradd es


# 3. 为 es 用户设置密码（wq666）
passwd es
```
> [!NOTE] 注意事项
> 1. 为用户设置密码时，两次回车键就是无密码，我们这里设置 `wq666`

---


###### 1.1.3.3.6. 开放必要端口

开放 9200、9300 TCP 端口
```
sudo ufw allow 9200/tcp && sudo ufw allow 9300/tcp
```

> [!NOTE] 注意事项
> 1. TCP 端口就是我们常说的 HTTP 端口

---


###### 1.1.3.3.7. 关闭 Swap 分区

```
# 1. 将 内容注释
vim /etc/fstab
"""
# 将此内容进行注释（/swap 开头的）
# /swap.img       none    swap    sw      0       0
"""


# 2. 立即关闭 Swap 分区
swapoff -a
```

----


###### 1.1.3.3.8. 设置主机名、主机名解析（不必须）

```
# 1. 设置主机名
# 1.1. 192.168.136.8
sudo hostnamectl set-hostname es-node1

# 1.2. 192.168.136.9
sudo hostnamectl set-hostname es-node2

# 1.3. 192.168.136.10
sudo hostnamectl set-hostname es-node3


# 2. 设置主机名互相解析
sudo vim /etc/hosts
"""
192.168.136.8 es-node1
192.168.136.9 es-node2
192.168.136.10 es-node3
"""
```

> [!NOTE] 注意事项
> 1. ES 集群发现其他节点不必须使用主机名，可以直接使用 IP 地址，但是我们必须要为第一个节点进行设置主机名，如 `sudo hostnamectl set-hostname es-node1`，因为我们的 `cluster.initial_master_nodes` 必须要用节点名称

---


##### 1.1.3.4. 安装软件

```
# 1. 创建 mystudy/es 目录
mkdir -p /mystudy/es


# 2. 将 ES 安装包上传至 /mystudy/es 目录


# 3. 进入 /mystudy/es 目录
cd /mystudy/es


# 4. 删除已存在 ES 目录
rm -rf /mystudy/es/elasticsearch


# 5. 解压
tar -zxvf  elasticsearch-8.18.0-linux-x86_64.tar.gz -C /mystudy/es


# 6. 重命名
mv elasticsearch-8.18.0 elasticsearch
```

---


##### 1.1.3.5. 检查网络互通

```
# 1. 192.168.136.8
nmap -p 9200,9300 192.168.136.9-10
"""
---------------------------------------------------------------------------------
root@user-virtual-machine:~# nmap -p 9200,9300 192.168.136.9-10
Starting Nmap 7.80 ( https://nmap.org ) at 2025-06-14 21:47 CST
Nmap scan report for 192.168.136.9
Host is up (0.00046s latency).

PORT     STATE  SERVICE
9200/tcp closed wap-wsp
9300/tcp closed vrace
MAC Address: 00:0C:29:C4:EE:0D (VMware)

Nmap scan report for 192.168.136.10
Host is up (0.00064s latency).

PORT     STATE  SERVICE
9200/tcp closed wap-wsp
9300/tcp closed vrace
MAC Address: 00:0C:29:0E:71:B9 (VMware)

Nmap done: 2 IP addresses (2 hosts up) scanned in 0.26 seconds
---------------------------------------------------------------------------------
1. Host is up
	1. 表示目标主机处于在线状态，网络连接正常（可类比于 ping 通）
2. 9200/tcp closed wap-wsp
	1. closed
		1. 表示端口处于可达状态，但当前没有任何应用在该端口监听
	2. filtered
		1. 表示探测请求被防火墙或安全组拦截，端口无法访问
"""


# 2. 192.168.136.9
nmap -p 9200,9300 192.168.136.8 192.168.136.10


# 3. 192.168.136.10
nmap -p 9200,9300 192.168.136.8-9
```

----


##### 1.1.3.6. 创建存放 ES 数据的目录

```
mkdir -p /mystudy/es/elasticsearch/data
```

---


##### 1.1.3.7. 生成 SSL 证书

###### 1.1.3.7.1. 创建存放证书的目录

```
mkdir -p /mystudy/es/elasticsearch/config/certs
```

---


###### 1.1.3.7.2. 生成 CA 证书（CA 私钥和 CA 公钥）、CA 公钥

Elasticsearch 要通过 HTTPS 提供服务，因此若要让用户正常访问 `https://192.168.136.8:9200`，就需要为其配置一份 HTTPS 证书，且该证书中必须包含 IP 地址 `192.168.136.8`。

尽管我们可以手动生成自签名证书，Elasticsearch 也提供了更便捷的工具来自动生成 HTTPS 证书。这个工具的第一步是生成一个 CA 证书。需要注意，这个 CA 证书不仅仅包含公钥，还包含私钥，和我们平常所说的“仅包含公钥”的 CA 根证书有所不同。

关于 CA 证书的去向：我们在决定 CA 证书的去向前，必须先明确它包含了 CA 公钥和 CA 私钥，其中 CA 私钥是用来签署其他证书的，签完后一般不会再用，但极其重要，不能泄露，所以不能将包含私钥的 CA 证书放入项目中，而 CA 公钥是用来验证其他证书是否由该 CA 签发的，经常需要用到，因此我们应当将 CA 公钥从 CA 证书中导出用于使用，而原始 CA 证书则需要妥善保管。

这个操作可以在任意一台节点上完成，比如在 es-node1 上执行即可。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch

# 2. 签发 ca 证书
sudo bin/elasticsearch-certutil ca
"""
---------------------------------------------------------------------------------（要求写入 / 模板写法）
1. Please enter the desired output file [elastic-stack-ca.p12]:
	1. 让你设置证书文件名
	2. 我们直接 Enter，他会默认使用 elastic-stack-ca.p12
2. Enter password for elastic-stack-ca.p12:
	1. 让你为 CA 证书设置一个密码
	2. 我们直接 Enter，不设置
---------------------------------------------------------------------------------
"""


# 3. 导出 CA 公钥
openssl pkcs12 -in elastic-stack-ca.p12 -nokeys -out ca.crt
"""
---------------------------------------------------------------------------------
1. Enter Import Password
	1. 让你输入 CA 证书的密码
	2. 我们直接 Enter，不填写
---------------------------------------------------------------------------------
"""


# 4. 将 CA 证书、CA 公钥放到存放证书目录下
mv elastic-stack-ca.p12 ca.crt config/certs
```

---


###### 1.1.3.7.3. 签发节点证书

由于 Elasticsearch 强调分布式节点间的安全通信，要求各节点使用节点证书实现双向认证和数据加密，因此我们需要为节点生成证书。

这一操作可以在任意拥有 CA 私钥的节点上执行，而我们的 CA 私钥保存在 es-node1，所以就在 es-node1 上完成。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 批量签发节点证书

# 2.1. 创建并编辑 instances.yml（用 notepad++）
vim instances.yml
"""
instances:
  - name: es-node1
    ip:  [192.168.136.8]
  - name: es-node2
    ip:  [192.168.136.9]
  - name: es-node3
    ip:  [192.168.136.10]
---------------------------------------------------------------------------------
instances:
  - name: es-node1
    dns: [es-node.example.com]
    ip:  [192.168.136.8, 192.168.136.9, 192.168.136.10]
  - name: es-node2
    dns: [es-node.example.com]
    ip:  [192.168.136.8, 192.168.136.9, 192.168.136.10]
  - name: es-node3
    dns: [es-node.example.com]
    ip:  [192.168.136.8, 192.168.136.9, 192.168.136.10]
---------------------------------------------------------------------------------
"""

# 2.2. 进行批量签发
bin/elasticsearch-certutil cert \
  --silent \
  --in instances.yml \
  --ca config/certs/elastic-stack-ca.p12 \
  --pem \
  --out certs.zip
"""
1. --silent：
	1. 不加 --silent 时，如果某些参数没写，会提示你输入
	2. 加了 --silent 时，不输出任何交互提示或额外信息，如果参数不完整会直接失败
2. --in：
	1. 从 YAML 中读取节点列表
3. --ca config/certs/elastic-stack-ca.p12：
	1. CA 私钥的位置，这里使用 CA 证书
4. --pem：
	1. 输出 PEM 格式的节点证书和节点私钥
5. --out certs.zip
	1. 输出为 certs.zip
	2. 解压后有多个文件夹，例如 es-node1、es-node2、es-node3
	3. 每个文件夹中都有独自的节点证书和节点私钥，例如 es-node1.key、es-node1.crt
---------------------------------------------------------------------------------
1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
	1. 让你输入 CA 证书的密码
	2. 我们直接 Enter，不填写
---------------------------------------------------------------------------------
""""

# 3. 将 certs.zip 放到存放证书目录下，到时候与 HTTPS 证书一起处理
mv certs.zip config/certs


# 4. 删除 instances.yml
rm -rf /mystudy/es/elasticsearch/instances.yml


# 5. 补充：单独为节点进行签发
# 5.1. 进行单独签发
bin/elasticsearch-certutil cert \
  --name es-node1 \
  --dns es-node1 \
  --ip 192.168.136.8 \
  --ca config/certs/elastic-stack-ca.p12 \
  --out es-node1.zip
"""
---------------------------------------------------------------------------------
1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
	1. 让你输入 CA 证书的密码
	2. 这里不填写
2. Enter password for es-node1.zip：
	1. 让你输入 es-node1.zip 的加密密码
	2. 只有输入你刚设的密码才能解压，我们直接 Enter
---------------------------------------------------------------------------------
"""

# 5.2. 将 certs.zip 放到存放证书目录下，到时候与 HTTPS 证书一起处理
mv es-node1.zip config/certs
```

---


###### 1.1.3.7.4. 签发 HTTPS 证书

这一操作可以在任意拥有 CA 私钥的节点上执行，而我们的 CA 私钥保存在 es-node1，所以同样在 es-node1 上完成。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 批量签发 HTTPS 证书
# 2.1. 创建并编辑 http-instances.yml（用 notepad++）
vim http-instances.yml
"""
instances:
  - name: es-node1-http
    ip:  [ "192.168.136.8" ]
  - name: es-node2-http
    ip:  [ "192.168.136.9" ]
  - name: es-node3-http
    ip:  [ "192.168.136.10" ]
---------------------------------------------------------------------------------
instances:
  - name: es-node1-http
    dns: [ "es-node1" ]
    ip:  [ "192.168.136.8" ]
  - name: es-node2-http
    dns: [ "es-node2" ]
    ip:  [ "192.168.136.9" ]
  - name: es-node3-http
    dns: [ "es-node3" ]
    ip:  [ "192.168.136.10" ]
---------------------------------------------------------------------------------
"""

# 2.2. 进行批量签发
bin/elasticsearch-certutil cert \
  --silent \
  --in http-instances.yml \
  --ca  config/certs/elastic-stack-ca.p12 \
  --pem \
  --out http-certs.zip
"""
---------------------------------------------------------------------------------
1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
	1. 让你输入 CA 证书的密码
	2. 我们直接 Enter，不填写
---------------------------------------------------------------------------------
""""

# 3. 将 http-certs.zip 放到存放证书目录下，到时候与 节点 证书一起处理
mv http-certs.zip config/certs


# 4. 删除 http-instances.yml
rm -rf /mystudy/es/elasticsearch/http-instances.yml


# 5. 补充：单独为节点进行签发
# 5.1. 单独签发
bin/elasticsearch-certutil cert \
  --name es-node1 \
  --dns es-node1 \
  --ip 192.168.136.8 \
  --ca config/certs/elastic-stack-ca.p12 \
  --pem \
  --out es-node1-https.zip
"""
---------------------------------------------------------------------------------
1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
	1. 让你输入 CA 证书的密码
	2. 这里不填写
2. Enter password for es-node3.zip：
	1. 让你输入 es-node1.zip 的加密密码
	2. 只有输入你刚设的密码才能解压，我们直接 Enter
---------------------------------------------------------------------------------
"""

# 5.2. 将 es-node1-https.zip 放到存放证书目录下，到时候与 节点 证书一起处理
mv es-node1-https.zip config/certs
```

---


###### 1.1.3.7.5. 分发证书

1. CA 证书：
	1. 由于 CA 证书中包含 CA 私钥，因此我们需要妥善保管，等到下次需要签发新证书时再取出来使用，否则就让它静静躺在仓库里“吃灰”吧。  
	2. 你可能会问，那我验证这些证书是不是由本 CA 签发的，不也得有 CA 公钥吗？是的，CA 证书中确实同时包含了私钥和公钥。虽然私钥要严密保存，但没有公钥也不行。  
	3. 别担心，我们在生成节点证书的时候，已经从 CA 证书中提取了公钥部分，也就是 `ca.crt` 文件，这就是 CA 的公钥，用这个来验证签名就可以了。
2. CA 公钥（ca.crt）：
	1. 同步到所有节点
3. 节点证书
	1. 各节点就用自己证书。比如 `es-node1` 的证书，别传给 `es-node2` 去用，各用各的，别串。
4. HTTPS 证书：
	1. 同样是各节点使用自己的证书，各用各的，别串。
```
# 1. 进入存放证书的目录
cd /mystudy/es/elasticsearch/config/certs


# 2. 将 CA 证书好好保管，以后需要签发证书再拿出来


# 3. 解压 节点 证书，分配到各自服务器的 /mystudy/es/elasticsearch/config/certs 目录下
unzip certs.zip 


# 4. 解压 HTTPS 证书，分配到各自服务器的 /mystudy/es/elasticsearch/config/certs 目录下
unzip http-certs.zip


# 5. 将 CA 公钥（ca.crt）分配到每个服务器的 /mystudy/es/elasticsearch/config/certs 目录下


# 6. 删除无用文件
rm -rf certs.zip http-certs.zip es-node{1,2,3} es-node{1,2,3}-http
```

---

##### 1.1.3.8. 修改 ES 文件拥有者为 es

因为要启动 ES 了，所以要将 ES 文件拥有者设置为 es，用 es 用户启动 ES
```
chown -R es:es /mystudy/es/elasticsearch
```

> [!NOTE] 注意事项
> 1. 为用户设置密码时，两次回车键就是无密码

---


##### 1.1.3.9. 启动第一个 ES

###### 1.1.3.9.1. 前言

第一个 ES的启动至关重要，因为要初始化 ES 集群的元数据状态，后续要加入集群的节点也要找第一个节点当做中间人。

---


###### 1.1.3.9.2. 配置主配置文件：config/elasticsearch.yml

```
# 1. 修改 ES 配置文件（用 notepad++）
vim /mystudy/es/elasticsearch/config/elasticsearch.yml
"""
# 集群与节点
cluster.name: es-cluster
node.name: es-node1

# 存储路径
path.data: /mystudy/es/elasticsearch/data
path.logs: /mystudy/es/elasticsearch/log

# 网络配置
network.host:
  - _local_
  - _site_
http.port: 9200

# 集群发现
discovery.seed_hosts:
  - "192.168.136.8"
  - "192.168.136.9"
  - "192.168.136.10"
cluster.initial_master_nodes:
  - "es-node1"

# 可选：关闭 GeoIP 自动下载
ingest.geoip.downloader.enabled: false

# 安全配置
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# HTTP 层 TLS
xpack.security.http.ssl:
  enabled: true
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1-http.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1-http.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
  client_authentication: none

# Transport 层 TLS
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
---------------------------------------------------------------------------------
# ================== 基础配置 ==================
# 设置 ES 集群名称
cluster.name: es-cluster                           # 同集群内的所有节点，集群名称必须一致

# 设置当前节点名称
node.name: es-node1                                # 每个 ES 节点在 ES 集群中唯一的名字（不必须主机名，但推荐）

# 设置数据和日志文件路径
path.data: /mystudy/es/elasticsearch/data
path.logs: /mystudy/es/elasticsearch/log

# 设置网络访问地址和端口
network.host:                                      # 配置 Elasticsearch 监听的网卡地址，影响 HTTP（9200）和节点通信（9300）
  - _local_
  - _site_      
http.port: 9200                                    # HTTP 接口已默认使用 9200 端口，如果需要更换可在此修改；
http.host: ["_local_", "_site_"]                   # 进一步细化 HTTP 的监听地址，会覆盖 network.host 的 HTTP 设置，通常无需单独配置 http.host，除非你希望 HTTP 和 Transport 分别绑定在不同的地址（如 HTTP 对外暴露，Transport 用于内网通信）

# 设置初始发现节点（后续节点加入集群中，需要这些节点作为中间人）
discovery.seed_hosts:
  - "192.168.136.8"
  - "192.168.136.9"
  - "192.168.136.10"

# 指定集群初始化主节点名称
cluster.initial_master_nodes:
  - "es-node1"                                      # 需注意，应填节点名称（node.name），不是主机名，也不能填写 IP，并且只需要在第一个节点启动时指定。是用于指定集群初始化时的主节点，负责创建和初始化集群的元数据。一旦集群稳定并完成初始化后，该配置将不再起作用，随后通过内部选举选出的主节点将接管

# ================== 安全配置 ==================
# 开启安全认证
xpack.security.enabled: true                        # 启用后，要使用 ES 内置的用户和角色进行身份验证才能访问集群资源。
xpack.security.enrollment.enabled: true             # 启用后，开启节点间安全通信和身份验证

# 配置 HTTPS（HTTP 层 SSL）
xpack.security.http.ssl:
  enabled: true
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1-http.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1-http.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
  client_authentication: none                       # 是否要求客户端上传证书
   
# 配置节点间通信加密（Transport 层 SSL）
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
---------------------------------------------------------------------------------
"""
```

> [!NOTE] 注意事项：
> 1. `network.host` 和 `http.host` 常用值
> 	1. 见笔记：IP

---


###### 1.1.3.9.3. 启动第一个 ES

```
# 1. 切换 es 用户
su es


# 2. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 3. 启动 ES
# 3.1. 前台启动
bin/elasticsearch

# 3.2. 后台启动
bin/elasticsearch -d


# 4. 保存密码信息
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  dc1hGRVxrQ*QR6rozhaJ



❌ Unable to generate an enrollment token for Kibana instances, try invoking `bin/elasticsearch-create-enrollment-token -s kibana`.

❌ An enrollment token to enroll new nodes wasn't generated. To add nodes and enroll them into this cluster:
• On this node:
  ⁃ Create an enrollment token with `bin/elasticsearch-create-enrollment-token -s node`.
  ⁃ Restart Elasticsearch.
• On other nodes:
  ⁃ Start Elasticsearch with `bin/elasticsearch --enrollment-token <token>`, using the enrollment token that you generated.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


# 5. 提取主要内容
用户名（默认的超级管理员用户）：elastic
初始密码：dc1hGRVxrQ*QR6rozhaJ


# 6. 重设 elastic 密码（如果忘记密码或未记录密码，执行此命令将会设置一个随机密码）
bin/elasticsearch-reset-password -u elastic


# 7. 重设 elastic 密码（适用于已知当前密码的情况，若不记得密码请先通过第 6 步设置随机密码）
curl -k -u elastic:<当前密码> -X POST "https://<本 ES 节点IP>:9200/_security/user/elastic/_password" -H 'Content-Type: application/json' -d '{
  "password" : "<新密码>"
}'


# 8. 补充：如何停止 ES
# 8.1. 查 ES pid
ps aux | grep elasticsearch

# 8.2. 杀死 ES pid
kill -9 123456


# 9. 补充：如何重置 ES（删除 ES 数据即可）
sudo rm -rf /mystudy/es/elasticsearch/data/*
```

> [!NOTE] 注意事项
> 1. 如果出现错误，很可能是由于 Elasticsearch 需要较多的内存映射，而默认的 65530 在多数生产环境中偏低；8.18.0 版本要求至少 **262144** 才能通过检查。
```
sudo vim /etc/sysctl.conf
"""
# 添加这一行
vm.max_map_count=262144
"""


sudo sysctl -p
```

----


###### 1.1.3.9.4. 访问服务器节点

访问服务器节点，返回下述内容即证明部署成功： https://192.168.136.8:9200


![](image-20250417090125699.png)

---


##### 1.1.3.10. 启动其他 ES

```
# 1. 修改 ES 配置文件
vim /mystudy/es/elasticsearch/config/elasticsearch.yml
"""
---------------------------------------------------------------------------------（模板写法，得改改）
# 集群与节点
cluster.name: es-cluster
node.name: es-node1

# 存储路径
path.data: /mystudy/es/elasticsearch/data
path.logs: /mystudy/es/elasticsearch/log

# 网络配置
network.host:
  - _local_
  - _site_
http.port: 9200

# 集群发现
discovery.seed_hosts:
  - "192.168.136.8"
  - "192.168.136.9"
  - "192.168.136.10"

# 可选：关闭 GeoIP 自动下载
ingest.geoip.downloader.enabled: false

# 安全配置
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# HTTP 层 TLS
xpack.security.http.ssl:
  enabled: true
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1-http.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1-http.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
  client_authentication: none

# Transport 层 TLS
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  certificate: /mystudy/es/elasticsearch/config/certs/es-node1.crt
  key: /mystudy/es/elasticsearch/config/certs/es-node1.key
  certificate_authorities:
    - /mystudy/es/elasticsearch/config/certs/ca.crt
---------------------------------------------------------------------------------
"""


# 2. 切换 es 用户
su es


# 3. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 4. 启动 Node ES（后台启动）
bin/elasticsearch -d


# 5. 访问服务器节点
https://192.168.136.X:9200/
```

> [!NOTE] 注意事项
> 1. 配置内容基本上与第一个节点一致，只需要改这三处
> 	1. 设置节点名称：
> 		1. node.name: es-nodeX（要改 1 个）
> 	2. 去掉 cluster.initial_master_nodes
> 	3. TLS 证书配置中，node1 改成nodeX（要改 4 个）

---


##### 1.1.3.11. 检查集群状态

```
https://192.168.136.8:9200/_cluster/health?pretty
"""
---------------------------------------------------------------------------------
{
  "cluster_name" : "es-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 3,
  "active_shards" : 6,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "unassigned_primary_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
---------------------------------------------------------------------------------
"""
```

----


##### 1.1.3.12. 补充说明

###### 1.1.3.12.1. 目录结构

```

```

----


# 补充


## 1. 正向索引

![](image-20250418184335457.png)

----


## 2. 倒排索引

![](image-20250418184408057.png)

----

























