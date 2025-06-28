---
title: 笔记：ElasticSearch 基础
date: 2025-04-13
categories:
  - 数据管理
  - ELK 三件套
  - ElasticSearch
  - Elasticsearch 基础
tags: 
author: 霸天
layout: post
---
# 一、理论

### 1. 导图：[Map：ElasticSearch](../../maps/Map：ElasticSearch.xmind)

---


### 2. ElasticSearch 概述

elasticsearch是一款非常强大的开源搜索引擎，可以帮助我们从海量数据中快速找到需要的内容

---







### 3. ES 语法





---



#### 3.3. 索引命令

##### 3.3.1. 创建索引

###### 3.3.1.1. 创建索引考虑事项

1. ==索引库名称==：
	1. ES索引通常对应数据库中的一张表，一个表根据业务需求可以对应多个ES索引
	2. 这里是一个模版：
		1. `[项目名]_[表名] _ [业务名] _ [时间分片] _ [环境标识]`，例如： `myproject_order_transaction_20250427_prod`
		2. <font color="#00b0f0">表名</font>：
			1. 使用数据库表名的简化版或直接版
			2. 表名较短时可直接使用，如 `t_user` → `user`；
			3. 表名较长时可适当简化
		3. <font color="#00b0f0">业务名</font>：
			1. 表示索引解决的具体业务，如 `email`、`profile`、`address` 等
		4. <font color="#00b0f0">时间分片</font>：
			1. 便于管理大规模数据，如订单表每年10亿数据，可以按月、周、日进行分片
		5. <font color="#00b0f0">环境标识</font>：
			1. 用于区分不同环境，如 `local`、`dev`、`test`、`perf`、`prod` 等
2. ==分片和副本数量==
3. ==自定义分词器==：
	1. 分词器通常由三部分组成：`character filter`、`tokenizer` 和 `tokenizer filter`（简称 `filter`）
	2. 因此，在自定义分词器时，通常先分别自定义 `char_filter`、`tokenizer` 和 `filter`，然后再基于这些组件创建自定义的 `analyzer`。
	3. 当然，自定义分词器并不一定要包含所有三种组件，具体可以根据实际需求选择其中两种或全部三种进行配置
4. ==刷新时间==：
5. ==字段定义==：
	1. <font color="#00b0f0">字段名称</font>：
		1. 通常与数据库字段名保持一致
	2. <font color="#00b0f0">字段数据类型</font>：
		1. 参照 [笔记：数据类型和传参](https://blog.wangjia.xin/2025/04/12/%E7%AC%94%E8%AE%B0%EF%BC%9A%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E5%92%8C%E4%BC%A0%E5%8F%82/#MySQL-ES)
	3. <font color="#00b0f0">插入文档时使用的分词器</font>：
		1. 用于对文档内容进行分词，一般是使用自定义分词器
		2. 主要适用于 `text` 类型字段，非 `text` 字段无需且不能指定分词器，不打算使用这个 `text` 检索，也不需要指定分词器
	4. <font color="#00b0f0">查询文档时使用的分词器</font>：
		1. 用于对搜索内容进行分词，一般是使用 `ik_smart` 和 `ik_max_word`
		2. 如果未显式指定，则默认使用该字段插入文档时使用的分词器
	5. <font color="#00b0f0">是否为该字段创建倒排索引库</font>：
		1. 如果只打算通过 `all` 字段进行搜索，而不需要对该字段进行精确查询，可以设置 `index: false`，以避免为该字段单独建立倒排索引，节省存储空间和索引构建时间。
	6.  <font color="#00b0f0">其他的 Mapping 属性</font>
		1. 参照 [ElasticSearch 文档](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/mapping-params.html)

> [!NOTE] 注意事项
> 1. 一个索引库中并非只存在一份倒排索引，每个字段在设置了 `index: true` 后，都会单独创建一份倒排索引：
> 	- 各字段的倒排索引分别收集该字段在所有文档中的内容
> 	- 搜索时，实际是通过每个字段对应的倒排索引查找匹配的文档，并按相关性打分，从高到低返回对应的文档 ID，最终返回文档内容
> 2. 为了简化跨字段搜索，我们通常会使用 `copy_to` 将多个字段的内容复制到一个统一的字段（如 `all` 字段），实现“一网打尽式”的搜索。但这种方式也带来一些问题：
> 	- <font color="#00b0f0">不该分词的字段分词了</font>：
> 		- `all` 字段通常是 `text` 类型，会进行分词处理。如果将 `keyword` 类型的字段内容也 `copy_to` 到 `all` 字段，它们在 `all` 中会被分词，破坏了原本用于精确匹配的特性，完全背离了它精确匹配的初衷
> 		- 为了规避这种混乱，我们可以：
> 			- 可以将 `keyword` 字段复制到 `all` 字段，但自身仍保留独立索引（`index: true`）
> 			- 精确查询时使用原字段，全文搜索时使用 `all` 字段，各司其职，互不干扰
> 	- <font color="#00b0f0">索引浪费问题</font>：
> 		- 如果仅通过 `all` 字段进行搜索，而其他字段不进行单独查询，则可以将这些字段的 `index` 属性设置为 `false`，避免为每个字段建立独立的倒排索引，从而节省存储和提升索引效率。

```
PUT /<索引库名称>                                       # 索引库名称
{
    "settings": {
        "number_of_shards": 3,                         # 分片数量
        "number_of_replicas": 1,                       # 每个分片的副本数量
        "refresh_interval": "1s",                      # 刷新时间
        "analysis": {
            "char_filter": {                           # 自定义 character filter
                "my_char_filter": {
                }
            },
            "tokenizer": {                             # 自定义 tokenizer
                "my_tokenizer": {
                }
            },
            "filter": {                                # 自定义 tokenizer filter
                "my_tokenizer_filter": {
                }
            },
            "analyzer": {                              # 自定义过滤器
                "my_analyzer" : {
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "<字段名1>": {                              # 字段名称
                "type": "<字段数据类型>",
                "analyzer": "<插入文档时使用的分词器>",
                "search_analyzer": "<查询文档时使用的分词器>",
                "index": true                          # 创建倒排索引库
            },
            "<字段名2>": {
                "type": "text",
                "analyzer": "standard",
                "search_analyzer": "standard",
                "index": true
            }
        }
    }
}
```

---


###### 3.3.1.2. Copy To 小技巧
```
# 1. Copy To All（）
PUT /user_index
{
  "mappings": {
    "properties": {
      "username": {
        "type": "text",
        "analyzer": "ik_smart",
        "copy_to": "all"
      },
      "email": {
        "type": "keyword",
        "index": false,
        "copy_to": "all"
      },
      "profile": {
        "properties": {
          "gender": {
            "type": "keyword",
	        "index": false,                         // Object 类型中的字段分别管理索引
            "copy_to": "all"
          },
          "age": {
            "type": "integer",
			"index": false,
            "copy_to": "all"
          }
        }
      },
      "all": {
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}


# 2. 使用 Copy To 之前的搜索
GET /user_index/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "username": "johndoe" } },
        { "match": { "profile.gender": "male" } },
        { "match": { "profile.age": "18" } }
      ]
    }
  }
}


# 3. 使用 Copy To 之后的搜索
GET /user_index/_search
{
  "query": {
    "match": {
      "all": "johndoe male 18"
    }
  }
}
```

---


#### 4.4. 自定义分词器

自定义分词器



```
# 1. 模版
PUT /<索引库名称>
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {                      // 分词器名称（自定义）
          "tokenizer": "ik_max_word",
          "filter": "pinyin"
        }
      }
    }
  }
}


# 2. 示例（一个还不错的可生产用的自定义分词器，很好解决了我们的问题）
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

> [!NOTE] 注意：
> 1. 拼音分词器的更多配置可以上官网查阅

如家酒店还不错 -> `["如家", "rujia", "rj", "酒店", "jiudian", "jd", "还不", "haibu", "hb", "不错", "bucuo", "bc"`

感觉并没有实现r家这种搜索，虽然自定义分词器在创建索引的时候很好，插入文档后能很好的分词，很不错，，但是如果查询的时候还用这个分词器就不太好了，因为拼音有很多的同声字，例如shizi 有狮子、十字、师资、柿子、虱子等等等等
![](source/_posts/笔记：ElasticSearch%20基础/image-20250427210357524.png)

为了解决这个问头，我们只能创建索引时使用自定义索引，但是查询时不使用自定义索引，应该用户输入拼音，你就那拼音搜，输入中文，就拿中文搜

```
PUT /test
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": {
        "py": { ... }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer",             // 创建索引、插入文档
        "search_analyzer": "ik_smart"          // 搜索时
      }
    }
  }
}
```

---


#### 4.5. 如何抉择分词器

在为文档进行分词的时候，我们希望尽可能的分词，并且还希望同样为阴雨进行分词，例如：`["如家", "rujia", "rj", "酒店", "jiudian", "jd", "还不", "haibu", "hb", "不错", "bucuo", "bc"`

所以在文档分词，即创建索引时，为字段的分词器为自定义分词器
但是在查询时，我们可能会输入 如家、rj、r家等等，如果还使用自定义分词器，发现如果还使用自定义分词器，我们输入 rj可能会查询到 人机、如家、软件、肉鸡等等，啊？？是这样吗？你再补充点，对了，如果r家这该怎么办，好像分词的时候没考虑这些问题啊。等等等等，一系列问题，你要为我总结出来用户可能输入的所有可能，然后我该怎么解决

所以为了避免这些一系列问题，我们推荐使用 ik_max_word 或者 ik_smart

---


### 5. ES 中文搜索优化方案

#### 5.1. 用户输入搜索信息的常见形式

1. ==全称==：
	1. 如家
2. ==拼音全拼==：
	1. rujia
3. ==拼音缩写==：
	1. rj
4. ==汉字 + 拼音混合==：
	1. r家
5. ==谐音字==：
	1. 如加、如佳
6. ==错别字==：
	1. 如加

----



# Elasticsearch 处理谐音字和错别字的解决方案

在 Elasticsearch 中处理谐音字和错别字是中文搜索场景中的常见需求。针对您提到的"如加、如佳"谐音字问题和"如加"错别字问题，以下是完整的解决方案。

## 谐音字处理方案

## 拼音分词器配置

处理谐音字的核心是使用拼音分词器，通过将中文转换为拼音来实现同音字的匹配[1](https://blog.csdn.net/LINING_GG/article/details/128212935)。

json

`PUT /test_index {   "settings": {    "analysis": {      "analyzer": {        "my_pinyin_analyzer": {          "tokenizer": "ik_max_word",          "filter": ["pinyin_filter"]        }      },      "filter": {        "pinyin_filter": {          "type": "pinyin",          "keep_full_pinyin": true,          "keep_original": true,          "keep_first_letter": true,          "keep_separate_first_letter": false,          "limit_first_letter_length": 16,          "lowercase": true,          "remove_duplicated_term": true        }      }    }  },  "mappings": {    "properties": {      "content": {        "type": "text",        "analyzer": "my_pinyin_analyzer",        "search_analyzer": "ik_smart"      }    }  } }`

## 解决同音字问题的关键配置

为了避免同音字过度匹配的问题，需要合理配置索引和搜索分析器[1](https://blog.csdn.net/LINING_GG/article/details/128212935)[2](https://elasticsearch.cn/question/11575)：

- **索引时**：使用包含拼音的分析器，同时保留原始中文
    
- **搜索时**：使用不包含拼音的分析器，避免中文搜索时匹配到所有同音字
    

json

`{   "content": {    "type": "text",    "analyzer": "my_pinyin_analyzer",    "search_analyzer": "ik_smart"  } }`

## 多字段搜索策略

为了平衡搜索的准确性和召回率，可以使用多字段搜索策略[3](https://blog.csdn.net/AiMaiShanHuHai/article/details/144430879)：

json

`{   "query": {    "multi_match": {      "query": "如加",      "fields": [        "content^5",        "content.pinyin^2"      ],      "operator": "or"    }  } }`

## 错别字处理方案

## 模糊查询（Fuzzy Query）

Elasticsearch 提供了基于莱文斯坦距离（Levenshtein Distance）的模糊查询功能来处理拼写错误[4](https://blog.csdn.net/Flying_Fish_roe/article/details/142754951)[5](https://elasticsearch.cn/article/14056)。

json

`{   "query": {    "fuzzy": {      "content": {        "value": "如加",        "fuzziness": "AUTO"      }    }  } }`

## 核心参数配置

**fuzziness 参数**：控制编辑距离的大小[6](https://blog.csdn.net/weixin_42692506/article/details/101555035)[7](https://www.jianshu.com/p/eb5665a9fd70)

- `AUTO`：自动模式，根据词长度调整编辑距离
    
- `0,1,2`：固定的编辑距离值
    
- `AUTO:3,6`：自定义阈值的自动模式
    

**prefix_length 参数**：设置前缀匹配长度，提高查询性能[7](https://www.jianshu.com/p/eb5665a9fd70)

json

`{   "query": {    "match": {      "content": {        "query": "如加",        "fuzziness": "AUTO",        "prefix_length": 1      }    }  } }`

## 建议器（Suggester）纠错

使用 Term Suggester 进行拼写纠错[5](https://elasticsearch.cn/article/14056)：

json

`{   "suggest": {    "text_suggest": {      "text": "如加",      "term": {        "field": "content",        "suggest_mode": "always",        "min_word_length": 2,        "max_edits": 2      }    }  } }`

## 综合解决方案

## 完整的索引配置

json

`PUT /comprehensive_index {   "settings": {    "analysis": {      "analyzer": {        "pinyin_analyzer": {          "tokenizer": "ik_max_word",          "filter": ["pinyin_filter"]        },        "search_analyzer": {          "tokenizer": "ik_smart"        }      },      "filter": {        "pinyin_filter": {          "type": "pinyin",          "keep_full_pinyin": true,          "keep_original": true,          "keep_first_letter": true,          "lowercase": true,          "remove_duplicated_term": true        }      }    }  },  "mappings": {    "properties": {      "content": {        "type": "text",        "analyzer": "pinyin_analyzer",        "search_analyzer": "search_analyzer",        "fields": {          "keyword": {            "type": "keyword"          },          "pinyin": {            "type": "text",            "analyzer": "pinyin_analyzer"          }        }      }    }  } }`

## 多策略搜索查询

结合模糊查询和拼音搜索的综合查询策略[3](https://blog.csdn.net/AiMaiShanHuHai/article/details/144430879)：

json

`{   "query": {    "bool": {      "should": [        {          "match": {            "content": {              "query": "如加",              "boost": 3            }          }        },        {          "match": {            "content.pinyin": {              "query": "如加",              "boost": 2            }          }        },        {          "fuzzy": {            "content": {              "value": "如加",              "fuzziness": "AUTO",              "boost": 1            }          }        }      ]    }  } }`

## 最佳实践建议

1. **性能优化**：使用 `prefix_length` 参数减少模糊查询的计算量[7](https://www.jianshu.com/p/eb5665a9fd70)
    
2. **准确性平衡**：合理设置 `fuzziness` 值，避免过度匹配[6](https://blog.csdn.net/weixin_42692506/article/details/101555035)
    
3. **权重调整**：通过 `boost` 参数调整不同匹配策略的权重[3](https://blog.csdn.net/AiMaiShanHuHai/article/details/144430879)
    
4. **索引优化**：根据业务需求选择合适的分词器组合[1](https://blog.csdn.net/LINING_GG/article/details/128212935)
    

通过以上配置和策略，可以有效处理"如加、如佳"等谐音字问题和"如加"等错别字问题，提升中文搜索的用户体验和准确性[8](https://elastic-search-in-action.medcl.com/3.site_search/3.3.search_box/pinyin_support/)[9](https://developer.baidu.com/article/details/2817267)。

1. [https://blog.csdn.net/LINING_GG/article/details/128212935](https://blog.csdn.net/LINING_GG/article/details/128212935)
2. [https://elasticsearch.cn/question/11575](https://elasticsearch.cn/question/11575)
3. [https://blog.csdn.net/AiMaiShanHuHai/article/details/144430879](https://blog.csdn.net/AiMaiShanHuHai/article/details/144430879)
4. [https://blog.csdn.net/Flying_Fish_roe/article/details/142754951](https://blog.csdn.net/Flying_Fish_roe/article/details/142754951)
5. [https://elasticsearch.cn/article/14056](https://elasticsearch.cn/article/14056)
6. [https://blog.csdn.net/weixin_42692506/article/details/101555035](https://blog.csdn.net/weixin_42692506/article/details/101555035)
7. [https://www.jianshu.com/p/eb5665a9fd70](https://www.jianshu.com/p/eb5665a9fd70)
8. [https://elastic-search-in-action.medcl.com/3.site_search/3.3.search_box/pinyin_support/](https://elastic-search-in-action.medcl.com/3.site_search/3.3.search_box/pinyin_support/)
9. [https://developer.baidu.com/article/details/2817267](https://developer.baidu.com/article/details/2817267)
10. [https://elastic-search-in-action.medcl.com/3.site_search/3.3.search_box/fuzzy_query/](https://elastic-search-in-action.medcl.com/3.site_search/3.3.search_box/fuzzy_query/)
11. [https://blog.csdn.net/u013041642/article/details/94416631](https://blog.csdn.net/u013041642/article/details/94416631)
12. [https://www.cnblogs.com/wufengtinghai/p/15800498.html](https://www.cnblogs.com/wufengtinghai/p/15800498.html)
13. [https://elasticsearch.cn/question/11351](https://elasticsearch.cn/question/11351)
14. [https://www.xiaoyeshiyu.com/post/f969d9e9.html](https://www.xiaoyeshiyu.com/post/f969d9e9.html)
15. [https://hackernoon.com/how-to-use-fuzzy-query-matches-in-elasticsearch-dh1h3167](https://hackernoon.com/how-to-use-fuzzy-query-matches-in-elasticsearch-dh1h3167)
16. [https://abhishek376.wordpress.com/2017/09/30/correcting-typos-and-spelling-mistakes-using-elasticsearch/](https://abhishek376.wordpress.com/2017/09/30/correcting-typos-and-spelling-mistakes-using-elasticsearch/)
17. [https://blog.csdn.net/tsx668899/article/details/135041636](https://blog.csdn.net/tsx668899/article/details/135041636)
18. [https://cloud.tencent.com/developer/article/2220244](https://cloud.tencent.com/developer/article/2220244)
19. [https://patents.google.com/patent/CN106202153A/zh](https://patents.google.com/patent/CN106202153A/zh)
20. [https://blog.csdn.net/Li_services/article/details/119052731](https://blog.csdn.net/Li_services/article/details/119052731)
21. [https://wjw465150.github.io/Elasticsearch/4_1_Dealing_with_language.html](https://wjw465150.github.io/Elasticsearch/4_1_Dealing_with_language.html)
22. [https://www.cnblogs.com/wufengtinghai/p/15811554.html](https://www.cnblogs.com/wufengtinghai/p/15811554.html)
23. [https://developer.aliyun.com/article/1331418](https://developer.aliyun.com/article/1331418)
24. [https://www.aliyun.com/sswb/channel_535235_8.html](https://www.aliyun.com/sswb/channel_535235_8.html)
25. [https://cloud.tencent.com/developer/article/1509405](https://cloud.tencent.com/developer/article/1509405)
26. [https://blog.51cto.com/topic/eszhongwenjiucuosuggest.html](https://blog.51cto.com/topic/eszhongwenjiucuosuggest.html)






# 二、实操：搭建 ES 环境

### 1. ES 高可用实现

#### 1.1. ES 组件

ES 中的集群节点有不同的职责划分，每个节点都可以有以下角色：
1. ==Master==：
	1. <font color="#00b0f0">配置参数</font>：
		1. node.master
	2. <font color="#00b0f0">默认值</font>：
		1. true
	3. <font color="#00b0f0">角色职责</font>：
		1. Master 角色本身不直接执行业务操作，仅参与 Leader（主节点）的选举。
		2. Leader 的核心职责：
			1. <font color="#7030a0">集群状态维护</font>
				1. 接收并处理各节点的心跳，实时跟踪节点的加入与离开
				2. 持集群元数据的最新视图，确保集群一致性
			2. <font color="#7030a0">分片与副本管理</font>
				1. 负责分片（primary 和 replica）的分配、重分配和均衡
				2. 监控副本健康状况，触发副本恢复或重建
			3. <font color="#7030a0">索引管理</font>
				1. 管理所有索引的元数据信息（名称、设置、映射等）
				2. 处理索引的创建、删除、修改请求，保证操作原子性
			4. <font color="#7030a0">全局配置管理</font>
				1. 管理 ILM（Index Lifecycle Management）、Index Templates、Component Templates 等全局配置
				2. 协调集群范围内的参数变更，广播至所有节点
		3. 注意事项：
			1. ES 中的 Master 实际上是 Master-eligible（候选主节点），真正的主节点（Leader）由候选节点通过选举产生，这里我说由 Master 选举 Leader 是方便记忆
			2. Master 节点必须获得 **超过** N/2 + 1 的投票（不含“等于”）才能成为 Leader，N 为历史上加入过集群的节点总数，即使节点已下线，其记录仍计入 N。因此我们必须保证集群中大多数节点存活时才可完成选举，例如 3 节点必须保证 2 节点存活。
			3. 此外我们需要注意，Master 节点的数量一般为奇数个，这样能有效防止脑裂
2. ==Data==：
	1. <font color="#00b0f0">配置参数</font>：
		1. node.data
	2. <font color="#00b0f0">默认值</font>：
		1. true
	3. <font color="#00b0f0">角色职责</font>：
		1. 存储文档数据，持有 primary 和 replica 分片。
		2. 执行数据相关的操作，如 CRUD、查询、聚合分析等。
		3. 处理分片级别的恢复、快照和搜索请求，通常负载较高
3. ==Ingest==：
	1. <font color="#00b0f0">配置参数</font>：
		1. node.ingest
	2. <font color="#00b0f0">默认值</font>：
		1. true
	3. <font color="#00b0f0">角色职责</font>：
		1. 在文档数据存储之前进行预处理，通过 Ingest Pipelines 应用各种 Processor（如 grok、date、set、geoip 等）对文档进行解析、转换、清洗。
		2. 将处理后的文档转发到 Data 节点，减轻客户端或外部系统的预处理压力。
4. ==Coordinating==：
	1. <font color="#00b0f0">配置参数</font>：
		1. 无需显式配置，所有节点默认为协调节点。
	2. <font color="#00b0f0">默认值</font>：
		1. 默认开启，无法修改
	3. <font color="#00b0f0">角色职责</font>：
		1. 路作为客户端请求的入口，解析和路由请求到 Master、Data 或 Ingest 节点。
		2. 将各分片返回的部分结果进行聚合、排序、分页等操作后，统一返回给用户。
		3. 管理 Scroll、Search After、Multi-Search 等跨分片的复杂查询流程。

> [!NOTE] 注意事项
> 1. 默认情况下，每个节点都同时承担上述四种角色，这对测试环境足够；
> 2. 在大型生产环境中，建议拆分角色：
> 	- **Data** 角色负载最重，应部署在 CPU 与内存资源充足的服务器上；
> 	- **Master** 角色建议部署在轻量节点或专用节点上，避免与数据节点争抢资源；
> 	- **Ingest** 角色可根据吞吐量高低独立部署，以优化预处理性能；
> 	- **Coordinating** 角色可部署在与客户端网络延迟低的节点上，提升请求路由效率

---


#### 1.2. ES 精细化把控（ES 本身）

==1.一个 ES 索引库最多能存储多少数据量==
ES 索引库的数据量是所有主分片数据量的总和，理论上来说一个 ES 索引库的数据量是无上限的


==2.一个 ES 索引库推荐存储多少数据量==
将单个索引库的存储量控制在几十 GB 到数百 GB之间，既能保证查询和恢复效率，又方便管理与扩展。


==3.一个分片最多能存储多少数据量==
Lucene引擎对单个分片设定了21.47亿文档的硬性限制（Integer.MAX_VALUE - 128）。假设平均文档大小为1KB，单分片理论存储上限可达20TB，但实际生产环境中受以下因素制约：
1. <font color="#00b0f0">内存分配</font>：
	1. JVM堆内存需满足倒排索引与文档值的驻留需求，建议每GB堆内存对应不超过20GB索引数据
2. <font color="#00b0f0">磁盘性能</font>：
	1. 机械硬盘场景下单个分片超过100GB时，查询延迟可能呈指数级增长
3. <font color="#00b0f0">恢复时效</font>：
	1. 50GB分片在万兆网络环境下恢复时间约为30分钟，超过该阈值将影响故障转移效率

==4.一个分片建议存储多少数据量==
单分片数据量建议 10 - 50 GB，极端场景下不超过 100GB，下面是特殊场景：
1. <font color="#00b0f0">日志分析</font>：
	1. 可将单分片扩容至 100 GB，但需配合冷热节点架构
2. <font color="#00b0f0">向量搜索</font>：
	1. 单分片建议上限压缩至20GB，以确保高维数据检索效率；
3. <font color="#00b0f0">时序数据</font>：
	1. 热数据分片控制在30GB，温/冷数据可放大至80GB。


==5.建议怎样分片、副本==
主分片数 = 总数据量 / 单分片最大数据量，一般是总数据量 / 50GB
副本数，你要有这个概念，一个超高可以的集群，副本数建议2-3个，50GB 以下你就一个副本数就行了
注意事项：
	1. 主分片数应尽量是节点数的整数倍，以确保负载均衡
	2. 主分片数一旦设置无法更改，需提前规划数据增长
	3. 每个节点的分片数 + 副本数建议 20~30个（官方建议是20 ~ 40）个

==数据一致性==

==节点安排==






---


#### 1.3. ES 企业级高可用集群要求







==6.建议怎样安排节点==
Master 节点数为奇数个


---



### 2. 环境搭建

#### 2.1. 分布式集群环境搭建

##### 2.1.1. 架构说明



---


##### 2.1.2. 环境要求

1. ==硬件要求==：
	1. 3 台服务器
	2. 2 核 CPU
	3. 4 GB 内存
	4. 50 GB 可用存储空间
2. ==系统、软件要求==：
	1. Ubuntu 22.04
	2. ElasticSearch 8.18
	3. 相关工具
		1. openssl
			1. 导出 CA 公钥
		2. dos2unix
			1. 将 脚本转化为 Unix 格式
		3. chrony
			1. 时间同步
	4. 时间同步
	5. 创建 es 用户
	6. 关闭 Swap 分区
	7. 开放 9200、9300 TCP端口
	8. 设置主机名、主机名解析

> [!NOTE] 注意事项
> 1. 由于可能在 192.168.136.8 部署 `Kibana`，CPU 和内存可稍多些

---


##### 2.1.3. 节点列表

| IP             | 主机名      | 角色                                       |
| -------------- | -------- | ---------------------------------------- |
| 192.168.136.8  | es-node1 | master-eligible、data、ingest、coordinating |
| 192.168.136.9  | es-node2 | master-eligible、data、ingest、coordinating |
| 192.168.136.10 | es-node3 | master-eligible、data、ingest、coordinating |

---


##### 2.1.4. 相关工具

```
# 1. openssl
command -v openssl >/dev/null 2>&1 || sudo apt-get install -y openssl


# 2. dos2unix
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix


# 3. chrony
command -v chrony >/dev/null 2>&1 || sudo apt-get install -y chrony
```

---


##### 2.1.5. 时间同步

```
# 1. 安装 chrony
sudo apt install -y chrony


# 2. 启动并开启自启动 chrony
sudo systemctl enable chrony && sudo systemctl start chrony
```

---


##### 2.1.6. 创建 es 用户

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


##### 2.1.7. 关闭 Swap 分区

```
# 1. 将 内容注释
vim /etc/fstab
"""
## 将此内容进行注释
## /swap.img       none    swap    sw      0       0
"""


# 2. 立即关闭 Swap 分区
swapoff -a
```

----


##### 2.1.8. 开放 9200、9300 TCP 端口

```
sudo ufw allow 9200/tcp && sudo ufw allow 9300/tcp
```

> [!NOTE] 注意事项
> 1. TCP 端口就是我们常说的 HTTP 端口

---


##### 2.1.9. 设置主机名、主机名互相解析

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

----


##### 2.1.10. 创建 mystudy/es 目录

```
mkdir -p /mystudy/es
```

---


##### 2.1.11. 下载并安装 ES

==1.下载 ES==
从 [ES 下载地址](https://www.elastic.co/cn/downloads/past-releases#elasticsearch)下载 ES 安装包：
![](source/_posts/笔记：ElasticSearch%20基础/image-20250416201625864.png)


==2.安装 ES==
```
# 1. 将 ES 安装包上传至 /mystudy/es 目录


# 2. 进入 /mystudy/es 目录
cd /mystudy/es


# 3. 删除已存在 ES 目录
rm -rf /mystudy/es/elasticsearch


# 4. 解压
tar -zxvf  elasticsearch-8.18.0-linux-x86_64.tar.gz -C /mystudy/es


# 5. 重命名
mv elasticsearch-8.18.0 elasticsearch
```

---


##### 2.1.12. 创建存放 ES 数据的目录

```
mkdir -p /mystudy/es/elasticsearch/data
```

---


##### 2.1.13. 生成 SSL 证书

###### 2.1.13.1. 创建存放证书的目录

```
mkdir -p /mystudy/es/elasticsearch/config/certs
```

---


###### 2.1.13.2. 生成 CA 证书（CA 私钥和 CA 公钥）、CA 公钥

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
1. 要求输入：
	1. Please enter the desired output file [elastic-stack-ca.p12]:
		1. 让你设置证书文件名
		2. 我们直接 Enter，他会默认使用 elastic-stack-ca.p12
	2. Enter password for elastic-stack-ca.p12:
		1. 让你为 CA 证书设置一个密码
		2. 这里不设置
"""


# 3. 导出 CA 公钥
openssl pkcs12 -in elastic-stack-ca.p12 -nokeys -out ca.crt
"""
1. 要求输入：
	1. Enter Import Password
		1. 让你输入 CA 证书的密码
		2. 这里不填写
"""


# 4. 将 CA 证书、CA 公钥放到存放证书目录下
mv elastic-stack-ca.p12 ca.crt config/certs
```

---


###### 2.1.13.3. 签发节点证书

由于 Elasticsearch 强调分布式节点间的安全通信，要求各节点使用节点证书实现双向认证和数据加密，因此我们需要为节点生成证书。

这一操作可以在任意拥有 CA 私钥的节点上执行，而我们的 CA 私钥保存在 es-node1，所以就在 es-node1 上完成。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 批量签发节点证书

# 2.1. 创建并编辑 instances.yml（书写节点信息）
vim instances.yml
"""
1. 模版写法
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

2. 我的写法
instances:
  - name: es-node1
    ip:  [192.168.136.8]
  - name: es-node2
    ip:  [192.168.136.9]
  - name: es-node3
    ip:  [192.168.136.10]
"""

# 2.2. 进行批量签发
bin/elasticsearch-certutil cert \
  --silent \
  --in instances.yml \
  --ca config/certs/elastic-stack-ca.p12 \
  --pem \
  --out certs.zip
"""
1. 参数说明：
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
2. 要求输入：
	1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
		1. 让你输入 CA 证书的密码
		2. 这里不填写
""""

# 2.3. 将 certs.zip 放到存放证书目录下，到时候与 HTTPS 证书一起处理
mv certs.zip config/certs


# 3. 单独为节点进行签发
# 3.1. 进行单独签发
bin/elasticsearch-certutil cert \
  --name es-node1 \
  --dns es-node1 \
  --ip 192.168.136.8 \
  --ca config/certs/elastic-stack-ca.p12 \
  --out es-node1.zip
"""
1. 要求输入：
	1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
		1. 让你输入 CA 证书的密码
		2. 这里不填写
	2. Enter password for es-node1.zip：
		1. 让你输入 es-node1.zip 的加密密码
		2. 只有输入你刚设的密码才能解压，我们直接 Enter
"""

# 3.3. 将 certs.zip 放到存放证书目录下，到时候与 HTTPS 证书一起处理
mv es-node1.zip config/certs
```

---


###### 2.1.13.4. 签发 HTTPS 证书

这一操作可以在任意拥有 CA 私钥的节点上执行，而我们的 CA 私钥保存在 es-node1，所以同样在 es-node1 上完成。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 批量签发 HTTPS 证书

# 2.1. 创建并编辑 http-instances.yml
vim http-instances.yml
"""
1. 模版写法
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

2.我的写法
instances:
  - name: es-node1-http
    ip:  [ "192.168.136.8" ]
  - name: es-node2-http
    ip:  [ "192.168.136.9" ]
  - name: es-node3-http
    ip:  [ "192.168.136.10" ]
"""

# 2.2. 进行批量签发
bin/elasticsearch-certutil cert \
  --silent \
  --in http-instances.yml \
  --ca  config/certs/elastic-stack-ca.p12 \
  --pem \
  --out http-certs.zip
"""
1. 要求输入：
	1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
		1. 让你输入 CA 证书的密码
		2. 这里不填写
""""

# 2.3. 将 http-certs.zip 放到存放证书目录下，到时候与 节点 证书一起处理
mv http-certs.zip config/certs


# 3. 单独为节点进行签发
# 3.1. 进行单独签发
bin/elasticsearch-certutil cert \
  --name es-node1 \
  --dns es-node1 \
  --ip 192.168.136.8 \
  --ca config/certs/elastic-stack-ca.p12 \
  --pem \
  --out es-node1-https.zip
"""
1. 要求输入：
	1. Enter password for CA (config/certs/elastic-stack-ca.p12)：
		1. 让你输入 CA 证书的密码
		2. 这里不填写
	2. Enter password for es-node3.zip：
		1. 让你输入 es-node1.zip 的加密密码
		2. 只有输入你刚设的密码才能解压，我们直接 Enter
"""

# 3.3. 将 http-certs.zip 放到存放证书目录下，到时候与 节点 证书一起处理
mv es-node1.zip config/certs
```

---


###### 2.1.13.5. 分发证书

1. ==CA 证书==：
	1. 由于 CA 证书中包含 CA 私钥，因此我们需要妥善保管，等到下次需要签发新证书时再取出来使用，否则就让它静静躺在仓库里“吃灰”吧。  
	2. 你可能会问，那我验证这些证书是不是由本 CA 签发的，不也得有 CA 公钥吗？是的，CA 证书中确实同时包含了私钥和公钥。虽然私钥要严密保存，但没有公钥也不行。  
	3. 别担心，我们在生成节点证书的时候，已经从 CA 证书中提取了公钥部分，也就是 `ca.crt` 文件，这就是 CA 的公钥，用这个来验证签名就可以了。
2. ==CA 公钥（ca.crt）==：
	1. 同步到所有节点
3. ==节点证书==
	1. 各节点就用自己证书。比如 `es-node1` 的证书，别传给 `es-node2` 去用，各用各的，别串。
4. ==HTTPS 证书==：
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
```

---

##### 2.1.14. 修改 ES 文件拥有者为 es

因为要启动 ES 了，所以7要将 ES 文件拥有者设置为 es，用 es 用户启动 ES
```
chown -R es:es /mystudy/es/elasticsearch
```

> [!NOTE] 注意事项
> 1. 为用户设置密码时，两次回车键就是无密码

---


##### 2.1.15. 启动第一个 ES

###### 2.1.15.1. 前言

第一个 ES的启动至关重要，因为要初始化 ES 集群的元数据状态，后续要加入集群的节点也要找第一个节点当做中间人。

---


###### 2.1.15.2. 配置主配置文件：config/elasticsearch.yml

```
# 1. 修改 ES 配置文件
vim /mystudy/es/elasticsearch/config/elasticsearch.yml
"""
1. 注解版本
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
  - "es-node1"                                      # 需注意，应填节点名称，不能填写 IP、主机名，并且只需要在第一个节点启动时指定。是用于指定集群初始化时的主节点，负责创建和初始化集群的元数据。一旦集群稳定并完成初始化后，该配置将不再起作用，随后通过内部选举选出的主节点将接管

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

2. 精简版本
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
"""
```

> [!NOTE] 注意事项：
> 1. `network.host` 和 `http.host` 常用值
> 	1. 见 IP 笔记

---


###### 2.1.15.3. 启动第一个 ES

```
# 1. 切换 es 用户
su es


# 2. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 3. 启动 ES（初次启动推荐前台启动）
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
ps aux | grep elasticsearch

kill -9 123456


# 9. 补充：如何重置 ES（删除 ES 数据即可）
sudo rm -rf /mystudy/es/elasticsearch/data/*
```

----


###### 2.1.15.4. 访问服务器节点

访问服务器节点，返回下述内容即证明部署成功： https://192.168.136.8:9200


![](source/_posts/笔记：ElasticSearch%20基础/image-20250417090125699.png)

---


##### 2.1.16. 启动其他 ES

```
# 1. 修改 ES 配置文件
vim /mystudy/es/elasticsearch/config/elasticsearch.yml
"""
1. 注意事项
	1. 配置内容基本上与第一个节点一致，只需要改这三处
		1. 设置节点名称：
			1. node.name: es-nodeX
		2. 去掉 cluster.initial_master_nodes
		3. TLS 证书配置中，node1 改成nodeX

2. 我的写法
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

---


##### 2.1.17. 脚本大帝

###### 2.1.17.1. 步骤

```
# 1. 配置 SSH 密钥对认证
sudo ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

ssh-keyscan -H 192.168.136.9 >> ~/.ssh/known_hosts 

ssh-keyscan -H 192.168.136.10 >> ~/.ssh/known_hosts

ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.136.9

ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.136.10

ssh root@192.168.136.9

exit


# 2. 设置代理，启动 NAT 服务
export http_proxy="http://172.20.10.3:7890" && export https_proxy="http://172.20.10.3:7890" && export no_proxy="localhost,127.0.0.1,.svc,.cluster.local,192.168.136.0/24,10.96.0.1,10.244.0.0/16" && export HTTP_PROXY=$http_proxy && export HTTPS_PROXY=$https_proxy && export NO_PROXY=$no_proxy


# 3. 创建 /mystudy/es 目录
mkdir -p /mystudy/es


# 4. 进入 /mystudy/es 目录
cd /mystudy/es


# 5. 上传 ES 文件到 /mystudy/es 目录


# 6. 创建并编写 Shell 脚本
sudo vim es-shell.sh


# 7. 添加可执行权限
chmod +x es-shell.sh


# 8. 安装 dos2unix
command -v dos2unix >/dev/null 2>&1 || sudo apt-get install -y dos2unix


# 9. 将脚本转为 Unix 格式
dos2unix es-shell.sh


# 10. 执行 Shell 脚本
./es-shell.sh


# 11. 从 启动第一个 ES 继续向下手动执行
```

---


###### 2.1.17.2. 脚本精简版

```
#!/bin/bash

# -------------------------------- 开启严格模式 ---------------------------------------------
set -euo pipefail

# -------------------------------- 检查是否以 root 权限运行 ---------------------------------------------
if [ "$(id -u)" != "0" ]; then
    echo "这个脚本需要以 root 用户运行" >&2
    exit 1
fi

# -------------------------------- 声明变量 ---------------------------------------------

work_directory="/mystudy/es/elasticsearch"
es_package="elasticsearch-8.18.0-linux-x86_64.tar.gz"
es_init_name="elasticsearch-8.18.0"

# -------------------------------- 安装 openssl ---------------------------------------------
echo "开始安装 openssl"
echo "检查 openssl 是否已安装"
if command -v openssl >/dev/null 2>&1; then
    echo "openssl 已安装，跳过安装"
else
    echo "未检测到 openssl，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get install -y openssl; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 openssl 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
    done
    echo "openssl 安装完成"
fi

# -------------------------------- 安装 chrony ---------------------------------------------
echo "开始安装 chrony"
echo "检查 chrony 是否已安装"
if command -v chronyd >/dev/null 2>&1; then
    echo "chrony 已安装，跳过安装"
else
    echo "未检测到 chrony，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get update && apt-get install -y chrony; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 chrony 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
        sleep 2
    done
    echo "chrony 安装完成"
fi

# -------------------------------- 时间同步 ---------------------------------------------
echo "开始时间同步"
echo "正在启动 chrony 服务..."
if systemctl enable chrony && systemctl start chrony; then
    echo "chrony 服务已启动"
else
    echo "启动 chrony 失败" >&2
fi

# -------------------------------- 创建 es 用户 ---------------------------------------------
echo "开始创建 es 用户"
echo "删除已有 es 用户（包括其主目录）"
if id "es" &>/dev/null; then
    echo "检测到已有 es 用户，正在删除..."
    userdel -r es
    echo "已删除旧的 es 用户"
fi
echo "新增 es 用户"
useradd -m -s /bin/bash es
echo "为 es 用户设置密码（wq666）"
passwd es
echo "创建 es 用户完成"

# -------------------------------- 关闭 Swap 分区 ---------------------------------------------
echo "开始关闭 Swap 分区"
echo "将 内容注释"
sed -i '/^[^#].*\bswap\b/ s/^/#/' /etc/fstab
echo "立即关闭 Swap 分区"
swapoff -a
echo "关闭 Swap 分区完成"

# -------------------------------- 开放 9200、9300 TCP 端口 ---------------------------------------------
echo "开始开放 9200、9300 TCP 端口"
ufw allow 9200/tcp && ufw allow 9300/tcp
echo "开放 9200、9300 TCP 端口完成"

# -------------------------------- 设置主机名、主机名互相解析 ---------------------------------------------
echo "开始设置主机名、主机名互相解析"
echo "获取本机内网 IP 地址"
INTERFACE=$(ip link | grep -oP '^[0-9]+: \K[^:]+' | grep -v lo | head -1)
LOCAL_IP=$(ip addr show "$INTERFACE" | grep -oP 'inet \K[\d.]+' | head -1)
if [ -z "$LOCAL_IP" ]; then
    echo "不能获取本机 IP 地址" >&2
    exit 1
fi
echo "本机内网 IP 地址获取成功：$LOCAL_IP"
echo "设置本机主机名"
case "$LOCAL_IP" in
    "192.168.136.8")
        HOSTNAME="es-node1"
        ;;
    "192.168.136.9")
        HOSTNAME="es-node2"
        ;;
    "192.168.136.10")
        HOSTNAME="es-node3"
        ;;
    *)
        echo "IP $LOCAL_IP does not match any configured hostname" >&2
        exit 1
        ;;
esac
hostnamectl set-hostname "$HOSTNAME"
echo "设置主机名互相解析"
HOSTS_CONTENT="
192.168.136.8   es-node1
192.168.136.9   es-node2
192.168.136.10  es-node3"
echo "检查 /etc/hosts 是否已包含指定追加内容"
if grep -Fx "$HOSTS_CONTENT" /etc/hosts > /dev/null; then
    echo "主机名解析记录已存在，跳过添加"
else
    echo "$HOSTS_CONTENT" >> /etc/hosts
    echo "主机名解析记录已添加"
fi
echo "设置主机名、主机名互相解析完成"

# -------------------------------- 安装 ES ---------------------------------------------
echo "开始安装 ES"
echo "进入 /mystudy/es 目录"
cd /mystudy/es
echo "删除已存在 ES 目录"
if [ -d "$work_directory" ]; then
    echo "检测到已存在 ES 目录，准备删除..."
    rm -rf "$work_directory"
    echo "ES 目录已删除"
fi
echo "检查是否存在 ES 安装包"
if [ ! -f "$es_package" ]; then
    echo "ES 安装包不存在" >&2
    exit 1
fi
echo "解压"
tar -zxvf $es_package -C /mystudy/es
echo "重命名"
mv "$es_init_name" elasticsearch
echo "安装 ES 完成"

# -------------------------------- 创建存放 ES 数据的目录 ---------------------------------------------
echo "开始创建存放 ES 数据的目录"
mkdir -p $work_directory/data
echo "创建存放 ES 数据的目录完成"

# -------------------------------- 创建存放证书的目录 ---------------------------------------------
echo "开始创建存放证书的目录"
mkdir -p $work_directory/config/certs
echo "创建存放证书的目录完成"

# -------------------------------- 生成 CA 证书、CA 公钥（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上生成 CA 证书、CA 公钥"
    echo "进入 ES 目录"
    cd $work_directory
    echo "签发 ca 证书"
    bin/elasticsearch-certutil ca
    echo "导出 CA 公钥"
    openssl pkcs12 -in elastic-stack-ca.p12 -nokeys -out ca.crt
    echo "将 CA 证书、CA 公钥放到存放证书目录下"
    mv elastic-stack-ca.p12 ca.crt $work_directory/config/certs/
    echo "生成 CA 证书、CA 公钥完成"
fi
# -------------------------------- 签发节点证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上签发节点证书"
    echo "进入 ES 目录"
    cd $work_directory
    echo "批量签发节点证书"
    echo "创建并编辑 instances.yml"
    cat > instances.yml <<EOF
instances:
  - name: es-node1
    ip:
      - "192.168.136.8"
  - name: es-node2
    ip:
      - "192.168.136.9"
  - name: es-node3
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
        --silent \
        --in instances.yml \
        --ca config/certs/elastic-stack-ca.p12 \
        --pem \
        --out certs.zip
    echo "将 certs.zip 放到存放证书目录下"
    mv certs.zip config/certs/
    rm -rf instances.yml
    echo "签发节点证书完成"
fi
# -------------------------------- 签发 HTTPS 证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在192.168.136.8 上签发 HTTPS 证书"
    echo "进入 ES 目录"
    cd $work_directory
    echo "批量签发 HTTPS 证书"
    echo "创建并编辑 http-instances.yml"
    cat > http-instances.yml << EOF
instances:
  - name: es-node1-http
    ip:
      - "192.168.136.8"
  - name: es-node2-http
    ip:
      - "192.168.136.9"
  - name: es-node3-http
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
      --silent \
      --in http-instances.yml \
      --ca  config/certs/elastic-stack-ca.p12 \
      --pem \
      --out http-certs.zip
    mv http-certs.zip config/certs/
    rm -rf http-instances.yml
    echo "签发 HTTPS 证书完成"
fi

# -------------------------------- 分发证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上分发证书"
    cd $work_directory/config/certs
    echo "分发节点证书"
    unzip certs.zip 
    mv $work_directory/config/certs/es-node1/es-node1.{crt,key} $work_directory/config/certs/
    scp $work_directory/config/certs/es-node2/es-node2.{crt,key} \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/es-node3/es-node3.{crt,key} \
        root@192.168.136.10:$work_directory/config/certs/
    echo "分发HTTPS 证书"
    unzip http-certs.zip
    mv $work_directory/config/certs/es-node1-http/es-node1-http.{crt,key} $work_directory/config/certs/
    scp $work_directory/config/certs/es-node2-http/es-node2-http.{crt,key} \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/es-node3-http/es-node3-http.{crt,key} \
        root@192.168.136.10:$work_directory/config/certs/
    echo "分发 CA 公钥"
    scp $work_directory/config/certs/ca.crt \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/ca.crt \
        root@192.168.136.10:$work_directory/config/certs/
    rm -rf es-node1 es-node1-http es-node2 es-node2-http es-node3 es-node3-http certs.zip http-certs.zip
    echo "分发证书完成"
fi

# -------------------------------- 修改 ES 文件拥有者为 es ---------------------------------------------
echo "开始修改 ES 文件拥有者为 es"
chown -R es:es $work_directory
echo "修改 ES 文件拥有者为 es 完成"
echo "本节点脚本执行结束"
```

---


###### 2.1.17.3. 脚本注释版

```
#!/bin/bash

# -------------------------------- 开启严格模式 ---------------------------------------------
set -euo pipefail

# -------------------------------- 检查是否以 root 权限运行 ---------------------------------------------
if [ "$(id -u)" != "0" ]; then
    echo "这个脚本需要以 root 用户运行" >&2
    exit 1
fi

# -------------------------------- 声明变量 ---------------------------------------------

work_directory="/mystudy/es/elasticsearch"
es_package="elasticsearch-8.18.0-linux-x86_64.tar.gz"
es_init_name="elasticsearch-8.18.0"

# -------------------------------- 安装 openssl ---------------------------------------------
echo "开始安装 openssl"
echo "检查 openssl 是否已安装"
if command -v openssl >/dev/null 2>&1; then
    echo "openssl 已安装，跳过安装"
else
    echo "未检测到 openssl，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get install -y openssl; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 openssl 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
    done
    echo "openssl 安装完成"
fi
"""
1. command -v openssl：
	1. command -v openssl 用于检测 openssl 命令是否存在于当前系统的环境变量 PATH 指定的路径中
	2. 如果找到了，会输出完整路径（例如 /usr/bin/openssl），并返回退出码 0（true），表示命令执行成功、任务也成功（即找到了）
	3. 如果找不到，不会输出任何内容，但命令本身仍然成功执行，只是任务失败（未找到命令），这时返回退出码为 1（false）
2. >/dev/null
	1. > 是输出重定向符，默认只作用于标准输出（stdout）。
	2. /dev/null 是 Linux 中的“黑洞”文件，任何输出重定向到这里都会被吞掉，相当于“我不想看到这个输出”。
	3. 因此，>/dev/null 表示：将命令的标准输出重定向到黑洞中，不显示在终端上。
3. 2>&1
	1. 1 表示标准输出，2 表示标准错误输出
	2. 2>&1 的意思是：“将标准错误重定向到标准输出的输出位置上”
	3. 因为前面已经执行了 >/dev/null，所以标准输出已经被扔进黑洞了，这时候标准错误也跟着一起被重定向到黑洞。
4. if...then...else...fi
5. max_retries=3
	1. 最大重试次数，总共 3 + 1 次
6. retry_count=0
	1. 已重试的次数
7. until apt-get install -y openssl; do
	1. 执行 apt-get install -y openssl; 成功就退出循环，不成功就继续循环
"""

# -------------------------------- 安装 chrony ---------------------------------------------
echo "开始安装 chrony"
echo "检查 chrony 是否已安装"
if command -v chronyd >/dev/null 2>&1; then
    echo "chrony 已安装，跳过安装"
else
    echo "未检测到 chrony，正在安装..."
    max_retries=3
    retry_count=0
    until apt-get update && apt-get install -y chrony; do
        retry_count=$((retry_count + 1))
        if [ "$retry_count" -ge "$max_retries" ]; then
            echo "安装 chrony 已尝试 ${retry_count} 次，仍然失败" >&2
            exit 1
        fi
        echo "安装失败，第 ${retry_count} 次重试，立即重新尝试..."
        sleep 2
    done
    echo "chrony 安装完成"
fi

# -------------------------------- 时间同步 ---------------------------------------------
echo "开始时间同步"
echo "正在启动 chrony 服务..."
if systemctl enable chrony && systemctl start chrony; then
    echo "chrony 服务已启动"
else
    echo "启动 chrony 失败" >&2
fi

# -------------------------------- 创建 es 用户 ---------------------------------------------
echo "开始创建 es 用户"
echo "删除已有 es 用户（包括其主目录）"
if id "es" &>/dev/null; then
    echo "检测到已有 es 用户，正在删除..."
    userdel -r es
    echo "已删除旧的 es 用户"
fi
echo "新增 es 用户"
useradd -m -s /bin/bash es
echo "为 es 用户设置密码（wq666）"
passwd es
echo "创建 es 用户完成"
"""
1. id "es"
	1. 用于检查 es 用户是否存在
	2. 如果存在，会输出例如：uid=1001(es) gid=1001(es) groups=1001(es)
2. userdel -r es
	1. 删除用户和其主目录
"""

# -------------------------------- 关闭 Swap 分区 ---------------------------------------------
echo "开始关闭 Swap 分区"
echo "将 内容注释"
sed -i '/^[^#].*\bswap\b/ s/^/#/' /etc/fstab
echo "立即关闭 Swap 分区"
swapoff -a
echo "关闭 Swap 分区完成"

# -------------------------------- 开放 9200、9300 TCP 端口 ---------------------------------------------
echo "开始开放 9200、9300 TCP 端口"
ufw allow 9200/tcp && ufw allow 9300/tcp
echo "开放 9200、9300 TCP 端口完成"

# -------------------------------- 设置主机名、主机名互相解析 ---------------------------------------------
echo "开始设置主机名、主机名互相解析"
echo "获取本机内网 IP 地址"
INTERFACE=$(ip link | grep -oP '^[0-9]+: \K[^:]+' | grep -v lo | head -1)
LOCAL_IP=$(ip addr show "$INTERFACE" | grep -oP 'inet \K[\d.]+' | head -1)
if [ -z "$LOCAL_IP" ]; then
    echo "不能获取本机 IP 地址" >&2
    exit 1
fi
echo "本机内网 IP 地址获取成功：$LOCAL_IP"
echo "设置本机主机名"
case "$LOCAL_IP" in
    "192.168.136.8")
        HOSTNAME="es-node1"
        ;;
    "192.168.136.9")
        HOSTNAME="es-node2"
        ;;
    "192.168.136.10")
        HOSTNAME="es-node3"
        ;;
    *)
        echo "IP $LOCAL_IP does not match any configured hostname" >&2
        exit 1
        ;;
esac
hostnamectl set-hostname "$HOSTNAME"
echo "设置主机名互相解析"
HOSTS_CONTENT="
192.168.136.8   es-node1
192.168.136.9   es-node2
192.168.136.10  es-node3"
echo "检查 /etc/hosts 是否已包含指定追加内容"
if grep -Fx "$HOSTS_CONTENT" /etc/hosts > /dev/null; then
    echo "主机名解析记录已存在，跳过添加"
else
    echo "$HOSTS_CONTENT" >> /etc/hosts
    echo "主机名解析记录已添加"
fi
echo "设置主机名、主机名互相解析完成"

# -------------------------------- 安装 ES ---------------------------------------------
echo "开始安装 ES"
echo "进入 /mystudy/es 目录"
cd /mystudy/es
echo "删除已存在 ES 目录"
if [ -d "$work_directory" ]; then
    echo "检测到已存在 ES 目录，准备删除..."
    rm -rf "$work_directory"
    echo "ES 目录已删除"
fi
echo "检查是否存在 ES 安装包"
if [ ! -f "$es_package" ]; then
    echo "ES 安装包不存在" >&2
    exit 1
fi
echo "解压"
tar -zxvf $es_package -C /mystudy/es
echo "重命名"
mv "$es_init_name" elasticsearch
echo "安装 ES 完成"
"""
1. [ -d "path" ]:
	1. path 是一个存在的目录吗？
	2. 类似的有：
		1. -f "path"：
			1. path 是一个存在的文件吗？
		2. -e "path"：
			1. path 是一个存在的路径吗？（文件或目录）
"""

# -------------------------------- 创建存放 ES 数据的目录 ---------------------------------------------
echo "开始创建存放 ES 数据的目录"
mkdir -p $work_directory/data
echo "创建存放 ES 数据的目录完成"

# -------------------------------- 创建存放证书的目录 ---------------------------------------------
echo "开始创建存放证书的目录"
mkdir -p $work_directory/config/certs
echo "创建存放证书的目录完成"

# -------------------------------- 生成 CA 证书、CA 公钥（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上生成 CA 证书、CA 公钥"
    echo "进入 ES 目录"
    cd $work_directory
    echo "签发 ca 证书"
    bin/elasticsearch-certutil ca
    echo "导出 CA 公钥"
    openssl pkcs12 -in elastic-stack-ca.p12 -nokeys -out ca.crt
    echo "将 CA 证书、CA 公钥放到存放证书目录下"
    mv elastic-stack-ca.p12 ca.crt $work_directory/config/certs/
    echo "生成 CA 证书、CA 公钥完成"
fi
# -------------------------------- 签发节点证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上签发节点证书"
    echo "进入 ES 目录"
    cd $work_directory
    echo "批量签发节点证书"
    echo "创建并编辑 instances.yml"
    cat > instances.yml <<EOF
instances:
  - name: es-node1
    ip:
      - "192.168.136.8"
  - name: es-node2
    ip:
      - "192.168.136.9"
  - name: es-node3
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
        --silent \
        --in instances.yml \
        --ca config/certs/elastic-stack-ca.p12 \
        --pem \
        --out certs.zip
    echo "将 certs.zip 放到存放证书目录下"
    mv certs.zip config/certs/
    rm -rf instances.yml
    echo "签发节点证书完成"
fi
# -------------------------------- 签发 HTTPS 证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在192.168.136.8 上签发 HTTPS 证书"
    echo "进入 ES 目录"
    cd $work_directory
    echo "批量签发 HTTPS 证书"
    echo "创建并编辑 http-instances.yml"
    cat > http-instances.yml << EOF
instances:
  - name: es-node1-http
    ip:
      - "192.168.136.8"
  - name: es-node2-http
    ip:
      - "192.168.136.9"
  - name: es-node3-http
    ip:
      - "192.168.136.10"
EOF
    echo "进行批量签发"
    bin/elasticsearch-certutil cert \
      --silent \
      --in http-instances.yml \
      --ca  config/certs/elastic-stack-ca.p12 \
      --pem \
      --out http-certs.zip
    mv http-certs.zip config/certs/
    rm -rf http-instances.yml
    echo "签发 HTTPS 证书完成"
fi

# -------------------------------- 分发证书（192.168.136.8） ---------------------------------------------
if [ "$LOCAL_IP" = "192.168.136.8" ]; then
    echo "开始在 192.168.136.8 上分发证书"
    cd $work_directory/config/certs
    echo "分发节点证书"
    unzip certs.zip 
    mv $work_directory/config/certs/es-node1/es-node1.{crt,key} $work_directory/config/certs/
    scp $work_directory/config/certs/es-node2/es-node2.{crt,key} \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/es-node3/es-node3.{crt,key} \
        root@192.168.136.10:$work_directory/config/certs/
    echo "分发HTTPS 证书"
    unzip http-certs.zip
    mv $work_directory/config/certs/es-node1-http/es-node1-http.{crt,key} $work_directory/config/certs/
    scp $work_directory/config/certs/es-node2-http/es-node2-http.{crt,key} \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/es-node3-http/es-node3-http.{crt,key} \
        root@192.168.136.10:$work_directory/config/certs/
    echo "分发 CA 公钥"
    scp $work_directory/config/certs/ca.crt \
        root@192.168.136.9:$work_directory/config/certs/
    scp $work_directory/config/certs/ca.crt \
        root@192.168.136.10:$work_directory/config/certs/
    rm -rf es-node1 es-node1-http es-node2 es-node2-http es-node3 es-node3-http certs.zip http-certs.zip
    echo "分发证书完成"
fi

# -------------------------------- 修改 ES 文件拥有者为 es ---------------------------------------------
echo "开始修改 ES 文件拥有者为 es"
chown -R es:es $work_directory
echo "修改 ES 文件拥有者为 es 完成"
echo "本节点脚本执行结束"
```

---





### 3. 高可用生产集群搭建

### 4. 常见问题


#### 4.1. 停掉几个节点之后，ES 集群不可用

在停掉几个节点后，发现 Elasticsearch 和 Kibana 都无法使用，这很可能是由于 Elasticsearch 集群不可用，可能原因是无法选出主节点。

我们之前说：Master 节点必须获得 **超过** N/2 + 1 的投票（不含“等于”）才能成为 Leader，N 为历史上加入过集群的节点总数，即使节点已下线，其记录仍计入 N。因此我们必须保证集群中大多数节点存活时才可完成选举，例如 3 节点必须保证 2 节点存活。

所以，通过启动两台服务器，问题应该能得到解决。
> [!NOTE] 注意事项
> 1. 以上说明的前提是：每个节点都是 Master 节点

---


# 三、工具

### 1. Kibana

#### 1.1. Kibana 概述

Kibana 是 ELK 三件套中的 “K”：
- **E**lasticsearch：负责数据的存储、计算与搜索
- **L**ogstash（或 Filebeat）：负责数据抓取
- **K**ibana：负责数据的可视化与分析

---


#### 1.2. Kibana 高可用实现

##### 1.2.1. 节点规划

Kibana 的部署数量没有强制要求，也不需要必须部署在 Elasticsearch 所在的服务器上。只要 Kibana 能访问 Elasticsearch 集群，就可以部署在任意一台服务器上。根据集群规模和可用性需求，推荐的部署数量如下：

| ES 节点数量  | Kibana 节点数量                  |
| -------- | ---------------------------- |
| 1 ~ 3 个  | 1 个 Kibana 实例即可满足需求          |
| 3 ~ 10 个 | 1 ~ 2 个 Kibana + 反向代理 + 负载均衡 |
| 高可用场景    | 多个 Kibana 实例 + 反向代理 + 负载均衡   |

---


#### 1.3. Kibana 使用

##### 1.3.1. 节点列表

由于当前处于测试学习阶段，且节点资源有限，暂不考虑实现高可用架构。此处仅部署一个 Kibana 实例，部署位置为服务器 `192.168.136.8`。

| IP            | 主机名   | 介绍                                              |
| ------------- | ----- | ----------------------------------------------- |
| 192.168.136.8 | node1 | master-eligible、data、ingest、coordinating、kibana |

---


##### 1.3.2. 下载 Kibana 安装包

从 [Kibana 下载地址](https://www.elastic.co/cn/downloads/past-releases#kibana)下载 Kibana 安装包，注意要和 ES 版本一致：
![](source/_posts/笔记：ElasticSearch%20基础/image-20250417102403638.png)

---


##### 1.3.3. 将 Kibana 安装包上传到机器并解压

在此步骤中，我将 Kibana 安装包上传至 `/mystudy/kibana` 目录，并直接在该目录中进行了解压。
```
# 1. 进入目录
cd /mystudy/kibana


# 2. 解压
tar -zxvf  kibana-8.18.0-linux-x86_64.tar.gz -C /mystudy/kibana


# 3. 重命名
mv kibana-8.18.0 kibana
```

---


##### 1.3.4. 为 Kibana 生成 HTTPS 证书

由于我们需要通过浏览器访问 Kibana，并要以 HTTPS 的方式进行安全访问，因此需要为 Kibana 配置 HTTPS 证书。

如果 kibana 和 es 在同一台服务器上，我们完全可以服用es 的https 证书，当然，如果 es 和 kibana 不在同一服务器上，我们需要为 kibana 整一个https 证书，我们可以使用es 提供的ca签发https 证书，也可以使用自己的方式签发 https 证书，这都无所谓，因为是我们客户端访问kibana，无需ca 一致，当然这是私网，如果公网访问还是要正儿八经去权威ca申请https 证书

但是建议 ca 一致，如果 Kibana 使用的证书和 Elasticsearch 使用的证书都来自同一个 CA，Elasticsearch 会默认信任来自 Kibana 的请求（如果开启了 `xpack.security` 和 `ssl`）。

那我们用 ca 证书签一个吧，一般在 ca 服务器上，因为有es 提供的cert 工具



```
cd /mystudy/es/elasticsearch

bin/elasticsearch-certutil cert \
  --name kibana1 \
  --ip 192.168.136.8 \
  --ca config/certs/elastic-stack-ca.p12 \
  --pem \
  --out kibana1-certs.zip



拿到 kibana1-certs.zip

解压文件
unzip kibana1-certs.zip


# 4. 将解压后的文件移动到 kibana 的 config 目录
cd /mystudy/kibana/kibana/config

mv kibana/kibana.csr kibana/kibana.key /mystudy/kibana/kibana/config/
```

```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 为 Kibana 生成 csr（一次空格）
bin/elasticsearch-certutil csr -name kibana -dns node1


# 3. 解压文件
unzip csr-bundle.zip


# 4. 将解压后的文件移动到 kibana 的 config 目录
mv kibana/kibana.csr kibana/kibana.key /mystudy/kibana/kibana/config/


# 5. 进入 Kibana/config 目录
cd /mystudy/kibana/kibana/config


# 6. 生成自签名证书
openssl x509 -req -in kibana.csr -signkey kibana.key -out kibana.crt
```

> [!NOTE] 注意事项
> 1. 上面的移动，是基于 ES 节点和 Kibana 节点相同，如果不同，自己想办法

---


##### 1.3.5. 为 Kibana 配置服务用户

服务账户专用于 Kibana 与 Elasticsearch 之间的通信，登录 Kibana 时使用的也是这个账户。
```
# 1. 进入 ES 目录
cd /mystudy/es/elasticsearch


# 2. 设置 kibana 密码（自定义密码）
curl -k -u elastic:<elastic 密码> -X POST "https://<随意 ES 节点IP>:9200/_security/user/kibana/_password" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "<新密码>"
}'

curl -k -u elastic:wq666666 -X POST "https://192.168.136.8:9200/_security/user/kibana/_password" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "wq666666"
}'
```

---


##### 1.3.6. 修改 Kibana 主配置文件：config/kibana.yml
Kibana 的连接流程（关键原理）
Kibana 启动时，它做的是：

1. **连接你在 `elasticsearch.hosts` 配置里写的节点**之一；
    
2. 如果连上了，Kibana 会调用该节点的 [**Cluster Info API**](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html)；
    
3. 从集群返回的响应里，它能看到整个集群的节点列表，包括你后来加的新节点；
    
4. 然后 Kibana 后续的请求，就可以分发到整个集群的任意节点（由 Elasticsearch 自己决定）。

所以 **不是每次新增节点都要改 `kibana.yml`**，**只要原来配置的节点还活着**，看似`elasticsearch.hosts: ["https://192.168.136.8:9200", "https://192.168.136.9:9200", "https://192.168.136.10:9200"]`只配置了三个，只要我们连接上一个，就能获得最新的信息，但是如果三个都连接不上就坏事了
```
# 1. 修改配置文件
vim /mystudy/kibana/kibana/config/kibana.yml


# 2. 添加下述内容
# 服务端口
server.port: 5601

# Kibana 服务主机地址
server.host: "192.168.136.8"

# 国际化 - 中文
i18n.locale: "zh-CN"

# Elasticsearch 服务主机地址
elasticsearch.hosts: ["https://192.168.136.8:9200", "https://192.168.136.9:9200", "https://192.168.136.10:9200"]


# 访问 Elasticsearch 服务的账号密码
elasticsearch.username: "kibana"
elasticsearch.password: "wq666666"

# Elasticsearch SSL 设置
elasticsearch.ssl.verificationMode: none
elasticsearch.ssl.certificateAuthorities:
  - "/mystudy/es/elasticsearch/config/certs/ca.crt"

# Kibana 服务 SSL 设置
server.ssl.enabled: true
server.ssl.certificate: "/mystudy/kibana/kibana/config/kibana1.crt"
server.ssl.key: "/mystudy/kibana/kibana/config/kibana1.key"
```

> [!NOTE] 注意事项
> 1. `kibana` 是 Elasticsearch 默认提供的服务账号，专用于 Kibana 与 ES 通信，不应使用具有最高权限的 `elastic` 超管账号进行配置

![](source/_posts/笔记：ElasticSearch%20基础/image-20250501142642453.png)

监听的地址，监听的网卡，与es 的http.host类似，但是没有_local_ 这种那么灵活的

Kibana 会按你 `elasticsearch.hosts` 列表的顺序尝试连接：它会逐一 ping 列表里的地址，直到找到一个能连通的节点才算启动成功。

如果当前连接的挂掉，他可能需要一段时间反应，来连接下一个，只要还有至少一个 ES 实例可用，Kibana 本身就不会中断服务，也不会让你下线或重新登录，当然，如果kibana 本身依赖的 ES 不可用，那 Kibana 也不能用了

例如你Kibana 部署在136.8，虽然你136.8还启动，但是 ES 集群不可用，那你访问 `https://192.168.136.8:5601/spaces/enter` 很大可能进不去，然后我们可能需要启动多台es 服务器，才能重新进去kibana

---


##### 1.3.7. 创建 kibana 用户

该账户不同于前面提到的服务账户。服务账户专用于 Kibana 与 Elasticsearch 之间的通信，登录 Kibana 时使用的也是这个账户。

这里的账户是由于 Kibana 也不允许以 root 用户身份运行，因此我们需要创建一个名为 `kibana` 的用户，并使用该用户进行后续的操作和配置。

但是由于我们的 Elasticsearch 和 Kibana 部署在同一台主机上，因此我们可以直接复用 `es` 用户来启动 Kibana，而不需要单独创建 `kibana` 用户啦

```
# 1. 切换 root 用户
su root


# 2. 修改 Kibana 文件拥有者为 es
chown -R es:es /mystudy/kibana/kibana


# 3. 切换回 es 用户
su es
```

---


##### 1.3.8. 启动 Kibana
```
# 1. 切换 es 用户
su es


# 2. 进入 Kibana 目录
cd /mystudy/kibana/kibana


# 3. 启动 Kibana
# 3.1. 前台启动
bin/kibana

# 3.2. 后台启动
nohup /mystudy/kibana/kibana/bin/kibana >kibana.log 2>&1 &


# 4. 补充：停止后台 Kibana
ps aux | grep kibana

kill -9 123456
```

> [!NOTE] 注意事项
> 1. `Kibana` 并不支持使用 `kibana -d` 进行后台启动

---


##### 1.3.9. 访问 Kibana 节点

访问 Kibana 节点： https://192.168.136.8:5601
![](source/_posts/笔记：ElasticSearch%20基础/image-20250417121921514.png)

---


# 四、补充

### 1. 相关网站

1. ==ElasticSearch 官方地址==：
	1. https://www.elastic.co/cn/
2. ==ElasticSearch 文档==：
	1. https://www.elastic.co/guide/en/elasticsearch/reference/8.18/mapping-params.html
3. ==ElasticSearch 下载地址==：
	2. https://www.elastic.co/cn/downloads/past-releases#elasticsearch
4. ==Kibana 下载地址==：
	1. https://www.elastic.co/cn/downloads/past-releases#kibana
5. ==IK 分词器下载地址==：
	1. https://release.infinilabs.com/analysis-ik/stable/
6. ==拼音分词器下载地址==：
	1. https://release.infinilabs.com/analysis-pinyin/stable/
7. ==拼音分词器文档==：
	1. https://github.com/infinilabs/analysis-pinyin
8. 手都有
	1. https://release.infinilabs.com/

---


### 2. ES 目录结构

```
elasticsearch /
|
|-- bin /                                             # ES 的可执行脚本
|
|-- config /                                          # ES 的配置目录
|
|-- jdk /                                             # 内置 JDK 
|
|-- lib /                                             # ES 依赖的类库
|
|-- logs /                                            # 日志目录
|
|-- modules /                                         # 模块目录
|
|-- plugins /                                         # 插件目录
```

---


### 3. 正排索引


---



| 词条            | ID      |
| ------------- | ------- |
| Elasticsearch | ....... |
| 是             | ....... |
| 一个            | ....... |
| 开源            | ....... |
| 搜索引擎          | ....... |
| 适合            | ....... |
| 处理            | ....... |
| 大规模           | ....... |
| 结构化           | ....... |
| 非结构化          | ....... |
| 数据            | ....... |




### 4. 倒排索引



---


### 5. ES Mapping 属性

Mapping 是对索引库中文档的约束，相当于 MySQL 中列名的属性，常见的 Mapping 属性包括：
1. ==type==：
	1. 字段数据类型
2. ==index==：
	1. 是否创建索引（是否参与搜索），默认为 true
	2. 可以简单理解为：“是否需要被搜索 -> 决定是否需要倒排索引”
3. ==analyzer==：
	1. 使用的分词器
4. ==properties==：
	1. 该字段的子字段

注意：这里只是列举了常见的 Mapping 属性，其他的可以查询 [ES 文档](https://www.elastic.co/guide/en/elasticsearch/reference/8.18/mapping-params.html)

---


### 6. ES 字段数据类型

1. ==字符串==：
	1. <font color="#00b0f0">可分词文本</font>：
		1. text
	2. <font color="#00b0f0">精确值</font>：
		1. keyword
2. ==数值==：
	1. <font color="#00b0f0">整数类型</font>：
		1. <font color="#7030a0">byte</font>：
			- 1 字节，-128 到 127
		2. <font color="#7030a0">short</font>：
			- 2 字节，-32768 到 32767
		3. <font color="#7030a0">integer</font>：
			- 4 字节，-2³¹ 到 2³¹-1
		4. <font color="#7030a0">long</font>：
			- 8 字节，-2⁶³ 到 2⁶³-1
	2. <font color="#00b0f0">浮点数类型</font>：
		1. <font color="#7030a0">half_float</font>：
			- 2字节，3~5 位十进制精度
		2. <font color="#7030a0">float</font>：
			- 4字节，7 位十进制精度
		3. <font color="#7030a0">double</font>：
			- 8字节，15~16 位十进制精度
3. ==布尔==：
	1. boolean
4. ==日期==：
	1. date
5. ==对象==：
	1. object

---




---


### 7. ES 打分算法

ES 5.0 之前选用 TF-IDF 打分算法，但是该算法有个缺点，打分会随着词频的增加而越来越大，在 ES 5.0 之后选用 BM 2.5 算法，虽然也会随着词频增加而增大，但增长曲线会逐渐趋于水平

![](source/_posts/笔记：ElasticSearch%20基础/PixPin_2025-04-27_11-26-54_PhotoGrid.png)

![](source/_posts/笔记：ElasticSearch%20基础/PixPin_2025-04-27_11-22-54_PhotoGrid.png)

> [!NOTE] 注意事项
> 1. 在以后设计的时候也需要注意，参与算分的条件越多，性能就越差


---


### 8. 常见距离单位

1. ==米==：
	1. `m`
2. ==千米==：
	1. `km`
3. ==英里==：
	1. `mi`
4. ==码==：
	1. `yd`

---





# 麻烦


![|500](source/_posts/笔记：ElasticSearch%20基础/image-20250418184544203.png)








![](source/_posts/笔记：ElasticSearch%20基础/image-20250417122657685.png)
![](source/_posts/笔记：ElasticSearch%20基础/image-20250417122752564.png)

![](source/_posts/笔记：ElasticSearch%20基础/image-20250417122922783.png)


---














# 补充，企业级高可用






构建企业级高可用（High Availability, HA）集群时，需要从多个方面进行综合考虑，以确保系统在各种故障场景下仍能提供稳定、可靠的服务。以下是高可用集群设计和部署时需要关注的核心方面：

---

### 1. **1. 架构设计**
- **冗余性**：
  - 确保系统中没有单点故障（Single Point of Failure, SPOF）。例如，RabbitMQ 集群需要多节点，负载均衡器（如 HAProxy）需要通过 Keepalived 实现主备切换。
  - 使用多副本机制（如 RabbitMQ 的队列镜像）保证数据冗余。
- **分布式部署**：
  - 考虑跨机房或跨数据中心部署，以应对区域性故障。
  - 使用 Federation 或 Shovel 插件实现 RabbitMQ 跨地域的数据同步。
- **负载均衡**：
  - 配置负载均衡器（如 HAProxy、Nginx、F5）将请求分发到多个后端节点。
  - 选择合适的负载均衡算法（如轮询、最少连接）以优化性能。
- **可扩展性**：
  - 设计时考虑集群的水平扩展能力，确保新增节点不会影响现有服务。
  - 评估系统在高负载下的性能瓶颈，如网络带宽、磁盘 I/O 等。

---

### 2. **2. 故障转移与恢复**

- **自动故障转移**：
  - 使用 Keepalived 或类似工具实现虚拟 IP（VIP）漂移，确保前端服务的高可用。
  - 配置 RabbitMQ 集群的队列镜像，确保节点故障时数据不丢失。
- **健康检查**：
  - 配置负载均衡器的健康检查机制（如 HAProxy 的 `check` 选项），及时剔除故障节点。
  - 监控 RabbitMQ 节点的运行状态（如通过 `rabbitmqctl cluster_status` 或管理插件）。
- **快速恢复**：
  - 制定故障恢复流程（如节点重启、数据同步）。
  - 测试恢复时间目标（RTO, Recovery Time Objective），确保满足业务需求。
- **备份与灾难恢复**：
  - 定期备份关键数据和配置（如 RabbitMQ 的元数据、队列定义）。
  - 制定灾难恢复计划（DR, Disaster Recovery），并定期演练。

---

### 3. **3. 性能优化**
- **资源分配**：
  - 确保服务器硬件（CPU、内存、磁盘）满足集群需求，特别是在高并发场景下。
  - 优化 RabbitMQ 的内存和磁盘使用（如调整 `vm_memory_high_watermark` 参数）。
- **网络优化**：
  - 使用高带宽、低延迟的网络，减少节点间通信开销。
  - 配置合适的 TCP 参数（如增大连接数、优化超时时间）。
- **队列管理**：
  - 合理设置队列的持久化策略和镜像策略，避免过多副本导致性能下降。
  - 使用 Lazy Queues 减少内存占用，适合大数据量场景。
- **负载均衡优化**：
  - 根据业务特点选择合适的负载均衡策略（如基于会话的粘性连接或动态权重）。
  - 调整 HAProxy 的 `maxconn` 和超时参数，适应高并发场景。

---

### 4. **4. 监控与告警**
- **实时监控**：
  - 部署监控系统（如 Prometheus + Grafana、Zabbix）跟踪集群的运行状态。
  - 监控关键指标：
    - RabbitMQ：队列长度、消息速率、连接数、内存/磁盘使用率。
    - HAProxy：请求速率、后端节点健康状态、响应时间。
    - Keepalived：VIP 状态、主备切换事件。
    - 服务器：CPU、内存、磁盘 I/O、网络流量。
- **告警机制**：
  - 配置告警规则（如队列积压、节点故障、VIP 漂移），通过邮件、短信或企业微信通知运维团队。
  - 设置多级告警阈值（如警告、严重），避免误报或漏报。
- **日志管理**：
  - 收集和分析系统日志（RabbitMQ、HAProxy、Keepalived），使用集中式日志系统（如 ELK Stack、Loki）。
  - 定期审查日志，排查潜在问题。

---

### 5. **5. 安全性**
- **网络安全**：
  - 配置防火墙，仅开放必要端口（如 RabbitMQ 的 5672、15672，HAProxy 的 80/443）。
  - 使用 VPC 或专用网络隔离集群，防止未经授权的访问。
- **数据加密**：
  - 为 RabbitMQ 和 HAProxy 配置 TLS/SSL，加密客户端与服务器的通信。
  - 确保节点间通信（如 Erlang 通信）也启用加密。
- **访问控制**：
  - 配置 RabbitMQ 的用户权限，限制客户端的操作范围（如只允许特定用户访问特定队列）。
  - 使用强密码或集成 LDAP/SSO 进行认证。
- **漏洞管理**：
  - 定期更新 RabbitMQ、HAProxy、Keepalived 和操作系统，修复已知漏洞。
  - 订阅安全公告，及时响应新的威胁。

---

### 6. **6. 数据一致性与可靠性**
- **数据冗余**：
  - 配置 RabbitMQ 的队列镜像（Mirrored Queues），确保数据在多个节点上有副本。
  - 评估镜像策略（如 `ha-mode: all` 或 `ha-mode: exactly`），平衡可靠性和性能。
- **消息持久化**：
  - 启用队列和消息的持久化（`durable: true`, `delivery_mode: 2`），确保消息在故障后不丢失。
  - 注意持久化对性能的影响，必要时优化磁盘 I/O。
- **事务与确认机制**：
  - 使用 Publisher Confirms 或 Consumer Acknowledgements 确保消息可靠投递。
  - 根据业务需求权衡性能和可靠性（如批量确认 vs 逐条确认）。

---

### 7. **7. 运维管理**
- **自动化运维**：
  - 使用 Ansible、Terraform 或 Kubernetes 自动化集群的部署和配置。
  - 配置 CI/CD 流程，简化版本升级和补丁应用。
- **容量规划**：
  - 定期评估集群的负载和增长趋势，提前规划扩容。
  - 测试集群在峰值负载下的表现，确保满足 SLA（Service Level Agreement）。
- **文档与培训**：
  - 维护详细的架构文档、运维手册和故障处理流程。
  - 定期培训运维团队，确保熟悉集群的维护和应急响应。

---

### 8. **8. 测试与验证**
- **高可用性测试**：
  - 模拟各种故障场景（如节点宕机、网络分区、VIP 切换），验证故障转移是否正常。
  - 测试数据一致性，确保故障后无消息丢失或重复。
- **性能测试**：
  - 使用工具（如 JMeter、PerfKit）模拟高并发场景，评估集群的吞吐量和响应时间。
  - 识别瓶颈（如网络、磁盘、队列配置）并优化。
- **灾难恢复演练**：
  - 定期进行灾难恢复演练，验证备份和恢复流程的有效性。
  - 记录演练结果，优化恢复时间和流程。

---

### 9. **9. 成本与资源管理**
- **资源成本**：
  - 评估硬件、云服务和 лицен费用，优化资源分配（如选择合适的实例类型）。
  - 使用自动化伸缩（如果在云端部署）降低闲置资源成本。
- **运维成本**：
  - 通过自动化和监控减少人工干预，降低运维成本。
  - 选择开源工具（如 HAProxy、Keepalived）或性价比高的商业解决方案。

---

### 10. **10. 业务需求与 SLA**
- **可用性目标**：
  - 根据业务需求定义可用性目标（如 99.9% 或 99.99% 的 uptime）。
  - 计算允许的宕机时间，确保架构设计满足要求。
- **延迟与吞吐量**：
  - 明确业务对消息延迟和吞吐量的要求，优化配置以满足这些指标。
- **数据保留策略**：
  - 根据业务需求设置消息的保留时间和队列的清理策略（如 TTL、最大长度）。
- **合规性**：
  - 确保集群满足行业法规（如 GDPR、HIPAA），特别是在数据隐私和审计方面。

---

### 11. **总结**
高可用集群的设计和维护需要从架构、故障转移、性能、监控、安全、数据可靠性、运维、测试、成本和业务需求等多个维度综合考虑。对于 RabbitMQ 集群搭配 HAProxy + Keepalived 的场景，核心是确保无单点故障、数据冗余和快速恢复，同时通过监控和自动化运维降低风险。建议根据具体业务场景（如消息量、延迟要求、预算）进一步细化配置，并定期进行测试和优化。

如果你有某个方面需要深入探讨（例如具体配置、工具选型或测试方法），请告诉我！



































































![](source/_posts/笔记：ElasticSearch%20基础/image-20250416192542363.png)




![](source/_posts/笔记：ElasticSearch%20基础/image-20250416192834497.png)

有 ES 脑裂问题
![](source/_posts/笔记：ElasticSearch%20基础/image-20250416194752833.png)

偶数的话，可能出现问题和脑裂

---
