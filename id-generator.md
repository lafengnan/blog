# 统一ID服务

@2015.12.18



## 背景

目前系统中有多个涉及理财产品与支付交易的环节，每个环节都需要使用唯一的流水号来标识事务的惟一性与可识别性。

统一ID服务就是用以生成系统中需要的多种唯一ID，为产品日终和支付交易提供流水号服务。

当前已知的流水号有：

- 与第三方支付机构对账的交易流水号
- 与各理财机构清算的产品流水号(订单号)

## 原则

目前第三方支付机构已经确认为金运通，其交易标识符要求必须以商户号开头，金运通建议格式为：

- 商户号(12位) + 时间戳YYYYMMDDHHMMSS (14位) + 流水号(6位)
- 批量交易时，流水号作为批次号且长度在测试环境不能超过26位，产品环境不能超过32位。

为了简化设计、统一规范，与各个理财机构之间清算使用的产品流水号也尽量遵循交易流水号规则。

## 设计方案 

因为统一ID服务会用于整个后台系统的各个模块中，因此必须满足下列几个基本需求：

1. 全局惟一性
2. 带有商户信息
3. 带有时间信息
4. 长度不超过26位
5. 满足可用性
6. 高QPS性能

其中条件2，3，4决定了ID格式；条件1要求ID的生成方式必须避免分布式环境中的ID冲突问题。条件5和条件6要求系统有足够的鲁棒性。

- 条件1要求生成的ID必须具有全局一致性，而当前系统架构中全局一致性只能在共享缓冲系统和数据库存储系统中实现。否则就需要引入外部一致性协调系统产生全局唯一ID。
- 条件2的商户信息可以由具体调用ID生成器的服务携带过来，无需集成在统一ID生成规则中。
- 条件3可以通过获取当前时间戳实现。
- 条件4由具体的服务携带的商家号 + 8位日期信息(YYYYMMDD) + 8位唯一流水号(00,000,000 - 99,000,000)组成，其中日期信息中的年份YYYY前两位作为保留位使用。
- 条件5可以通过分布式系统实现，满足一定的HA。
- 条件6可以通过采用Redis作为ID生成和分发点，提高响应速度。



## 方案比较

### Twitter Snowflake

ID一共63位，最高位0。格式（高位到低位）：

- 41bits：时间戳(毫秒级)
- 10bits：Worker号
- 12bits：序列号

采用Zookeeper作ID请求协调中心，保证系统中的分片号惟一性。

### Boundary flake

该方案扩展了Snowflake，引入如下变动：

- ID长度扩展到128位：64位时间戳 + 48位Worker号(和mac地址同长）+ 16位序列号
- 基于Erlang实现
- 采用Mac地址区分惟一性，从而避免引入Zookeeper依赖

### Simple flake

该方案取消了Worker号，保留了41位时间戳，同时把序列号扩展位22位，这样也导致Simple flake在移除Zookeeper依赖的同时引入了新的问题：

- 序列号完全随机化，可能出现重复ID

### Instagram方案

Instagram采用PostgreSQL通过auto-increment 来生成序号，整个ID的结构为：

- 41 bits: Timestamp (毫秒)
- 13 bits: 每个 logic Shard 的代号 (最大支持 8 x 1024 个 logic Shards)
- 10 bits: sequence number; 每个 Shard 每毫秒最多可以生成 1024 个 ID

### Redis方案

上述几种方案基本上是通过Zookeeper或者数据库来完成统一ID的分配，为了简化系统设计，尽量不引入新的第三方系统，因此决定采用系统中的Redis作为统一ID的生成和分发位置。

目前系统中Redis集群采用 1 Master + 1 Slave模式，不存在不同的Redis Instance上产生冲突ID的问题。但是后期，系统需要水平扩展Redis集群，这样会出现数据分片而导致一致性问题。因此统一的ID生成规则中需要引入分片索引号。

$$ID = DATE(8位) + ShardID(2位) + SerialOrder(6位)$$

采用sharding之后，流水号实际采用了分段计算。因此每天可用的流水号即为：

> [00] 000,000 — 999,999
>
> [01] 000,000 — 999,999
>
> …
>
> [99] 000,000 — 999,999

即，可用流水号总量为$$100 \times 1000,000$$，共一亿个流水号，每天00:00:00之后流水号清空置零。

实际使用的流水号建议格式为：

商户号(12位） + 时间戳 (YYYYMMDD, 8位) + 分片号(2位) + 流水号(6位)，其中商户号由各个service调用者依据具体业务配置。对账类型可以作为Redis中的ID Key使用。

![id-generator](/resources/id-generator.png)

其中，ID的生成采用在redis服务器中执行lua脚本的方式实现:

```lua
--
-- Created by IntelliJ IDEA.
-- User: chris
-- Date: 15/12/21
-- Time: 下午4:02
-- To change this template use File | Settings | File Templates.
--
local prefix = '__UtilIdGenerator__';
local totalPartitions = 1024; -- Upto 1024 partitions
local step = 1; -- Only one node now, step set 1
local startStep = 0; -- Only one node, starts from 0

local tag = KEYS[1];
--local toEndInterval = ARGV[1]; -- how long will expire
local dailyReset = KEYS[2];
local interval = tonumber(KEYS[3]);

local partion;
if KEYS[4] == nil then
    partion = 0;
else
    partion = tonumber(KEYS[4]) % totalPartitions;
end


local id_key = prefix .. ':' .. tag;
local count;
count = tonumber(redis.call('INCRBY', id_key, step));
if dailyReset == 'true' and count == startStep + step then
    redis.call('EXPIRE', id_key, interval); -- Expires after midnight
end
local now = redis.call('TIME');
-- second, milliSecond, partition, seq
return {tonumber(now[1]), tonumber(now[2]), partion, count};

```





## 引用

1. [分布式系统中的统一ID服务](http://darktea.github.io/notes/2013/12/08/Unique-ID?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
2. https://github.com/hengyunabc/redis-id-generator