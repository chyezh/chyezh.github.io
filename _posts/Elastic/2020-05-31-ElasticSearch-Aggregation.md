# Elasticsearch Aggreagation

[ref](https://www.elastic.co/guide/en/elasticsearch/reference/7.7/search-aggregations.html)

## Intro

### ES的聚合计算特点

ES提供了丰富的聚合运算，聚合运算的作用文档对象取决于运算过程的上下文，比如顶级的聚合运算会作用于 query/filter之后。
聚合运算可以分为以下几类：

- Bucketing：即对文档进行分类。
- Metric：即对一批文档获取一个统计指标。
- Matrix：通过多个字段进行的矩阵运算。
- Pipeline：对聚合结果进一步聚合。

同时由于*Bucketing*的存在，聚合运算可以只作用于某些桶，因此聚合运算可以实现嵌套。

### ES的聚合运算结构

    "aggregations" : { // 运算标识
        "<aggregation_name>" : { // 运算名
            "<aggregation_type>" : { // 运算类型
                <aggregation_body> // 运算体
            }
            [,"meta" : {  [<meta_data_body>] } ]? //
            [,"aggregations" : { [<sub_aggregation>]+ } ]? // 子运算
        }
        [,"<aggregation_name_2>" : { ... } ]*
    }

## ES聚合运算类型

### Metrics Aggregations

| 运算符 | 操作 | 备注 |
| :------: | :----: | :----: |
|avg| 平均数||
|weighted_avg|加权平均数||
|boxplot|箱型图，获取最大最小以及4等分点对应值|使用了TDigest算法，精度通过compression参数控制|
|cardinality|基数统计|基于hyperloglog++算法，precision_threshold控制进行模糊计算的下限阈值；具备高基数特征的字符串变量，可以通过预计算哈希映射到单独字段来实现高效率基数统计|
|stats|常规统计指标获取，含min max avg sum||
|extended_stats|拓展统计指标获取，相比stats，增加了sum_of_squares variance std_deviation三种指标|算子同时会返回标准差范围std_deviation_bounds，通过指定参数sigma，可以决定范围缩放大小|
|geo_bounds|针对type geo_point获取经纬度范围||
|geo_centroid|针对type geo_point获取地理几何中心||
|max|获取最大值||
|min|获取最小值||
|sum|获取和||
|median_abosolute_deviation|获取绝对差中位数|使用TDigest算法估计，精度通过compression参数控制|
|percentiles|在正排序下，获取大于百分数的文档数量的对应数值，可以用于获取分布曲线|使用percents参数指定百分位数列表；聚合结果通常是统计值，使用了TDigest算法，百分数越极端，集合越小，结果一般更精准；结果精准程度取决于使用内存大小与原数据的分布情况，可以通过tdigest.compression参数控制|
|percentile_ranks|在正排序下，获取低于特定值文档数量在文档总数中的百分比|参考运算perentiles|
|scripted_mertric|通过script实现map-reduce|init_script初始化script运行环境；map_script对各分片的文档分别进行映射；combine_script将各分片结果整合后存储到states变量中；reduce_script将states计算后返回聚合结果|
|string_stats|对keyword类型，获取非空计数 count、最长最短平均长度 max_length min_length avg_length，香农熵 entropy|计算香农熵基于各字符出现概率，使用show_distribution可以直接显示字符概率分布|
|top_hits|可以对bucketing聚合结果实现TopN运算||
|top_metrics|类似top_hits，但是一般比top_hits更快并使用更少的内存|作为子嵌套聚合时，可以参与外层聚合排序运算|
|value_count|对于进入聚合运算的文档进行计数||

### Bucket Aggregations

|运算符|操作|备注|
|:---:|:---:|:---:|
|filter|指定单桶进行聚合||
|filters|指定多个桶进行聚合||
|global|指定对整个索引进行聚合，使索引不受搜索语句影响||
|missing|单桶，聚合字段丢失的部分||
|terms|对指定字段进行离散值分桶|一般来说总是按计数的倒序返回size个数的桶，但由于各个分片各自排序，导致最后部分数据不够精确；可以指定order来决定排序作用字段【含子聚合产生的字段】，但是注意升序排列会使由于分片引入误差的概率加大；通过设置collect_mode，可以在bfs或者dfs之间切换嵌套聚合的模式|
|rare_terms|对指定字段的稀有值进行分桶聚合|可以解决terms在计数升序排列时，由分片引入的误差|
|significant_terms|获取前景集相对背景集，显著关联的term的聚合结果|背景集为检索操作前的检索集合，前景集为检索操作后的结果，如果term在前景集的出现概率显著高于背景集，这个term将作为桶并进行聚合；返回值中的bg_*则为在背景集的聚合结果；由于该聚合运算消耗资源大，并且对长尾低质匹配结果不敏感，通常可以作为sampler的子聚合|
|significant_text|作用类似于significant_terms，不过是专门针对text类型进行聚合的；|filter_duplicate_text参数可以过滤掉如人物介绍，typo等大段落复制导致污染显著聚合结果的桶，十分适合新闻、论文等领域分析。|
|auto_date_histogram|通过指定分桶数量，ES自动适配时间间隔来进行聚合计数||
|date_histogram|针对时间字段，指定固定时间间隔进行分桶聚合||
|histogram|通过指定值间隔实现直方图聚合||
|range|通过指定多个值间隔进行分桶聚合||
|date_range|可以指定多个前闭后开时间区间，分别对每个时间区间进行计数聚合|和range不同点在于，专门针对时间类型，添加时间表达式|
|ip_range|可以指定多个ip区间进行聚合||
|geo_distance|针对地理坐标类型，通过指定range和出发点按距离进行聚合||
|sampler|用于在子嵌套聚合前过滤出高分文档，实现抽样|一般用于长尾匹配质量较低的子聚合前、子聚合结果可用的情况下，减少子聚合耗时、高分抽样结果能使子聚合产出更好的预期结果，如子聚合significant_terms|
|diversified_sampler|作用类似于sampler，但sampler抽样时关心评分高低；可以通过指定字段，来限制该字段的最高重复数量，从而实现抽样多元化|使用场景类似sampler，通过设置max_docs_per_value来改变最高重复数|
|children|针对需要聚合包含join类型的父文档，通过指定子文档的标记type来统计子文档数量||
|adjacency_matrix|通过给定的filters决定的多个集合，获取各个集合直接的邻接矩阵，权值为集合共有的文档个数|可以通过指定separator参数来指定集合名分隔符|
|composite|通过指定source字段，产生由多个bucket复合成的key，并以该key作为新的bucket|可以通过指定after参数实现分页|

- range类型可能会导致意外的聚合结果，在histogram聚合中，对range字段聚合时，文档会落到每个range值和桶的range重合的桶中；在使用边界过滤range后并进行聚合的时候，使用histogram也会出现超越过滤边界的聚合结果，原因是只有range完全落在边界外时文档才会被过滤，否则将参与聚合，导致histogram出现超越边界的桶。

#### Example

    // adjacency_matrix
    POST /kibana_sample_data_logs/_search
    {
      "size" : 0,
      "aggs": {
        "tags": {
          "adjacency_matrix": {
            "separator":"#",
            "filters": {
              "success": {
                "term": {
                  "tags": "success"
                }
              },
              "security":{
                "term":{
                  "tags":"security"
                }
              },
              "login": {
                "term": {
                  "tags":"login"
                }
              },
              "warning": {
                "terms": {"tags": ["warning"]}
              },
              "error": {
                "terms": {"tags": ["error"]}
              },
              "info": {
                "terms": {"tags": ["info"]}
              }
            }
          }
        }
      }
    }

    // auto_date_histogram
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "log_over_time": {
          "auto_date_histogram":{
            "field": "timestamp",
            "format": "yyyy-MM-dd HH:mm:ss",
            "time_zone": "Asia/Shanghai"
          }
        }
      }
    }

    // child

    PUT child_example
    {
      "mappings": {
        "properties": {
          "join": {
            "type" : "join",
            "relations": {
              "question" : "answer"
            }
          }
        }
      }
    }
    PUT child_example/_doc/1
    {
      "join": {
        "name": "question"
      },
      "body": "<p>I have Windows 2003 server and i bought a new Windows 2008 server...",
      "title": "Whats the best way to file transfer my site from server to a newer one?",
      "tags": [
        "windows-server-2003",
        "windows-server-2008",
        "file-transfer"
      ]
    }
    PUT child_example/_doc/2?routing=1
    {
      "join": {
        "name": "answer",
        "parent": "1"
      },
      "owner": {
        "location": "Norfolk, United Kingdom",
        "display_name": "Sam",
        "id": 48
      },
      "body": "<p>Unfortunately you're pretty much limited to FTP...",
      "creation_date": "2009-05-04T13:45:37.030"
    }
    PUT child_example/_doc/3?routing=1&refresh
    {
      "join": {
        "name": "answer",
        "parent": "1"
      },
      "owner": {
        "location": "Norfolk, United Kingdom",
        "display_name": "Troll",
        "id": 49
      },
      "body": "<p>Use Linux...",
      "creation_date": "2009-05-05T13:45:37.030"
    }
    POST child_example/_search
    {
      "size": 0,
      "aggs": {
        "top_tags": {
          "terms": {
            "field": "tags.keyword",
            "size": 10
          },
          "aggs": {
            "to_answers": {
              "children": {
                "type": "answer"
              },
              "aggs": {
                "top_names": {
                  "terms": {
                    "field": "owner.display_name.keyword",
                    "size": 10
                  }
                }
              }
            }
          }
        }
      }
    }

    // composite
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "new_key": {
          "composite": {
            "after": {
              "clientip_new": "0.207.229.147",
              "event_new": "sample_web_logs"
            },
            "size": 2,
            "sources": [
              {
                "clientip_new": {
                  "terms": {
                    "field": "clientip"
                  }
                }
              },
              {
                "event_new": {
                  "terms": {
                    "field": "event.dataset"
                  }
                }
              }
            ]
          }
        }
      }
    }

    // date_histogram
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "by_day": {
          "date_histogram": {
            "field": "timestamp",
            "calendar_interval": "day",
            "offset": "+6h",
            "time_zone": "Asia/Shanghai",
            "keyed": true,
            "format": "yyyy-MM-dd"
          }
        }
      }
    }

    // date_range
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "range": {
          "date_range": {
            "field": "timestamp",
            "format": "dd-MM-yyyy",
            "ranges": [
              {
                "from": "now-10d",
                "to": "now-9d"
              },
              {
                "from": "now-10d"
              }
            ]
          }
        }
      }
    }

    // sampler
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "query": {
        "query_string": {
          "query": "tags:warning"
        }
      },
      "aggs": {
        "sample": {
          "sampler": {
            "shard_size": 200
          },
          "aggs":{
            "keywords": {
              "significant_terms": {
                "field": "tags.keyword",
                "exclude": ["warning"]
              }
            }
          }
        }
      }
    }

    // diversified_sampler
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "query": {
        "query_string": {
          "query": "tags:warning"
        }
      },
      "aggs": {
        "sample": {
          "diversified_sampler": {
            "shard_size": 200,
            "field": "ip",
            "max_docs_per_value": 1
          },
          "aggs":{
            "keywords": {
              "significant_terms": {
                "field": "tags.keyword",
                "exclude": ["warning"]
              }
            }
          }
        }
      }
    }

    // filter
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "response_200": {
          "filter": {
            "term": {
              "response": 200
            }
          }
        }
      }
    }

    // filters
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "messages": {
          "filters": {
            "filters": {
              "Mozilla" : {
                "match": {
                  "message": "Mozilla"
                }
              },
              "Chrome": {
                "match": {
                  "message": "Chrome"
                }
              }
            }
          }
        }
      }
    }

    // geo_distance
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs" : {
        "distance": {
          "geo_distance": {
            "keyed" : true,
            "field": "geo.coordinates",
            "origin": {
              "lat": 52.376,
              "lon": 4.894
            },
            "ranges": [
              {
                "from": 100,
                "to": 300
              },
              {
                "from": 10000
              }
            ]
          }
        }
      }
    }

    // global
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "query": {
        "term": {
          "tags": {
            "value": "info"
          }
        }
      },
      "aggs": {
        "total_count": {
          "global": {},
          "aggs": {
            "val": {
              "value_count": {
                "field": "tags.keyword"
              }
            }
          }
        },
        "info_count": {
          "value_count": {
            "field": "tags.keyword"
          }
        }
      }
    }

    // missing
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "missing_count": {
          "missing": {
            "field": "message.keyword"
          }
        }
      }
    }

    // histogram
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "bytes_histogram": {
          "histogram": {
            "field": "bytes",
            "interval": 50,
            "min_doc_count": 0,
            "extended_bounds": {
              "min": 0,
              "max": 30000
            },
            "keyed": true
          }
        }
      }
    }

    // ip_range
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "ips": {
          "ip_range": {
            "field": "clientip",
            "ranges": [
              {
                "to": "127.0.0.1"
              },
              {
                "from": "10.0.0.5",
                "to": "10.0.0.10"
              },
              {
                "from": "255.0.0.0"
              }
            ]
          }
        }
      }
    }

    // range
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "bytes_bucket": {
          "range": {
            "field": "bytes",
            "ranges": [
              {
                "to": 50
              },
              {
                "from": 50,
                "to": 100
              },
              {
                "from": 200
              }
            ]
          }
        }
      }
    }

    // terms
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "aggs": {
        "machines": {
          "terms": {
            "field": "machine.os.keyword",
            "size": 10,
            "order": {
              "max_bytes": "desc"
            }
          },
          "aggs": {
            "max_bytes": {
              "max": {
                "field": "bytes"
              }
            }
          }
        }
      }
    }

    // rare_terms
    POST /kibana_sample_data_logs/_search
    {
      "size": 1,
      "aggs": {
        "machines": {
          "rare_terms": {
            "field": "machine.os.keyword",
                    "max_doc_count": 1
          }
        }
      }
    }

    // significant_terms
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "query": {
        "match": {
          "url": "windows-x86_64.zip"
        }
      },
      "aggs": {
        "significat_os": {
          "significant_terms": {
            "field": "machine.os.keyword"
          }
        }
      }
    }

    // significant_text
    POST /kibana_sample_data_logs/_search
    {
      "size": 0,
      "query": {
        "match": {
          "url": "windows-x86_64.zip"
        }
      },
      "aggs": {
        "significat_message_word": {
          "significant_text": {
            "field": "message",
            "filter_duplicate_text": "true"
          }
        }
      }
    }
