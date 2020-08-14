---
title : "Orleans 最佳实践"
---

内容来自于微软的官方文档 https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/Orleans20Best20Practices.pdf

https://dotnet.github.io/orleans/Documentation/resources/Best_Practices.html

## Scenarios & General Fit

### 什么时候我们需要试试看Orleans

- 拥有大量（成百上千万的）低耦合的实体
- 实体足够小到可以在单一线程上完成工作
- 以“请求–响应”或“启动–监测–完成”的交互方式工作的
- 需要或可能需要在不止一台服务器上协同工作（集群）
- 一次只需要在几个实体之间进行协调工作，而不需要整体的协调工作。
- 在不同事件使用不同的实体。（视具体情况）

### 不适用于以下情况

- 实体需要直接去访问彼此的内存。
- 数量少，但是单体庞大的、需要在多线程下工作的实体
- 需要整体的协调和一致性
- 长期一直运行的操作、批处理或SIMD（视具体情况）

------

## 设计Grains

- **Actor 并不是平时理解的Object , 尽管它们非常相似。**
- 低耦合、基本独立
  - 独立于其他的Grains去封装和管理它们的状态。
  - 可以单独出故障（自闭可以，但不要影响其他Grains继续工作）
- 避免Grains之间有频繁的交流
  - 消息传递相较于直接访问内存的方式，性能开销的代价要大得多
  - 如果两个Grains之间一直有频繁的互调用，也许应该考虑是不是应该把它两设计合并成一个Grains
  - 要考虑参数的尺寸和复杂度，以及序列化问题。
    - 有时候，重发一个二进制消息并反序列化两次的开销是比较小的。
- 避免Grains的瓶颈
  - 单一的调度者、登记处或监测器
  - 如果有必要的话，进行分段的聚合

------

## 实现Grains —— 异步

- 一切都应该是异步的（TPL），不要有堵塞线程的操作
- 最佳的实践机制是 async/await

#### 经典案例：

返**回一个具体的值**：

return Task.FromResult(value);



**返回一个同类型的Task**

return foo.Bar();



**await 一个 Task之后继续执行点什么**

var x = await bar.Foo();
var y = DoSomething(x);
return y;



**Fan-out**:

var tasks = new List();
foreach(var grain in grains)
tasks.Add(grain.Foo());
await Task.WhenAll(tasks);
DoMore();

------

## 实现Grains

**什么时候使用**`**[StatelessWorker]**`

- 功能操作： 解码、解压缩，然后转发处理
- 多次激活，并且总是在本地的
- 比如：良好的分段聚合（首先在本地，不在Silo中）

**默认情况下，Grains是不可以重入的（意思应该是不可以递归）**

- 在调用周期中会出现死锁， 比如调用自身
- 死锁会因为超时而自动中止。
- 使用`[Reentrant]`属性（attribute）来使一个Grain类可重入。
- 重入将依然是单线程下的，但是可能会交叉。
- 处理交叉问题容易出错。

**继承**

- 继承一个Grain的接口（interface）是简单的
- 当多个Grain类继承了相同的接口时，可能需要消除歧义。
- Grain类的继承是有限的
  - 持久化声明会中断继承

**泛型是被支持的**

------

## 关于Grains持久化的概述

Orleans的Grain的状态持久化API被设计用来提供易于使用的可扩展的存储功能。

### Grain的状态持久化

- 定义一个接口（`interface`）继承自`Orleans.IGrainState`, 并在该接口中包含Grain里需要持久化的字段。
- Grain类需要继承 `GrainBase<T>` 并在Grain的基础类中添加强类型State字段