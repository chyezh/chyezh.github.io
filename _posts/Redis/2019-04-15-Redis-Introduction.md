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

## Redis Strings

*redis*通常还是作为*k-v*数据库使用，
*string*是*redis*最简单的数据类型。

*set*和*get*指令分别实现数据的赋值和查询

    set key value [ex seconds] [px milliseconds] [nx|xx]
    get key

其中，*NX|XX*代表设置值的时候是否进行覆盖。
此外，如果*string*符合*redis*的基础数值类型，可以附加其他特殊操作。
比如*string*符合整型数值，*incr*和*decr*实现原子性的自增和自减，*incrby*和*decrby*实现原子性的增减数值。

    incr key
    incrby key increment
    decr key
    decrby key decrement

此外，*get*和*set*也有变种，*mget*和*mset*查询和设置多个值，*getset*获取旧值并设置新值。

    getset key value
    mset key value [key value...]
    mget key [key...]

还有一些不约束类型的操作的键空间命令*key-space-command*，比如*exists*查询多个*key*是否被设置，*del*删除多个*key*，*type*查询*key*对应的*val*类型。

    exists key [key...]
    del key [key...]
    type key [key...]

同时*redis*提供了*expire*，用于给数据设置生命长度。*expire*设置了数据可以生存多少秒，*expireat*设置了数据可以生存到某个时间点，*set*提供了*ex*字段，直接设置生命周期，*persist*取消过期，*ttl*查看剩余时间。

此外，以增加*p*前缀可以转换时间单位为*ms*，如*pexpire*，*pexpireat*，*pttl*以及*set*的*px*字段。

    expire key seconds
    expireat key timestamp
    persist key
    ttl key
