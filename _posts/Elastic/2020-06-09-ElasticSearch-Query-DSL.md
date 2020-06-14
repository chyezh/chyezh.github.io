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

|子语句类型|描述|
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

filter的query对最终文档分数没有影响，在没有match以及should语句的情况下，所有结果分数为0。如果需要实现给结果文档赋分数，可以使用在must中加入match_all，则对结果文档集没有影响，并所有文档均拥有1.0分。此外，也可以不用bool，而通过constant_score给所有结果文档赋予固定分数。

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

#### basic

boosting语句可以检索出满足positive query，同时抑制结果集中满足negative query的相关分。与bool.must_not不同，该语句不剔除满足negative query的文档，只是降低其分数。

|子语句类型|描述|
|--|--|
|postive|满足该语句的文档将会出现在结果集中，并获得对应query决定的分数|
|negative|满足该语句的文档的最终分数=postive的分数*降级比例，该比例由negative_boost参数决定。如果negative_boost大于1，则为升级行为，应该考虑通过使用bool.should实现。|

#### example

1. common usage

    在该查询中，full_name匹配出Underwood的文档会被返回，first_name为Eddie的文档相关分会被乘以0.5，从而实现降级。

        POST /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "boosting": {
              "positive": {
                "match": {
                  "customer_full_name": "Underwood"
                }
              },
              "negative": {
                "term": {
                  "customer_first_name.keyword": {
                    "value": "Eddie"
                  }
                }
              },
              "negative_boost": 3
            }
          }
        }
}

### constant_score

#### basic

该语句可以封装一个filter query，并给filter的结果集一个通过*boost*参数指定的分数。

#### example

1. common usage

该查询获取first_name为Eddie的文档，并给所有文档赋予1.2分。

        GET /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "constant_score": {
              "filter": {
                "term": {
                  "customer_first_name.keyword": "Eddie"
                }
              },
              "boost": 1.2
            }
          }
        }

### dis_max

#### basic

该语句会返回符合所封装的若干个query要求的文档集合。如果一个文档满足多个query，则其最终分数为各个query给出的最大分+剩余分之和乘以系数*tie_breaker*

    final_score = max(scores)+tie_breaker*sum(rest(scores))。

|子语句类型|描述|
|--|--|
|queries|一个被封装query的数组，结果文档集合必然满足其中一个或者多个query，文档满足query的结果分会按照上述规则参与最终分数的运算|

#### example

1. common usage

        GET /kibana_sample_data_ecommerce/_search
        {
          "query": {
            "dis_max": {
              "tie_breaker": 0.7,
              "boost": 1.2,
              "queries": [
                {
                  "term": {
                    "customer_first_name.keyword": {
                      "value": "Eddie"
                    }
                  }
                },
                {
                  "term": {
                    "day_of_week": {
                      "value": "Monday"
                    }
                  }
                },
                {
                  "term": {
                    "customer_last_name.keyword": {
                      "value": "Underwood"
                    }
                  }
                }
              ]
            }
          }
        }

### function_score

#### basic

function_score允许为一个query自定义一个相关分计算函数。

1. 关键参数列表


    |关键参数|描述|
    |--|--|
    |query|指定用于检索的query，用于产出最终返回的结果文档集合，工作在query context下，会给出相应的相关分|
    |functions or single-function|针对每个结果集中的文档，产出对应的自定义分数。单函数可以直接使用函数类型做字段名产出对应规则的分数；而*functions*中不仅可以关联多个函数，同时还可以指定*filter*实现只对满足过滤规则的文档给出分数，见[表2](#table2)|
    |score_mode|用于指定*functions*中各个函数给出分数的组合模式|
    |boost_mode|用于指定*functions*与*query*两者分数的组合模式|
    |boost|整个query的权重系数|
    |max_boost|设置相关分上限，大于该分数的文档将被直接取该数值为相关分|
    |min_score|设置相关分过滤下限，少于该分数的文档将直接从结果集中过滤|

2. <span id="table2">*functions*函数参数列表</span>

    |关键参数|描述|
    |--|--|
    |filter|给定filter，过滤出需要赋值的文档；当未指定时，相当于应用了*match_all*|
    |function|对filter结果赋予分数的函数模型，详情见[表3](#table3)|
    |weight|该参数可以用于给function所得分数进行调权；当未指定function时，相当于function为*weight*|

3. <span id="table3">函数类型列表</span>

    |函数类型|描述|
    |--|--|
    |script_score|对文档的数值字段应用一段脚本，计算得到分数；_score参数可以用于获取被封装的query给出的分数|
    |weight|直接给出*weight*指定的分数|
    |random|给出一个范围[0, 1)的均与分布的随机数作为分数|
    |field_value_factor|作用与script_score类似，将计算过程模型化；当作用于多值字段时，只会使用第一个数值进行计算|
    |decay functions|通过指定数值字段、原点、衰减函数模型、衰减系数、衰减尺度、不变平台的偏移量实现数值到分数的映射。衰减尺度使指达到衰减系数的距离原点偏移量与不变平台偏移量之差。|
