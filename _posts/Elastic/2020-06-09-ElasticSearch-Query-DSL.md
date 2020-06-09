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



### boosting

### constant_score

### dis_max

### function_score
