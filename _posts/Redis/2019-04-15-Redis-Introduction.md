# Redis Introduction

[ref](https://redis.io/topics/data-types-intro)

*redis*与其说是一个*k-v*数据库，不如说是一个数据结构存储服务(*data structures server*)，并且支持如下表所示的丰富的数据结构

- 二进制安全的*string*
- 保持插入顺序的*linked lists*
- 无序的*set*
- 依照*score*排序，支持*range*操作的*sorted set*
- *hash map*
- *bitmap*
- *hyperLogLogs*
- *streams*

## Redis keys

*redis*的*key*是二进制安全的，因此可以使用一个*string*，甚至是文件内容做*key*，*empty string*也是合法的*key*。
但是*key*选取一般有以下规则：

- 不要使用过长的*key*。不仅是存储上的浪费，*key*常需要执行比较操作，过长的*key*带来垃圾的性能。可以使用一些*hash*算法来缩小*key*的大小。
- 不要使用过短的*key*，尽量保证*key*的可读性。
- 使得*key*具备某种模式。
- 最大的*key*可以达到*512MB*。
