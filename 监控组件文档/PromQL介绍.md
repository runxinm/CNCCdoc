prometheus从根本上存储的所有数据都是时间序列: 具有时间戳的数据流只属于单个度量指标和该度量指标下的多个标签维度。除了存储时间序列数据外，Prometheus也可以利用查询表达式存储5分钟的返回结果中的时间序列数据。

# Prometheus查询

Prometheus提供一个函数式的表达式语言，可以使用户实时地查找和聚合时间序列数据。表达式计算结果可以在图表中展示，也可以在Prometheus表达式浏览器中以表格形式展示，或者作为数据源, 以HTTP API的方式提供给外部系统使用。

# 1. 表达式语言数据类型

在Prometheus的表达式语言中，任何表达式或者子表达式都可以归为四种类型：

- 即时向量(instant vector) 包含每个时间序列的单个样本的一组时间序列，共享相同的时间戳。
- 范围向量(Range vector) 包含每个时间序列随时间变化的数据点的一组时间序列。
- 标量(Scalar) 数字浮点值
- 字符串(String) 字符串值(目前未被使用)

examples：

查询K8S集群内所有apiserver健康状态

```
(sum(up{job="apiserver"} == 1) / count(up{job="apiserver"})) * 100
```

查询pod 聚合一分钟之内的cpu 负载

```
sum by (container_name)(rate(container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="acw62egvxd95l7t3q5uxee"}[1m]))
```

# 2. 时间序列选择器

## 1) 即时向量选择器

   即时向量选择器允许在给定时间戳（即时）选择一组时间序列和每个时间序列的单个样本值：在最简单的形式中，仅指定度量名称。这会生成一个即时向量，其中包含具有此指标名称的所有时间序列的元素。下面这个例子选择了具有container_cpu_usage_seconds_total指标的时间序列。

```
 container_cpu_usage_seconds_total
```

可以通过**附加一组标签**，并用{}括起来，来进一步筛选这些时间序列。
可以将标签值反向匹配，或者对正则表达式匹配标签值。
匹配操作符如下：

```
=：选择正好相等的字符串标签
!=：选择不相等的字符串标签
=~：选择匹配正则表达式的标签（或子标签）
!=：选择不匹配正则表达式的标签（或子标签）
```

下面这个例子只选择有container_cpu_usage_seconds_total名称的、有prometheus工作标签的、有pod_name组标签的时间序列。

```
container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="acw62egvxd95l7t3q5uxee"}
```

选择staging、testing、development环境下的，GET之外的HTTP方法的http_requests_total的时间序列

```
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

## 2) 范围向量选择器

   范围向量表达式如即时向量表达式一样运行，只不过返回的是当前时刻的时间序列。语法是，在一个向量表达式之后添加[]来表示时间范围，持续时间用数字表示。单位如下：

- s - seconds
- m - minutes
- h - hours
- d - days
- w - weeks
- y - years
  选择过去1分钟内的所有记录(即当前时间-1分钟 到 当前时间 这一时间段中prometheus收集到的相应的指标值)，metric名称为container_cpu_usage_seconds_total、作业标签为pod_name的时间序列的所有值。返回的记录的数量会因为prometheus规则中设置的指标的采集时间间隔而变化。

```
irate(container_cpu_usage_seconds_total{image!="",container_name!="POD",pod_name="acw62egvxd95l7t3q5uxee"}[1m])
```

    过去5分钟

```
http_requests_total{job="prometheus"}[5m]
```

## 3) 偏移修饰符

   偏移修饰符允许更改查询中单个即时向量和范围向量的时间偏移量。语法上只是在上述表达式后面加上 offset time
   下面给出两个例子:

1. 返回相对于当前查询时间5分钟前的container_cpu_usage_seconds_total值

```
    container_cpu_usage_seconds_total offset 5m
```

2. 返回container_cpu_usage_seconds_total在一天前5分钟内的速率

```
   (rate(container_cpu_usage_seconds_total{pod_name="acw62egvxd95l7t3q5uxee"} [5m] offset 1d)) 
```

## 4) 操作符

Prometheus 支持许多二元和聚合运算符。[官网介绍](https://prometheus.io/docs/prometheus/latest/querying/operators/)

### 算术二元运算符

   存在以下二元算术运算符

    - +加
      - -减
      - *乘
      - /除
      - %模
      - ^指数

二元算术运算符在标量/标量、向量/标量和向量/向量值对之间定义。

**在两个标量之间**，行为是显而易见的：它们评估为另一个标量，该标量是应用于两个标量操作数的运算符的结果。

**在瞬时向量和标量之间**，运算符应用于向量中每个数据样本的值。例如，如果时间序列即时向量乘以 2，则结果是另一个向量，其中原始向量的每个样本值都乘以 2。度量名称被删除。

**在两个瞬时向量之间**，对左侧向量中的每个条目及其右侧向量中的匹配元素应用二元算术运算符 。结果传播到结果向量中，分组标签成为输出标签集。指标名称被删除。在右侧向量中找不到匹配条目的条目不是结果的一部分(即这部分内容被删除)。

### 三角二元运算符

项目中没用到 忽略

### 比较二元运算符

Prometheus 中存在以下二进制比较运算符：

* `==`（等于）
* `!=`（不等于）
* `>`（大于）
* `<`（小于）
* `>=`（大于或等于）
* `<=`（小于或等于）

比较运算符在标量/标量、向量/标量和向量/向量值对之间定义。

标量之间，返回0/false或1/true。

瞬时向量和标量之间，将应用到向量中每个数据样本的值，并且删除那些结果为0/falsed的。

### 逻辑/集合二元运算符

这些运算符仅在瞬时向量之间定义。

- and
- or
- unless

`vector1 and vector2`产生一个由v1元素组成的向量， 其中的元素 `与vector2`具有完全匹配的标签集。其他元素被丢弃。度量名称和值从左侧向量继承（即得到的是v1中剩余的值）。（即同时在v1和v2的元素)

`vector1 or vector2`生成一个向量，该向量包含 v1的所有原始元素（标签集 + 值）以及 v2 中没有匹配v1标签集的所有元素。(即所有值)

`vector1 unless vector2`产生一个由v1元素组成的向量， 而v2中没有元素具有完全匹配的标签集。两个向量中的所有匹配元素都被丢弃。（即在v1中而不在v2中的元素）

### 向量匹配

向量之间的操作试图为左侧的每个条目在右侧向量中找到匹配元素。匹配行为有两种基本类型：一对一和多对一/一对多。

这些矢量匹配关键字允许在具有不同标签集的系列之间进行匹配：

* `on`
* `ignoring`

提供给匹配关键字的标签列表将决定向量的组合方式。[示例可以在一对一向量匹配](https://prometheus.io/docs/prometheus/latest/querying/operators/#one-to-one-vector-matches)和 [多对一和一对多向量匹配](https://prometheus.io/docs/prometheus/latest/querying/operators/#many-to-one-and-one-to-many-vector-matches)中找到。

### 组修饰符

启动多对一和一对多向量匹配

* `group_left`
* `group_right`

标签列表可以提供给组修饰符，其中包含来自“一”侧的标签以包含在结果度量中。

*多对一和一对多匹配是需要仔细考虑的高级用例（项目中尽可能少用，通常情况下，合理使用ignoring即可满足需求）*

### 聚合运算符

Prometheus 支持以下内置聚合运算符，可用于聚合单个瞬时向量的元素，从而产生具有聚合值的元素更少的新向量。

* `sum`（总和）
* `min`（选择最小）
* `max`（选择最大）
* `avg`（计算平均值）
* `group`（分组）
* `stddev`（总体标准偏差）
* `stdvar`（总体标准方差）
* `count`（计算向量中元素的数量）
* `count_values`（计算具有相同值的元素的数量）
* `bottomk`（样本值最小的 k 个元素）
* `topk`（样本值最大的 k 个元素）
* `quantile`（ 分位数）

另外通过 without 和 by 可以保留不同纬度的数据。

- without 可以删除指定标签，保留剩余标签
- by 保留指定标签，删除其他标签

## 5) 函数

- abs(v) 样本值转为绝对值
- absent() 项目可忽略
- absent_over_time() 项目可忽略
- ceil(v) 样本值四舍五入到最接近的整数
- changes(v) 返回输入的时间序列在提供的时间范围内更改的次数作为瞬时向量
- clamp(v,min,max) 将所有元素的样本值限制在min和max之间
- clamp_max()
- clamp_min()
- delta(范围向量v) 返回范围向量v中每个时间序列元素的第一个值和最后一个值之间的差值。
- idelta() 计算范围向量中最后两个样本之间的差异
- exp() 指数
- floor() 向下取整
- increase() 计算范围向量中时间序列的增加

以下示例表达式返回范围向量中每个时间序列在过去 5 分钟内测量的 HTTP 请求数：

```
increase(http_requests_total{job="api-server"}[5m])
```

- rate() 计算范围向量中时间序列的每秒平均增长率。
- irate() 计算范围向量中时间序列每秒的瞬时增长率。
- sort() 返回按样本值升序排序的向量元素
