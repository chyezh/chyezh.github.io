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
|top_metrics|类似top_hits，没大看明白使用场景|
|value_count|对于进入聚合运算的文档进行计数||
