# Elasticsearch Query DSL

[ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

## Intro

Elasticsearch提供了一套基于JSON的Query DSL用于实现文档检索。其语法树设计可以分成两种语句

- Leaf query clause: 叶检索语句通过针对指定的字段检索出文档。
- Compound query clause: 复合检索语句通过包装叶检索或者其他复合检索语句实现特殊的逻辑。

同时检索语句根据行为不同可以区分为两种上下文类型query context/filter context；elasticsearch一般通过相关性分数来评估检索复合度，并倒序排列之后给出最佳的检索结果，该分数会返回在检索结果的_score字段中。

- query context: 如果一个检索语句是用于回答被检索的文档有多匹配检索要求，则该语句工作在query context；检索语句会给出文档的相关性分数，并参与最终的相关性分数计算。
- filter context: 如果一个检索语句是用于回答被检索文档是否符合检索条件，则该语句工作在filter context；检索语句过滤掉不符合要求的文档，并不参与相关性分数的计算；同时频繁使用的filter，会被ES自动缓存，以提高过滤性能。

query context中需要计算相关性分数以及filter context缓存加持，filter context往往性能高于同样语义的query context；在不需要相关性排序的情况下，应该优先考虑使用filter context。

## Compound query

Compound query通过封装其他Compound query以及leaf query，实现组合检索结果、调整检索行为、切换context等作用。

### bool

#### basic

bool复合语句用于实现子语句的bool逻辑的关联，其直接映射于Lucene的BooleanQuery。子语句包含以下几种类型：

|关联类型|描述|
|--|--|
|must|查询结果必须满足该语句下的query，工作于query context|
|filter|查询结果必须满足该语句下的query，工作于filter context，可能被缓存，不影响最终分数|
|should|查询结果应该满足该语句下的query，行为可以通过参数mimimum_should_match调整，工作于query context|
|must_not|查询结果不应该满足该语句下的query，工作于filter context，可能被缓存，不影响最终分数|

bool关系遵循*more-matches-is-better*规则，会将所有出现在must与should语句的分数之和作为每个文档的最终分数 _score，并默认按该分数倒序排列给出结果。

#### extra

- minimum_should_match

通过添加minimum_should_match可以指定should语句中，需要满足的最少query数。默认情况下，在没有must以及filter的情况下，该参数默认为1，即结果文档至少满足其中一个；在存在must以及filter的情况下，默认为0，即结果文档可以全不满足。

- bool.filter scoring

filter的query对最终文档分数没有影响，在没有match以及should语句的情况下，所有结果分数为0。如果需要实现给结果文档赋分数，可以使用在must中加入match_all，则对结果文档集没有影响，并所有文档均拥有1.0分。此外，也可以不用bool，而通过constant_score给所有结果文档赋固定分数。

- boost

通过添加boost参数，可以使结果集分数放大或者缩小一定倍数，从而实现权重控制。

#### example

1. basic query

    如下查询语句实现效果为，查询发生在周一(filter)、姓氏为*Underwood*、姓名最好为*Eddie*并且不位于亚洲的交易记录。通过该查询，*Eddie Underwood*在周一的交易记录如果存在的话，会赋予高分数，并优先返回。

        POST /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "bool": {
              "filter": [
                {
                  "term": {
                    "day_of_week": "Monday"
                  }
                }
              ],
              "must": [
                {
                  "term": {
                    "customer_last_name.keyword": {
                      "value": "Underwood"
                    }
                  }
                }
              ],
              "should": [
                {
                  "term": {
                    "customer_first_name.keyword": {
                      "value": "Eddie"
                    }
                  }
                }
              ],
              "must_not": [
                {
                  "term": {
                    "geoip.continent_name": {
                      "value": "Asia"
                    }
                  }
                }
              ]
            }
          }
        }

2. use minimum_should_match

    如下查询语句指定了*minimum_should_match*，should中列举的query至少要满足2个，才能出现在结果集中。

        POST /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "bool": {
              "should": [
                {
                  "match": {
                    "customer_first_name": "Eddie"
                  }
                },
                {
                  "match": {
                    "customer_last_name": "Underwood"
                  }
                },
                {
                  "match": {
                    "customer_full_name": "Weber"
                  }
                }
              ],
              "minimum_should_match": 2
            }
          }
        }

3. bool.filter scoring

    如下语句通过filter过滤文档，同时must中通过使用must+match_all（该语句不对结果集产生影响）给所有文档赋上1.0分，再通过*boost*参数使所有文档分数乘以5。此外，must_not也可以使用一样的方式，使之拥有分数。

        POST /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "bool": {
              "boost": 5,
              "must": [
                {
                  "match_all": {}
                }
              ],
              "filter": [
                {
                  "term": {
                    "day_of_week": "Monday"
                  }
                }
              ]
            }
          }
        }

### boosting

### constant_score

### dis_max

### function_score
