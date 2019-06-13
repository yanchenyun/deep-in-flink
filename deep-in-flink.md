---


typora-copy-images-to: images
typora-root-url: ./
---

**<font size=20 textAlign ="center">Deep In Flink</font>**

[TOC]

# 序言

本文不是简单的Flink中文文档的内容的摘录或者翻译，内容不会像Flink文档那样面面俱到，可以视作Flink文档之外的补充。

网上关于Flink的文章非常多，从Flink的基本开发、原理、源码解析，无所不包。经过近1年接触Flink，发现网上的内容过于零散，很少能有1篇文章或者系列文章，能够把Flink从全局到细节统一起来，让新接触Flink的人既有全局的视角，又能深入到Flink的细节设计之中，把握Flink的本质，不会望而生畏，特此编写。

另，类似于Flink的分布式计算引擎，如Spark、Storm等在原理上是相近的，深入解析Flink的原理，尽量抽取通用的概念，使读者对于分布式计算引擎领域能够更深的认识，则是更深层次的追求。

# Flink入门

## Flink可以做什么
## Flink的应用场景
## Flink生态
## Flink技术栈

![img](./images/flink-ecosystem.png)

## Flink抽象层次

![img](./images/flink-abstract-layer.svg)

- **Stateful Streaming Processing**

  最底层。它通过Process Function嵌入到DataStream API中。它允许用户从一个或多个流自由处理事件，并使用一致的容错状态。此外，用户可以注册事件时间和处理时间回调，允许程序实现复杂的计算。

- **Core APIs**

  实际上，大多数应用程序不需要上述的低级别抽象，而是针对Core API（如DataStream API（有界/无界流））和DataSet API（有界数据集）进行编程。这些流畅的API为数据处理提供了常见的构建模块，如用户指定的各种转换形式，连接，聚合，窗口，状态等。在这些API中处理的数据类型以各自的编程语言表示为classes。底层的Process Function和DataStream API的整合，使得针对一些特定的操作可以实现更低层次的抽象。DataSet API为有界数据集提供了额外的原函数，如循环/迭代。

- **Table API**

  Table API是以表为中心的声明式DSL，可能是动态更改表（表示流时）。Table API遵循（扩展）关系模型：Table 具有附加schema（与关系数据库中的表相似），API提供操作，例如select，project，join，group-by，aggregate等。Table API代表的是应该做什么逻辑操作，而不是直接指定如何编写操作的源代码。虽然Table API可以通过各种类型的用户定义的函数进行扩展，但它不如Core API那么具有表达力，但使用起来更加简洁（少写很多代码）。

  此外，Table API程序还可以通过在执行之前应用优化规则的优化器。

  可以在表和DataStream / DataSet之间无缝转换，允许程序将Table API和DataStream和DataSet API混合使用。

- **SQL**

  最高层次的抽象就是SQL。无论是语法还是表达，该层次的抽象都很像Table API。SQL抽象与Table API紧密交互，SQL查询可以在Table API中定义的表上执行。

## Flink运行架构

![img](./images/flink-run-architecture.svg)

## 从代码到运行

![img](./images/code-run1.svg)
[上图解释说明]

![img](./images/code-run2.svg)
[上图解释说明]

# Flink类型与序列化系统

## Flink类型系统

### Flink类型分类

![img](./images/flink-type-system.png)

### Flink类型Class

![img](./images/flink-type-class.png)

​		TypeInformation 类是描述一切类型的公共基类，它和它的所有子类必须可序列化（Serializable），因为类型信息将会伴随 Flink 的作业提交，被传递给每个执行节点。

​		由于 Flink 自己管理内存，采用了一种非常紧凑的存储格式，因而类型信息在整个数据处理流程中属于至关重要的元数据。

### 类型提取

#### 类型提取机制

​		Flink 内部实现了名为 **TypeExtractror** 的类，可以利用方法签名、子类信息等蛛丝马迹，自动提取和恢复类型信息（当然也可以显式声明）。

​		然而由于 Java 的类型擦除，自动提取并不是总是有效。因而一些情况下（例如通过 URLClassLoader 动态加载的类），仍需手动处理；例如下图中对 DataSet 变换时，使用 .returns() 方法声明返回类型。

![img](./images/type-extractor-code1.png)

​		**returns()** 接受三种类型的参数：**TypeInformation**（例如 "**String**"）、returns() 接受三种类型的参数：**TypeInformation**（例如 "String"）、**TypeHint**（接下来会讲到，用于泛型类型参数）、Java 原生 Class（例如 String.class) 。（接下来会讲到，用于泛型类型参数）、Java 原生 Class（例如 **String**.class) 。

#### Java中的类型擦除

​        [Java](http://lib.csdn.net/base/17) 泛型（Generic）的引入加强了参数类型的安全性，减少了类型的转换，但有一点需要注意：Java的泛型机制是在编译级别实现的。编译器生成的字节码在运行期间并不包含泛型的类型信息。也就是说所有泛型参数类型在编译后都会被清除掉，看下面一个列子，代码如下：

```java
public class Foo {  
    public void listMethod(List<String> stringList){  
    }  
    public void listMethod(List<Integer> intList) {  
    }  
}  
```

​		代码很简单，看起来没什么问题，但是编译器却报出如下错误信息：

```java 
Method listMethod(List<String>) has the same erasure listMethod(List<E>) as another method in type Foo
```

​        此错误的意思是说listMethod(List<String>) 方法在编译时擦除类型后的方法是listMethod(List<E>)，它与另外一个方法重复，也就是方法签名重复。反编译之后的方法代码如下：

```java
public void listMethod(List list)  
{  
}
```

**类型擦除的一些示例**

```java
List<String>、List<T> 擦除后的类型为 List。
List<String>[]、List<T>[] 擦除后的类型为 List[]。
List<? extends E>、List<? super E> 擦除后的类型为 List<E>。
List<T extends Serialzable & Cloneable> 擦除后类型为 List<Serializable>。
```

**Java类型擦除的原因**

```
1. 避免 JVM 的大换血。如果 JVM 将泛型类型延续到运行期，那么到运行期时 JVM 就需要进行大量的重构工作了，提高了运行期的效率。
2. 版本兼容。 在编译期擦除可以更好地支持原生类型（Raw Type）。
```

### 类型声明

#### 类型声明的快捷方式1

​		例如 BasicTypeInfo，这个类定义了一系列常用类型的快捷方式，对于String、Boolean、Byte、ShortInteger、Long、Float、Double、Char等基本类型的类型声明，可以直接使用。

![img](./images/type-declare-shotcut1.png)

例如下面是对Row 类型各字段的类型声明，使用方法非常简明，不再需要 new XxxTypeInfo<>(很多很多参数)

![img](./images/type-declare-shotcut2.png)

#### 类型声明的快捷方式2

​	如果觉得 BasicTypeInfo 还是太长，Flink 还提供了完全等价的Types类(org.apache.flink.api.common.typeinfo.Types)

![img](./images/type-declare-shortcut3.png)

特别需要注意的是，flink-table模块也有一个Types类（org.apache.flink.table.api.Types），用于
table模块内部的类型定义信息，用法稍有不同.

![img](./images/2.1.4.2flink-table-type.png)

## Flink 序列化

### 从数据类型到二进制数组

![img](./images/from-type-to-byte-array.png)

### Flink 序列化的结构示例

如下图所示，一个内嵌型的Tuple3对象的序列化过程。

![img](./images/flink-serialize-example.png)

### String类型序列过程示例

​		**StringSerializer.class**其中实现了serialize和deserialize方法，调用StringValue.class实现了数据的序列化和反序列化。

```java
@Override
	public void serialize(String record, DataOutputView target) throws IOException {
		StringValue.writeString(record, target);
	}
```

​		最终的实际序列化的动作交给**StringValue.class**执行，写入String的长度和String的值到java.io.DataOutput，实际上就是MemorySegment中。Flink中的序列化的类实际上都继承了DataOutputView接口，DataOutputView接口继承了DataOutput接口。

**序列化**

```java
public static final void writeString(CharSequence cs, DataOutput out) throws IOException {
		if (cs != null) {
			// the length we write is offset by one, because a length of zero indicates a null value
			int lenToWrite = cs.length()+1;
			if (lenToWrite < 0) {
				throw new IllegalArgumentException("CharSequence is too long.");
			}
	
			// write the length, variable-length encoded
			while (lenToWrite >= HIGH_BIT) {
				out.write(lenToWrite | HIGH_BIT);
				lenToWrite >>>= 7;
			}
			out.write(lenToWrite);
	
			// write the char data, variable length encoded
			for (int i = 0; i < cs.length(); i++) {
				int c = cs.charAt(i);
	
				while (c >= HIGH_BIT) {
					out.write(c | HIGH_BIT);
					c >>>= 7;
				}
				out.write(c);
			}
		} else {
			out.write(0);
		}
	}
```

**反序列化**

反序列化的逻辑是相反的。

```java
public static String readString(DataInput in) throws IOException {
		// the length we read is offset by one, because a length of zero indicates a null value
		int len = in.readUnsignedByte();
		
		if (len == 0) {
			return null;
		}

		if (len >= HIGH_BIT) {
			int shift = 7;
			int curr;
			len = len & 0x7f;
			while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
				len |= (curr & 0x7f) << shift;
				shift += 7;
			}
			len |= curr << shift;
		}
		
		// subtract one for the null length
		len -= 1;
		
		final char[] data = new char[len];

		for (int i = 0; i < len; i++) {
			int c = in.readUnsignedByte();
			if (c < HIGH_BIT) {
				data[i] = (char) c;
			} else {
				int shift = 7;
				int curr;
				c = c & 0x7f;
				while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
					c |= (curr & 0x7f) << shift;
					shift += 7;
				}
				c |= curr << shift;
				data[i] = (char) c;
			}
		}
		
		return new String(data, 0, len);
	}
```



## 类型系统的不足和问题

​		Flink 内置的类型系统虽然强大而灵活，但仍然有一些需要注意的点：

**1. Lambda** **函数的类型提取**

​		由于 Flink 类型提取依赖于继承等机制，而 lambda 函数比较特殊，它是匿名的，也没有与之相关的类，所以其类型信息较难获取。

​		Eclipse 的 JDT 编译器会把 lambda 函数的泛型签名等信息写入编译后的字节码中，而对于 javac 等常见的其他编译器，则不会这样做，因而 Flink 就无法获取具体类型信息了。

**2.** **Kryo** **的** **JavaSerializer** **在** **Flink** **下存在** **Bug**

​		推荐使用 org.apache.flink.api.java.typeutils.runtime.kryo.JavaSerializer 而非 com.esotericsoftware.kryo.serializers.JavaSerializer 以防止与 Flink 不兼容。

# Flink中常见的数据流的类型

## 数据流（DataStream）是什么

​		DataStream 是 Flink 流处理 API中最核心的数据结构。它代表了一个运行在多个分区上的并行流。一个 DataStream 可以从 StreamExecutionEnvironment 通过env.addSource(SourceFunction) 获得 。

​		DataStream 上的转换操作都是逐条的，比如map()，flatMap()，filter()。DataStream 也可以执行 rebalance（再平衡，用来减轻数据倾斜）和 broadcaseted（广播）等分区转换。 

![img](./images/datastream-example.png)

​		如上图的执行图所示，DataStream 各个算子会并行运行，算子之间是数据流分区。如 Source 的第一个并行实例（S1）和 flatMap() 的第一个并行实例（m1）之间就是一个数据流分区。而在 flatMap() 和 map() 之间由于加了 rebalance()，它们之间的数据流分区就有3个子分区（m1的数据流向3个map()实例）。

##  常用数据流

![img](./images/flink-stream-type.png)

### KeyedStream

​		KeyedStream用来表示根据指定的key进行分组的数据流。一个KeyedStream可以通过调用DataStream.keyBy()来获得。而在KeyedStream上进行任何transformation都将转变回DataStream。在实现中，KeyedStream是把key的信息写入到了transformation中。每条记录只能访问所属key的状态，其上的聚合函数可以方便地操作和保存对应key的状态。 

### WindowedStream **&** AllWindowedStream

​		WindowedStream代表了根据key分组，并且基于WindowAssigner切分窗口的数据流。所以WindowedStream都是从KeyedStream衍生而来的。而在WindowedStream上进行任何transformation也都将转变回DataStream。

```scala
val stream: DataStream[MyType] = env.addSource(new FlinkKafkaConsumer08[String](...))
val str1: DataStream[(String, MyType)] = stream.flatMap { ... }
val str2: DataStream[(String, MyType)] = stream.rebalance()
val str3: DataStream[AnotherType] = stream.map { ... }
```

上述 WindowedStream 的样例代码在运行时会转换成如下的执行图：

![img](./images/datastream-window-example.png)

​		Flink 的窗口实现中会将到达的数据缓存在对应的窗口buffer中（一个数据可能会对应多个窗口）。当到达窗口发送的条件时（由Trigger控制），Flink 会对整个窗口中的数据进行处理。Flink 在聚合类窗口有一定的优化，即不会保存窗口中的所有值，而是每到一个元素执行一次聚合函数，最终只保存一份数据即可。

​		在key分组的流上进行窗口切分是比较常用的场景，也能够很好地并行化（不同的key上的窗口聚合可以分配到不同的task去处理）。不过有时候我们也需要在普通流上进行窗口的操作，这就是 `AllWindowedStream`。`AllWindowedStream`是直接在DataStream上进行`windowAll(...)`操作。`AllWindowedStream` 的实现是基于 `WindowedStream` 的（Flink 1.1.x 开始）。Flink 不推荐使用`AllWindowedStream`，因为在普通流上进行窗口操作，就势必需要将所有分区的流都汇集到单个的Task中，而这个单个的Task很显然就会成为整个Job的瓶颈。

**<font color='red'>注意</font>**

> 不推荐使用AllWindowedStream，因为AllWindowedStream在执行时并发度为1.

### JoinedStreams **&** CoGroupedStreams

​		双流 Join 也是一个非常常见的应用场景。深入源码你可以发现，JoinedStreams 和 CoGroupedStreams 的代码实现有80%是一模一样的，JoinedStreams 在底层又调用了 CoGroupedStreams 来实现 Join 功能。除了名字不一样，一开始很难将它们区分开来，而且为什么要提供两个功能类似的接口呢？？

​		实际上这两者还是很点区别的。首先 co-group 侧重的是group，是对同一个key上的两组集合进行操作，而 join 侧重的是pair，是对同一个key上的每对元素进行操作。co-group 比 join 更通用一些，因为 join 只是 co-group 的一个特例，所以 join 是可以基于 co-group 来实现的（当然有优化的空间）。而在 co-group 之外又提供了 join 接口是因为用户更熟悉 join（源于数据库吧），而且能够跟 DataSet API 保持一致，降低用户的学习成本。

​		JoinedStreams 和 CoGroupedStreams 是基于 Window 上实现的，所以 CoGroupedStreams 最终又调用了 WindowedStream 来实现。

```scala
val firstInput: DataStream[MyType] = ...
val secondInput: DataStream[AnotherType] = ...

val result: DataStream[(MyType, AnotherType)] = firstInput.join(secondInput)
    .where("userId").equalTo("id")
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...})
```
上述 JoinedStreams 的样例代码在运行时会转换成如下的执行图：

![img](./images/datastream-join&group-example.png)

​		双流上的数据在同一个key的会被分别分配到同一个window窗口的左右两个篮子里，当window结束的时候，会对左右篮子进行笛卡尔积从而得到每一对pair，对每一对pair应用 JoinFunction。不过目前（Flink 1.1.x）JoinedStreams 只是简单地实现了流上的join操作而已，距离真正的生产使用还是有些距离。因为目前 join 窗口的双流数据都是被缓存在内存中的，也就是说如果某个key上的窗口数据太多就会导致 JVM OOM（然而数据倾斜是常态）。双流join的难点也正是在这里，这也是社区后面对 join 操作的优化方向，例如可以借鉴Flink在批处理join中的优化方案，也可以用ManagedMemory来管理窗口中的数据，并当**数据超过阈值时能spill到硬盘**。

### ConnectedStreams

​		在 DataStream 上有一个 union 的转换 dataStream.union(otherStream1, otherStream2, …)，用来合并多个流，新的流会包含所有流中的数据。union 有一个限制，就是所有合并的流的类型必须是一致的。ConnectedStreams 提供了和 union 类似的功能，用来连接两个流，但是与 union 转换有以下几个区别：

- ConnectedStreams 只能连接两个流，而 union 可以连接多于两个流。
- ConnectedStreams 连接的两个流类型可以不一致，而 union 连接的流的类型必须一致。
- ConnectedStreams 会对两个流的数据应用不同的处理方法，并且双流之间可以共享状态。这在第一个流的输入会影响第二个流时, 会非常有用。

​		如下 ConnectedStreams 的样例，连接 input 和 other 流，并在input流上应用map1方法，在other上应用map2方法，双流可以共享状态（比如计数）。

```scala
val input: DataStream[MyType] = ...
val other: DataStream[AnotherType] = ...

val connected: ConnectedStreams[MyType, AnotherType] = input.connect(other)

val result: DataStream[ResultType] = 
        connected.map(new CoMapFunction[MyType, AnotherType, ResultType]() {
            override def map1(value: MyType): ResultType = { ... }
            override def map2(value: AnotherType): ResultType = { ... }
        })
```

当并行度为2时，其执行图如下所示：

![img](./images/datastream-connect-example.png)

# Flink中的时间

​		与批处理相比，流处理中的时间的是不光是计算上的意义，更重要的是对计算行为的影响。

## Flink 中时间相关概念

 ### 时间类型

 **Event Time**

​		事件时间，事件发生时的时间。

  **Ingestion Time**

​		摄取时间，事件进入流处理系统的时间。

  **Processing Time**

​		处理时间，消息被计算引擎处理的时间，以各个计算节点的本地时间为准。

![Flink时间类型](E:/develop/github/deep-in-flink/images/flink-time-type.svg)

### 时间窗口

​		1.按分割标准划分：**TimeWindow、CountWindow**

​		2.按窗口行为划分：**Tumbling Window、Sliding Window、Session Window**

​	![img](./images/flink-window-type-example.png)

## Window原理与机制

![img](./images/flink-window-component.png)

​		1、首先上图中的组件都位于一个算子（window operator）中，数据流源源不断地进入算子，每一个到达的元素都会被交给 WindowAssigner。WindowAssigner 会决定元素被放到哪个或哪些窗口（window），可能会创建新窗口。因为一个元素可以被放入多个窗口中，所以同时存在多个窗口是可能的。注意，Window本身只是一个ID标识符，其内部可能存储了一些元数据，如TimeWindow中有开始和结束时间，但是并不会存储窗口中的元素。窗口中的元素实际存储在 Key/Value State 中，key为Window，value为元素集合（或聚合值）。为了保证窗口的容错性，该实现依赖了 Flink 的 State 机制（参见 [state ](https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/state.html)[文档](https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/state.html)）。

​		2、每一个窗口都拥有一个属于自己的 Trigger，Trigger上会有定时器，用来决定一个窗口何时能够被计算或清除。每当有元素加入到该窗口，或者之前注册的定时器超时了，那么Trigger都会被调用。Trigger的返回结果可以是 continue（不做任何操作），fire（处理窗口数据），purge（移除窗口和窗口中的数据），或者 fire + purge。一个Trigger的调用结果只是fire的话，那么会计算窗口并保留窗口原样，也就是说窗口中的数据仍然保留不变，等待下次Trigger fire的时候再次执行计算。一个窗口可以被重复计算多次知道它被 purge 了。在purge之前，窗口会一直占用着内存。

​		3、当Trigger fire了，窗口中的元素集合就会交给Evictor（如果指定了的话）。Evictor 主要用来遍历窗口中的元素列表，并决定最先进入窗口的多少个元素需要被移除。剩余的元素会交给用户指定的函数进行窗口的计算。如果没有 Evictor 的话，窗口中的所有元素会一起交给函数进行计算。

​		4、计算函数收到了窗口的元素（可能经过了 Evictor 的过滤），并计算出窗口的结果值，并发送给下游。窗口的结果值可以是一个也可以是多个。DataStream API 上可以接收不同类型的计算函数，包括预定义的sum(),min(),max()，还有 ReduceFunction，FoldFunction，还有WindowFunction。WindowFunction 是最通用的计算函数，其他的预定义的函数基本都是基于该函数实现的。

​		5、Flink 对于一些聚合类的窗口计算（如sum,min）做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个result值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改result值。这样可以大大降低内存的消耗并提升性能。但是如果用户定义了 Evictor，则不会启用对聚合窗口的优化，因为 Evictor 需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

### Window Assigner

**Window Assigner**用来决定某个元素被分配到哪个/哪些窗口中去。

**Session WindowAssigner**比较特殊，因为**Session Window**无法事先确定窗口的范围，是动态改变的。

### 触发器 Window Trigger

​		Trigger触发器。决定了一个窗口何时能够被计算或清除，每一个窗口都拥有一个属于自己的Trigger，Trigger上会有定时器，用来决定一个窗口何时能够被计算或清除。每当有元素加入到该窗口，或者之前注册的定时器超时了，那么Trigger都会被调用。

**Trigger触发的结果**

- Continue(不做任何操作)
- Fire(处理窗口数据)
- Purge(移除窗口和窗口中的数据) 
- fire + purge

​        一个Trigger的调用结果只是fire的话，那么会计算窗口并保留窗口原样，也就是说窗口中的数据仍然保留不变，等待下次Trigger fire的时候再次执行计算。一个窗口可以被重复计算多次直到它被purge了。在**purge**之前，窗口会一直占用着内存。

**Trigger fire的流程**

1. 当Trigger fire了，窗口中的元素集合就会交给Evictor（如果指定了的话）。Evictor 主要用来遍历窗口中的元素列表，并决定最先进入窗口的多少个元素需要被移除。剩余的元素会交给用户指定的函数进行窗口的计算。如果没有Evictor 的话，窗口中的所有元素会一起交给函数进行计算。
2. 计算函数收到了窗口的元素（可能经过了 Evictor 的过滤），并计算出窗口的结果值，并发送给下游。窗口的结果值可以是一个也可以是多个。DataStream API 上可以接收不同类型的计算函数，包括预定义的sum(),min(),max()，还有 ReduceFunction，FoldFunction，还有WindowFunction。WindowFunction 是最通用的计算函数，其他的预定义的函数基本都是基于该函数实现的。
3. Flink 对于一些聚合类的窗口计算（如sum,min）做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个result值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改result值。这样可以大大降低内存的消耗并提升性能。但是如果用户定义了 Evictor，则不会启用对聚合窗口的优化，因为 **Evictor** 需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

### 窗口过滤器Evictor

​		**Evictor****直译为“驱逐者”。在Trigger触发之后，在窗口被处理之前，Evictor（如果有Evictor的话）会用来剔除窗口中不需要的元素，相当于一个filter。

> **<font color='red'>注意：</font>**Flink 对于一些聚合类的窗口计算（如sum,min）做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个result值就可以了。每个进入窗口的元素都会执行一次聚合函数并修改result值。这样可以大大降低内存的消耗并提升性能。但是如果用户定义了Evictor，则不会启用对聚合窗口的优化，因为Evictor需要遍历窗口中的所有元素，必须要将窗口中所有元素都存下来。

### Count Window实现

​	Count Window 是使用三组件的典范，我们可以在 `KeyedStream` 上创建 Count Window，其源码如下所示：

​	在**KeyedStream** 上创建 **CountWindow**，其源码如下所示：

```java
// tumbling count window 翻滚计数窗口
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size) {
  return window(GlobalWindows.create())  // create window stream using GlobalWindows
      .trigger(PurgingTrigger.of(CountTrigger.of(size))); // trigger is window size
}
// sliding count window 滑动计数窗口
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
  return window(GlobalWindows.create())
    .evictor(CountEvictor.of(size))  // evictor is window size
    .trigger(CountTrigger.of(slide)); // trigger is slide size
```

​		第一个函数是申请翻滚计数窗口，参数为窗口大小。第二个函数是申请滑动计数窗口，参数分别为窗口大小和滑动大小。它们都是基于 `GlobalWindows` 这个 WindowAssigner 来创建的窗口，该assigner会将所有元素都分配到同一个global window中，所有`GlobalWindows`的返回值一直是 `GlobalWindow` 单例。基本上自定义的窗口都会基于该assigner实现。

​		翻滚计数窗口并不带evictor，只注册了一个trigger。该trigger是带purge功能的 CountTrigger。也就是说每当窗口中的元素数量达到了 window-size，trigger就会返回fire+purge，窗口就会执行计算并清空窗口中的所有元素，再接着储备新的元素。从而实现了tumbling的窗口之间无重叠。

> 翻滚计数窗口之间无重叠，所以可以安全的清理已触发窗口的数据

​		滑动计数窗口的各窗口之间是有重叠的，但我们用的 GlobalWindows assinger 从始至终只有一个窗口，不像 sliding time assigner 可以同时存在多个窗口。所以trigger结果不能带purge，也就是说计算完窗口后窗口中的数据要保留下来（供下个滑窗使用）。另外，trigger的间隔是slide-size，evictor的保留的元素个数是window-size。也就是说，每个滑动间隔就触发一次窗口计算，并保留下最新进入窗口的window-size个元素，剔除旧元素。

​		**滑动计数窗口示例**

假设有一个滑动计数窗口，每2个元素计算一次最近4个元素的总和，那么窗口工作示意图如下所示：

![img](./images/count-window-example.png)

​		图中所示的各个窗口逻辑上是不同的窗口，但在物理上是同一个窗口。该滑动计数窗口，trigger的触发条件是元素个数达到2个（每进入2个元素就会触发一次），evictor保留的元素个数是4个，每次计算完窗口总和后会保留剩余的元素。所以第一次触发trigger是当元素5进入，第三次触发trigger是当元素2进入，并驱逐5和2，计算剩余的4个元素的总和（22）并发送出去，保留下2,4,9,7元素供下个逻辑窗口使用。

### Time Window实现

同样的，我们也可以在 `KeyedStream` 上申请 Time Window，其源码如下所示：

```java
// tumbling time window 滚动时间窗口
public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size) {
  if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
    return window(TumblingProcessingTimeWindows.of(size));
  } else {
    return window(TumblingEventTimeWindows.of(size));
  }
}
// sliding time window 滑动时间窗口
public WindowedStream<T, KEY, TimeWindow> timeWindow(Time size, Time slide) {
  if (environment.getStreamTimeCharacteristic() == TimeCharacteristic.ProcessingTime) {
    return window(SlidingProcessingTimeWindows.of(size, slide));
  } else {
    return window(SlidingEventTimeWindows.of(size, slide));
  }
}
```

这里我们主要分析sliding process time window，如下是相关源码：

```java
public class SlidingProcessingTimeWindows extends WindowAssigner<Object, TimeWindow> {
  private static final long serialVersionUID = 1L;

  private final long size;
  private final long slide;

  private SlidingProcessingTimeWindows(long size, long slide) {
    this.size = size;
    this.slide = slide;
  }

  @Override
  public Collection<TimeWindow> assignWindows(Object element, long timestamp) {
    timestamp = System.currentTimeMillis();
    List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
    // 对齐时间戳
    long lastStart = timestamp - timestamp % slide;
    for (long start = lastStart;
      start > timestamp - size;
      start -= slide) {
      // 当前时间戳对应了多个window
      windows.add(new TimeWindow(start, start + size));
    }
    return windows;
  }
  ...
}
public class ProcessingTimeTrigger extends Trigger<Object, TimeWindow> {
  @Override
  // 每个元素进入窗口都会调用该方法
  public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) {
    // 注册定时器，当系统时间到达window end timestamp时会回调该trigger的onProcessingTime方法
    ctx.registerProcessingTimeTimer(window.getEnd());
    return TriggerResult.CONTINUE;
  }

  @Override
  // 返回结果表示执行窗口计算并清空窗口
  public TriggerResult onProcessingTime(long time, TimeWindow window, TriggerContext ctx{
    return TriggerResult.FIRE_AND_PURGE;
  }
  ...
}
```

​		首先，`SlidingProcessingTimeWindows`会对每个进入窗口的元素根据系统时间分配到`(size / slide)`个不同的窗口，并会在每个窗口上根据窗口结束时间注册一个定时器（相同窗口只会注册一份），当定时器超时时意味着该窗口完成了，这时会回调对应窗口的Trigger的`onProcessingTime`方法，返回FIRE_AND_PURGE，也就是会执行窗口计算并清空窗口。整个过程示意图如下：

![img](./images/sliding-window-example.png)

​		如上图所示横轴代表时间戳（为简化问题，时间戳从0开始），第一条record会被分配到[-5,5)和[0,10)两个窗口中，当系统时间到5时，就会计算[-5,5)窗口中的数据，并将结果发送出去，最后清空窗口中的数据，释放该窗口资源。

### Session Window实现

​	假设有这么个场景，用户点开手机淘宝后会进行一系列的操作（点击、浏览、搜索、购买、切换tab等），这些操作以及对应发生的时间都会发送到服务器上进行用户行为分析。那么用户的操作行为流的样例可能会长下面这样：

![img](/images/7260028-b1afe34a217525b0.webp)

​		通过上图，我们可以很直观地观察到，用户的行为是一段一段的，每一段内的行为都是连续紧凑的，段内行为的关联度要远大于段之间行为的关联度。我们把每一段用户行为称之为“session”，段之间的空档我们称之为“session gap”。所以，理所当然地，我们应该按照 session window 对用户的行为流进行切分，并计算每个session的结果。如下图所示：

​	![img](/images/1560242105943.png)

为了定义上述的窗口切分规则，我们可以使用 Flink 提供的 SessionWindows这个 widnow assigner API。如果你用过SlidingEventTimeWindows、TumlingProcessingTimeWindows等，你会对这个很熟悉。

```java
DataStream input = …
DataStream result = input
  .keyBy(<key selector>)
  .window(SessionWindows.withGap(Time.seconds(<seconds>))
  .apply(<window function>) // or reduce() or fold()
```

这样，Flink 就会基于元素的时间戳，自动地将元素放到不同的session window中。如果两个元素的时间戳间隔小于 session gap，则会在同一个session中。如果两个元素之间的间隔大于session gap，且没有元素能够填补上这个gap，那么它们会被放到不同的session中。

Session窗口是动态窗口，无法实现定义好，所以可能会出现窗口大小改变、两个Session窗口合并的情况。

#### Session Winow合并

例如处理过程中到了两个元素，它们被分配到两个独立的窗口中，两个窗口目前不相交，如下图：

![1560242565322](/images/1560242565322.png)

当第三个元素进入时，分配到的窗口与现有的两个窗口发生了叠加，情况变成了这样：

![1560242596923](/images/1560242596923.png)

​		由于我们支持了窗口的合并，`WindowAssigner`可以合并这些窗口。它会遍历现有的窗口，并告诉系统哪些窗口需要合并成新的窗口。Flink 会将这些窗口进行合并，合并的主要内容有两部分：

1. 需要合并的窗口的底层状态的合并（也就是窗口中缓存的数据，或者对于聚合窗口来说是一个聚合值）

2. 需要合并的窗口的Trigger的合并（比如对于EventTime来说，会删除旧窗口注册的定时器，并注册新窗口的定时器）

   最终的结果是三个元素现在在同一个窗口中了：

![1560242611265](/images/1560242611265.png)

​		需要注意的是，对于每一个新进入的元素，都会分配一个属于该元素的窗口，都会检查并合并现有的窗口。在触发窗口计算之前，每一次都会检查该窗口是否可以和其他窗口合并，直到trigger触发后，会将该窗口从窗口列表中移除。对于 event time 来说，窗口的触发是要等到大于窗口结束时间的 watermark 到达，当watermark没有到，窗口会一直缓存着。所以基于这种机制，可以做到对乱序消息的支持。

​		这里有一个优化点可以做，因为每一个新进入的元素都会创建属于该元素的窗口，然后合并。如果新元素连续不断地进来，并且新元素的窗口一直都是可以和之前的窗口重叠合并的，那么其实这里多了很多不必要的创建窗口、合并窗口的操作，我们可以直接将新元素放到那个已存在的窗口，然后扩展该窗口的大小，看起来就像和新元素的窗口合并了一样

除了合并窗口之外还需要合并**State**和**Trigger**：

1. 需要合并的窗口的底层状态的合并（也就是窗口中缓存的数据，或者对于聚合窗口来说是一个聚合值）。
2. 需要合并的窗口的Trigger的合并（比如对于EventTime来说，会删除旧窗口注册的定时器，并注册新窗口的定时器）。

#### Session Window State、Trigger合并

​		例如A、B、C三个窗口需要合并，合并后的窗口为D窗口。每个窗口有自己的State，State和Trigger同时也会被合并。

​		State合并时，会选择A、B、C中一个的State作为新的State。比如选择B作为状态窗口，那么A和C窗口中的数据需要合并到B窗口中去。这样就没有新状态产生了，但是我们需要额外维护窗口与状态窗口之间的映射关系（D>B），这就是MergingWindowSet负责的工作。这个映射关系需要在失败重启后能够恢复，所以MergingWindowSet内部也是对该映射关系做了容错。状态合并的工作示意图如下所示：

![1560242391352](./images/1560242391352.png)



​		旧的Trigger需要被清除掉，为新的窗口创建新的Trigger。

### 窗口函数

Flink中窗口提供如下4类函数

- **ReduceFunction**
- **AggregateFunction**
- **FoldFunction**
- **ProcessWindowFunction** 

ReduceFunction、AggregateFunction、FoldFunction属于增量聚合类的函数，执行效率高。ProcessWindowFunction通过增量缓存，实现灵活，计算效率相比增量聚合稍低，毕竟要占用更多的内存。

#### Reduce Function



#### Aggregation Function



#### Fold Function 



#### ProcessFunction

## Watermark

### Watermark的作用

watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用watermark机制结合window来实现。

流处理从事件产生，到流经source，再到operator，中间是有一个过程和时间的。虽然大部分情况下，流到operator的数据都是按照事件产生的时间顺序来的，但是也不排除由于网络、背压等原因，导致乱序的产生（out-of-order或者说late element）。

但是对于late element，我们又不能无限期的等下去，必须要有个机制来保证一个特定的时间后，必须触发window去进行计算了。这个特别的机制，就是watermark。

### Watermark的生成

通常Watermark在Source中生成，但是Flink提供了额外的机制，允许在调用map、filter类的操作之后生成。

Watermark的生成机制分为如下两类：

- **With Periodic Watermarks**

  周期性的(一定时间间隔或者达到一定的记录条数)产生一个Watermark。在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

- **With Punctuated Watermarks**

  数据流中每一个递增的EventTime都会产生一个Watermark。在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

### Watermark在Flink内部的处理逻辑

![1560245376304](/images/1560245376304.png)

### 多流的Watermark处理

在实际的流计算中往往一个job中会处理多个Source的数据，对Source的数据进行GroupBy分组，那么来自不同Source的相同key值会shuffle到同一个处理节点，并携带各自的Watermark，Apache Flink内部要保证Watermark要保持单调递增，多个Source的Watermark汇聚到一起时候可能不是单调自增的，这样的情况ApacheFlink内部是如何处理的呢?如图所示

![1560245393737](/images/1560245393737.png)

Apache Flink内部实现每一个边上只能有一个递增的Watermark，当出现多流携带Eventtime汇聚到一起(GroupBy orUnion)时候，Apache Flink会选择所有流入的Eventtime中最小的一个向下游流出。从而保证watermark的单调递增和保证数据的完整性。

![1560245464567](/images/1560245464567.png)

# Flink 容错

## 流处理保证语义

Flink**提供了吐下流处理保证语义。

| **语义**                                         | **说明**                                                     |
| :----------------------------------------------- | :----------------------------------------------------------- |
| **At-Most-Once**   **最多一次**                  | 不开启checkpoint即最多一次                                   |
| **At-Least-Once**   **最少一次**                 | 开启checkpoint不进行barrier对齐                              |
| **Exactly-Once**   **引擎内严格一次**            | 开启checkpoint，且对barrier进行对齐                          |
| **End-to-end Exactly-Once**   **端到端严格一次** | 端到端Exactly Once语义需要Source的实现能够从失败的位置重新读取，Sink的实现需要外部数据存储提供RollBack机制，flink提供了Two Phase Commit模式实现端到端的Exactly Once |

## Exactly Once语义

### Fault Tolerant的演进之路

对于7×24小时不间断运行的流程序来说，要保证fault tolerant是很难的，这不像是离线任务，如果失败了只需要清空已有结果，重新跑一次就可以了。对于流任务，如果要保证能够重新处理已处理过的数据，就要把数据保存下来；而这就面临着几个问题：比如一是保存多久的数据？二是重复计算的数据应该怎么处理，怎么保证幂等性？

**基本要求**

- 最好能做到exactly-once
- 处理延迟越低越好
- 吞吐量越高越好
- 计算模型应当足够简单易用，又具有足够的表达力
- 从错误恢复的开销越低越好
- 足够的流控制能力（背压能力）

#### Storm的Record acknowlement模式

![1560245775200](/images/1560245775200.png)

Tuple的完全处理需要Spout、Bolt以及Acker（Storm中用来记录某颗Tuple树是否被完全处理的节点）协调完成，如图所示。从Spout发送Tuple到下游，并把相应信息通知给Acker，整颗Tuple树中某个Tuple被成功处理了都会通知Acker,待整颗Tuple树都被完全处理完成之后，Acker将成功处理信息返回给Spout；如果某个Tuple处理失败，或者超时，Acker将会给Spout发送一个处理失败的消息，Spout根据Acker的返回信息以及用户对消息保证机制的选择判断是否需要进行消息重传。

#### Google Dataflow的事务日志模式

##### 执行模型

![1560245870942](/images/1560245870942.png)

##### Exectly Once --Shuffle

Exactly Once语义的保证主要是在跨节点交互的时候需要。本地JVM内部并不会有Exactly Once的问题。

Dataflow通过为每一条记录提供全局唯一uid的方式提供Exactly Once语义。每收到1个Record，都要验证是否重复，并且每1个Step的output都要checkpoint到可靠存储中。每一个Step都进行checkpoint的设计计算成本相当高，后边会介绍Google Dataflow的优化策略。

**上游备份 upstream backup，防止丢失**

​        为了防止数据丢失，Dataflow引入了上游备份的概念。发送者如果收不到下游的确认则会尝试重发。

​        同时保证上游发送者崩溃恢复之后仍然会重发。这样就提供了至少一次（at least once）的的保证。

​        在分布式系统的RPC调用中，只有对方明确的确认收到消息了才是可靠的，其他的失败情况如超时、对端处理完但是确认消息没有发出来等等，RPC调用失败，数据可能被对方成功处理，也可能没有，是一种不确定的状态，在shuffle中的潜在风险是数据可能被重复传输。

**下游去重，防止重复计算**

​       为每一个record消息赋予唯一标识符uid，接受者维护一个catalog目录，目录里记录接收和处理的数据。每收到一条数据，就在catalog中查询是否有重复记录，如果有则丢弃，反之没有重复记录，则在处理完成后记录在catalog中。Dataflow基于谷歌云，使用BigTable(Hbase）作为Catalog的存储。

![1560246911710](/images/1560246911710.png)

##### 分布式计算中的不确定性

**什么是非确定性？**即对同一条数据的两次计算输出结果可能是不同的。

非确定性是不可避免，并且是数据处理过程中必要的。例如：

​      1）维表join，通过查询外部表补充字段，维表可能会被修改（插入、删除、更细），两次维表Join的执行中间，维表的数据改变了，结果可能会不一样。

​     2）再比如依赖于当前时间的计算大概率是非确定性计算。

​     3）即便代码是确定性的，但是允许数据延迟的基于事件时间的聚合，也很有可能在Worker失败之后恢复之后，在上次计算中迟到的数据到来了，迟到数据变成了非迟到数据，这也可能导致窗口两次执行的聚合结果是不同的。

在Dataflow中，一条数据因为work节点失败、rpc传输失败等情况被重复计算是很常见的，但是要保证Exactly Once语义，也就是说重复计算多次，但有且只有1次计算结果被采纳。

Dataflow中采用了Checkpoint日志机制，每一次要交给下游Stage的输出，（uid，output）都会被持久化到可靠存储中（此处需要特别注意性能问题）。无论重复执行多少次，其本质是重放checkpoint日志，确保Exactly Once。

##### Dataflow中的优化

###### Graph Optimization

**Fusion**

类似于flink中的Operator Chain，把多个step组合在一起，执行的时候会调度在同1个机器的同一个线程中执行，Operator chain内部的step之前不需要checkpoint进行Exactly Once的语义的保证。

![1560251780998](/images/1560251780998.png)

**聚合优化**

先执行本地的局部聚合，向下游发送聚合后的结果，下游再进行全局的结合类运算。Blink中已经实现了此类优化（LocalAggregation）。

![1560251831405](/images/1560251831405.png)

###### BloomFilter

Dataflow中依赖于一个Catalog进行去重，使用bloom filter可以提高查重效率。Bloom Filter查询没有，则没有重复数据。Bloom Filter查询有重复的，则可能存在重复，此时再去Catalog中进行查询。

**使用BloomFilter的基本假设**

正常情况下，重复的数据很少！

但是随着时间推移，BloomFilter越来越大，需要进行二次查询的概率逐渐增大。二次查询的成本很高。

DataFlow中按照时间范围对BloomFilter进行分桶（划分bucket），接收数据的时候，根据数据的时间戳选择使用的Bloom Filter所在的Bucket。每个分桶内的数据量不多，可以保证二次查询的概率特别低。

![1560251913815](/images/1560251913815.png)

###### GC垃圾回收

Dataflow的状态和一致性模型是基于Key的，在实现中，每个key会存储一个catalog记录该key下，收到的record。此时就涉及一个问题，这些状态不能一直保存下去，如何清理？何时清理？

垃圾回收策略一般有如下两种：

- **GC watermark**（Dataflow选择的实现方式）

类似于计算数据WaterMark一样，Dataflow会计算一个GC Watermark，Watermark之前的Catalog中的数据会被清理掉。

GC Watermark基于当前的系统时间，不依赖于事件时间。

GC Watermark可以体现在每一个State中数据处理的延迟。

- **严格有序递增序列号**

上游的发送者为每一条记录赋予一个严格递增的序号，跟踪序号，就知道序号为n之前的数据是否完全达到了，如果完全到达了，则n之前的0 … n-1的记录的catalog中记录的记录的uid可以被安全的清理掉。

#### Flink的分布式快照模式

##### Flink分布式快照的基本假设

- Job异常和失败极少发生，因为一旦发生异常，Job回滚到上一个状态的成本很高。
- 为了低延迟，快照需要很快就能完成。
- Task与Worker之间的关系是静态的，即分配完成之后，在Job运行过程中不会改变（至少是一个快照周期内），除非手动改变并行度。恢复的时候也会恢复到原先的拓扑结构。

##### 分布式系统的Global State

​	分布式系统由空间上分立的process和连接这些process的channel组成。空间上分立的含义是,这些process不共享memory,而是通过在communication channel上进行的message pass来异步交流。分布式系统的global state就是所有process,channel的local state的集合process的local state取决于the state of local memory and the history of its activity。channel的local state是上游process发送进channel的message集减去下游process从channel接收的message的差集。

**一致性Global State**

![1560252275202](/images/1560252275202.png)

假设有两个银行账户A,B.A中初始有600美元,B中初始有200美元. SA, SB, CAB, CBA由A和B分别记录,组成了global state.

在t0时刻,A向B转账50美元;在t1时刻,B向A转账80美元.

如果SA, SB记录于(t0, t1), CAB, CBA记录于(t1, t2),那么global state = 550+200+50+80 = 880,比真实值多了$80. 这就是**不一致**性global state.

如果 SA, SB, CAB, CBA同属于一个时间区间,那么得到的global state就是**一致性**的.

**Snapshot算法获得一致性state的难点**

分布式系统没有共享内存(globally shared memory)和全局时钟(global clock)。

如果分布式系统有共享内存,那么可以从共享内存中直接获取整个分布式系统的snapshot,无需分别获得各个process,channel的local state再组合成global state。

如果分布式系统有global clock,那么所有的process能在同一时刻各自记录local state,这样就保证了state的一致性。

**获得一致性global state的算法----Chandy-Lamport算法**

**精髓**:该算法在普通message中插入了control message – marker

**前提**:

1. message的传输可能有delay,但一定会到达

   2.每两个process之间都有一条communication path(可能由多条channel组成)

3. Channel是单向的FIFO

**描述**:

```java
Marker sending rule for process Pi`

	(1)Process Pi 记录自身state`

	(2)Pi在记录自身state后,发送下一条message前,Pi向自己所有的outgoing channel发送marker

	Marker receiving rule for process Pj on receiving a marker along channel C

	如果Pj第一次接收到marker,那么

		把channel C的state记为空集

		执行marker sending rule

	否则(并非第一次接收到marker)

		把记录自身state(或最近一次记录另一个channel的state)后,收到这个marker前的message集记为C的state

	每个process会记录自身的state和它的incoming channel的state
```

**Chandy-Lamport算法示例**

**1**.Process A启动snapshot算法,A执行marker sending rule(记录自身state,然后发送marker):

![1560252524960](/images/1560252524960.png)

**2**. Process B接收到marker,执行marker receiving rule:将channel AB的state记为空集,然后记录自身state并向下发送marker:

![1560252587160](/images/1560252587160.png)

**3.**Process C接收到marker,执行markerreceiving rule:将channel AC的state记为空集,然后记录自身state并向下发送marker:

![1560252646954](/images/1560252646954.png)

**4.** Process D接收到来自于process B的marker, 执行marker receiving rule:将channel BD的state记为空集,然后记录自身state并向下发送marker:

![1560252690460](/images/1560252690460.png)

**5**.Process D接收到来自于processC的marker,执行marker receiving rule:这是process D第二次接收到marker,将channel CD的state记为{5},不会向下发送marker:

![1560252816852](/images/1560252816852.png)

**6.** 到此process A,B,C,D的local state和所有Channel的state都记录完毕. 将这些local state组合,所得到的就是global state。

##### Flink的Aysnchronous Barrier Snapshotting算法

为了消去记录channel state这一步骤,process在接收到第一个barrier后不会马上做snapshot,而是等待接受其他上游channel的barrier.在等待期间,process会把barrier已到的channel的record放入input buffer。当所有上游channel的barrier到齐后,process才记录自身state,之后向所有下游channel发送barrier。因为先到的barrier会等待后到的barrier,所有barrier相当于同时到达process,因此,该process的上游channel的state都是空集.这就避免了去记录channel的state。

**异步分布式快照示例-exactly once**

1.JobManager发起一次snapshot:向所有source发送barrier。

![1560253056321](/images/1560253056321.png)

2.每个Barrier先后到达各自的source.Source在收到barrier后记录自身state,然后向下游节点发送barrier

![1560253084694](/images/1560253084694.png)

3.Barrier (from)B 到达process D,但不会进行snapshot

![1560253119362](/images/1560253119362.png)

4.Barrier (from)B已经到达process D,所以当来自于channel BD的record 6 7到达后,process D不会处理它们,而是将它们放入input buffer。而Barrier (from)C尚未到达process D,所以当来自于channel CD的record 4到达后,process D会处理它。

![1560253232764](/images/1560253232764.png)

5.Barrier C也到达process D.这样,process D已经接收到了所有上游process的barrier。process D记录自身state,然后向下游节点发送barrier。

![1560253279698](/images/1560253279698.png)

6.最终状态

![1560253302335](/images/1560253302335.png)



异步分布式快照示例-At Least once

不进行barrier对齐，则实现了At least once语义，过程如下图所示

1.

![1560253361289](/images/1560253361289.png)

2.

![1560253380714](/images/1560253380714.png)

3.

![1560253393915](/images/1560253393915.png)

​		Record 6本不属于这次checkpoint,却包含在process D的local state中。在recovery时,source认为record 6还没有被处理过,所以重发record 6. 这就导致stream中出现了两个record 6,造成了at least once。

​		这里的问题在于,当第二个barrier到达时,节点D再次对自身做了snapshot。而在Chandy-Lamport的算法中,第二个barrier到达时,节点D应该对barrier来源的channel做snapshot。

​		对单一input channel的算子来说,没有Alignment这个概念.这些算子在at least once模式下也是呈现exactly once的行为。



## End-to-End Exactly Once

Flink通过**两阶段提交协议**实现了Source->计算->Sink整个过程的端到端Exactly Once，无论是什么原因导致的Job失败，都保证数据严格读取、处理、写入一次。

**计算引擎内Exactly Once**

![1560253850058](/images/1560253850058.png)

**端到端Exactly Once**

![1560253874938](/images/1560253874938.png)



### 两阶段提交协议



## Flink State原理

### State的类型和作用

Flink中包含两种基础的状态：Keyed State和Operator State。

**Keyed State**

- ​	基于KeyedStream上的状态。这个状态是跟特定的key绑定的，对KeyedStream流上的每一个key，可能都对应一个state。KeyedState可以使用所有的State。

**Operator State**

- ​	与Keyed State不同，Operator State跟一个特定operator的一个并发实例绑定，整个operator只对应一个state。相比较而言，在一个operator上，可能会有很多个key，从而对应多个keyed state。
- ​	Operator States的数据结构不像Keyed States丰富，现在只支持List。

### 原始和托管State

Keyed State和Operator State，可以以两种形式存在：原始状态和托管状态。

托管Managed State是由Flink框架管理的状态，如ValueState, ListState, MapState等。

原始Raw state即原始状态，由用户自行管理状态具体的数据结构，框架在做checkpoint的时候，使用byte[]来读写状态内容，对其内部数据结构一无所知。

> 通常在DataStream上的状态推荐使用托管的状态，当实现一个用户自定义的operator时，会使用到原始状态。

### State类继承关系

![1560254360337](/images/1560254360337.png)

### State过期

#### 使用API控制State过期

Flink SQL是高层的抽象所以并没有提供对State清理精细控制，对于复杂的DataStrea程序而言，有时需要精细的控制，此时可以通过API来进行精细的控制。如下代码所示，对每一个State可以设置清理的策略StateTtlConfig。

**过期时间：**超过多久未访问则视为过期，类似于缓存

**过期时间更新策略：**创建和写时更新、读取和写时更新

**State的可见性：** 未清理则可用，超期即不可用

![1560254448026](/images/1560254448026.png)

#### Flink Sql中State过期

Flink SQL是数据分析的高层抽象，在SQL的世界里并无State的概念，而在流join、聚合类的场景中，使用了State，如果State不定时清理，则可能会导致State过多，内存溢出。为了稳妥起见，最好为每个Flink SQL Job提供State清理的策略。

![1560254525791](/images/1560254525791.png)

> ​	**阿里经验**
>
> TTL一般为1.5天左右

### State过期清理

默认情况下，只有在明确读出过期值时才会删除过期值，例如 通过调用ValueState.value（）。

**做完整快照时清理**
在获取完整状态快照时激活清理，减小其大小。
在当前实现下不清除本地状态，但在从上一个快照恢复的情况下，它不会包括已删除的过期状态。 可以在StateTtlConfig中配置：

![1560254701357](/images/1560254701357.png)

> <font color='red'>注意：此选项不适用于RocksDB StateBackend中的增量Checkpoint。</font>

**增量清理**

通过增量触发器渐进清理State。一种设计是当进行状态访问或者在处理Event的进行回调，在回调函数中进行处理。当每次增量清理触发时，遍历StateBackend中的状态，清理掉过期的。

![1560254827047](/images/1560254827047.png)

### Flink容错信息存储

Flink支持三种容错信息的存储(StateBackend)

**纯内存**

MemoryStateBackend，适用于验证、测试，不推荐生产环境。

**内存+文件**

FsStateBackend，适用于长周期大规模的数据。

**RocksDB**

RocksDBStateBackend，适用于长周期大规模的数据。

![1560255001589](/images/1560255001589.png)



存储机制示意图如下：

![1560254896983](/images/1560254896983.png)

#### MemoryStateBackend&FsStateBackend



##### MemoryStateBackend

MemoryStateBackend的State数据保存在java堆内存中，执行checkpoint的时候，会把state的快照数据保存到jobmanager的内存中。基于内存的statebackend在<font color='red'>生产环境下不建议使用</font>。

**适用场景**

- 本地开发调试测试。

**注意点**

- State存储在JobManager的内存中
- 每个State默认5M，可通过MemoryStateBackend构造函数调整
- 每个State的不能超过Akka Frame大小

##### FsStateBackend

State数据保存在taskmanager的内存中，执行checkpoint的时候，会把state的快照数据保存到配置的文件系统中，可以使用hdfs等分布式文件系统。

**适用场景**

- 适用于处理大状态，长窗口，或大键值状态的有状态处理任务。
- FsStateBackend 非常适合用于高可用方案。

**注意点**

- State数据首先会被存在TaskManager的内存
- State大小不能超过TM内存
- TM异步将State数据写入外部存储

##### 存储结构

![1560255197616](/images/1560255197616.png)

#### RocksDBBackend

RocksDBStateBackendRocksDB跟上面的都略有不同，它会在本地文件系统中维护状态，state会直接写入本地rocksdb中。同时RocksDB需要配置一个远端的filesystem。uri（一般是HDFS），在做checkpoint的时候，会把本地的数据直接复制到filesystem中。Fail over的时候从filesystem中恢复到本地。RocksDB克服了state受内存限制的缺点，同时又能够持久化到远端文件系统中，比较适合在生产中使用。

![1560255246670](/images/1560255246670.png)

**基于RocksDB的增量Checkpoint**

![1560255278586](/images/1560255278586.png)

**适用场景**

- 最适合用于处理大状态，长窗口，或大键值状态的有状态处理任务。
- RocksDBStateBackend 非常适合用于高可用方案。
- RocksDBStateBackend 是目前唯一支持增量 checkpoint 的后端。增量 checkpoint 非常使用于超大状态的场景。

**注意点**

- 总State大小仅限于磁盘大小，不受内存限制
- RockDBStateBackend也需要配置外部文件系统，集中保存State
- RocksDB的JNI API基于byte[]，单key和单Value的大小不能超过2^31
- 对于使用具有合并操作的状态的应用程序，例如 ListState，随着时间可能会累积到超过 2^31 字节大小，这将会导致在接下来的查询中失败。

### State动态扩展

#### OperatorState扩展



**ListState**

并发度在改变的时候，会将并发上的每个List都取出，然后把这些List合并到一个新的List,然后根据元素的个数在均匀分配给新的Task;

如下图所示，如果刚开始operator的并行度为3，那么在重新分区之后 ，会将所有的元素平均分配给每一个state。

![img](/images/20190529120446518.png)

**UnionListState**

比于ListState更加灵活，把划分的方式交给用户去做，当改变并发的时候，会将原来的List拼接起来。然后不做划分，直接交给用户；

如下图中，并行度为3，当重新分区后会将之前的所有的state的元素分配给每一个分区后的state。

![img](/images/2019052912075149.png)

**BroadcastState**

如大表和小表做Join时，小表可以直接广播给大表的分区，在每个并发上的数据都是完全一致的。做的更新也相同，当改变并发的时候，把这些数据COPY到新的Task即可

![img](/images/2019052912114988.png)

#### KeyedState扩展

基于Key-Group，每个Key隶属于唯一Key-Group

Key-Group分配给Task实例，每个Task至少有1个Key-Group

Key-Group数量取决于最大并行度(MaxParallism)

KeyedStream并发的上限是Key-Group的数量，等于MaxParallism

**KeyGroup分配算法**

numberOfKeyGroups = maxParallelism

hash = hash(key)

KG = hash%numberOfKerGroups //key所属的KeyGroup是确定的

Subtask = KG*parallelism/numberOfKeyGroups

## Flink Checkpoint与Savepoint

**Checkpoint的作用**

Checkpoint是Flink实现容错机制最核心的功能，它能够根据配置周期性地基于Stream中各个Operator的状态来生成Snapshot，从而将这些状态数据定期持久化存储下来，当Flink程序一旦意外崩溃时，重新运行程序时可以有选择地从这些Snapshot进行恢复，从而修正因为故障带来的程序数据状态中断。

**Savepoint的作用**



**Checkpoint与Savepoint的异同**

![1560255515671](/images/1560255515671.png)



### Checkpoint

#### Checkpoint配置

默认情况下，如果设置了Checkpoint选项，则Flink只保留最近成功生成的1个Checkpoint，而当Flink程序失败时，可以从最近的这个Checkpoint来进行恢复。但是，如果我们希望保留多个Checkpoint，并能够根据实际需要选择其中一个进行恢复，这样会更加灵活，比如，我们发现最近4个小时数据记录处理有问题，希望将整个状态还原到4小时之前。

**配置文件**

![1560255624546](/images/1560255624546.png)

**使用API**

![1560255664324](/images/1560255664324.png)

#### Checkpoint保留策略

**默认情况下，checkpoint 不会被保留**，取消程序时即会删除他们，但是可以通过配置保留定期检查点，根据配置
当作业失败或者取消的时候 ，不会自动清除这些保留的检查点 。

**通过API配置**

```java
CheckpointConfig config = env.getCheckpointConfig(); config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

**ExternalizedCheckpointCleanup 可选项如下**

- ​	**ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION：** 取消作业时保留检查点。请注意，在这种情况下，您必须在取消后手动清理检查点状态。
- ​	**ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION：** 取消作业时删除检查点。**只有在作业失败时，检查点状态才可用**。

#### Checkpoint 保存目录结构

![1560255827180](/images/1560255827180.png)

其中倒数第二层目录是JobID，最后一层目录是CheckpointID。

#### Flink Checkpoint可视化

**Checkpoint总体视图**

![1560255896578](/images/1560255896578.png)

**Checkpoint清单**

![1560255903376](/images/1560255903376.png)

### Savepoint

Savepoint可以视为一个 Operator ID -> State的Map，对于每一个有状态的Operator，Key是Operator
ID，Value是Operator的State，如下图所示：

![1560301157227](/images/1560301157227.png)

**物理目录存储结构**

![1560301175095](/images/1560301175095.png)

如上面列出的HDFS路径中，11bbc5是Flink Job ID字符串前6个字符，后面bd967f90709b是随机生成的字符串，然后savepoint-11bbc5-bd967f90709b作为存储此次Savepoint数据的根目录，最后savepoint-11bbc5-bd967f90709b目录下面_metadata文件包含了Savepoint的元数据信息，其中序列化包含了savepoint-11bbc5-bd967f90709b目录下面其它文件的路径，这些文件内容都是序列化的状态信息。

### 作业恢复

**恢复方式**

- **Externalized Checkpoint** 

  Checkpoint完成时，在用户给定的外部持久化存储保存。

  当作业Failed（或者Cancled）时，外部存储的Checkpoint会保留下来。

  用户在恢复时需要提供用于恢复的作业状态的Checkpoint路径。

- **Savepoint**

  用户通过命令触发，由用户管理其创建于删除。

  标准化格式存储，允许作业升级或者配置变更

  用户在恢复时需要提供用于恢复作业状态的savepoint路径

#### 从Checkpoint恢复

**自动****Checkpoint****恢复**

| **重启策略**     | **重启策略值**   | **说明**                                                     |
| --------------------- | --------------------- | ------------------------------------------------------------ |
| **Fixed delay**  | **fixed-delay**  | 固定延迟重启策略会尝试一个给定的次数来重启Job，如果超过了最大的重启次数，Job最终将失败。在连续的两次重启尝试之间，重启策略会等待一个固定的时间。默认**Integer.MAX_VALUE**次。 |
| **Failure rate** | **failure-rate** | 失败率重启策略在Job失败后会重启，但是超过失败率后，Job会最终被认定失败。在两个连续的重启尝试之间，重启策略会等待一个固定的时间。 |
| **No restart**   | **None**         | 失败不重启                                                   |

**手动Checkpoint恢复**

因为 flink checkpoint 目录 分别对应的是 jobId ， 每通过 flink run 方式 / 页面提交方式 都会重新生成 jobId ，那么如何通过checkpoint 恢复 失败任务或者重新执行保留时间点的 任务？

flink 提供了 在启动 之时 通过设置 -s 参数指定checkpoint 目录 , 让新的jobId 读取该checkpoint 元文件信息和状态信息 ，从而达到指定时间节点启动 job 。

启动方式如下 :

```powershell
./bin/flink -s /flink/checkpoints/0171897fa809692093b4a9b223cb35e4/chk-50/_metadata -p @Parallelism -c @Mainclass @jar 
```

#### **从Savepoint恢复**

##### 从Savepoint恢复需要考虑的点

1.**Operator的顺序改变**

如果对应的UID没变，则可以恢复，如果对应的UID变了则恢复失败。

2.**Job中添加了新的Operator**

如果是无状态Operator，没有影响，可以正常恢复，如果是有状态的Operator，跟无状态的Operator一样处理。

3.**从Job中删除了一个有状态的Operator**

默认需要恢复Savepoint中所记录的所有的Operator的状态，删除了一个有状态的Operator，从Savepoint恢复的时候被删除的OperatorID找不到，所以会报错，可以通过在命令中添加--allowNonRestoredState (short: -n)跳过无法恢复的Operator。

4.**添加和删除无状态的Operator**

如果手动设置了UID，则可以恢复，Savepoint中不记录无状态的Operator，如果是自动分配的UID，那么有状态Operator的UID可能会变（flink使用一个单调递增的计数器生成UID，DAG改版，计数器极有可能会变），很有可能恢复失败。

5.**恢复的时候调整并行度**

Flink1.2.0及以上版本,如果没有使用作废的API，则没问题，1.2.0以下版本需要首先升级到1.2.0，才可以。

#### 恢复时的时间问题

例如从kafka中消费数据，每1分钟统计一次结果，进行job或者flink升级的时候，停了3个小时，使用Savepoint进行恢复的时候，可能kafka中已经累积了3个小时的数据，使用事件时间可以保证最终的处理结果是正确的一致的。如果使用的是处理时间，在这三个小时内的积压的数据，可能会在10分钟之内处理完毕，这3个小时内的统计结果是0，10分钟之内的统计结果暴涨了几十几百倍。

<font color='red'>**所以如果需要进行恢复、升级，最好使用事件时间，而不是处理时间。**</font>

#### 恢复时修改并行度

如下图所示并行度从 3 -> 4，利用KeyGroup修改并发。

![1560302008548](/images/1560302008548.png)

## Checkpoint源码解读

### Checkpoint生命周期

#### 触发Checkpoint

**CheckpointCoordinator**负责协调Checkpoint，如左侧所示在提交Job的时候会调用startCheckpointScheduler方法。

启动之后，就会以设定好的频率调用triggerCheckPoint()方法。

Checkpoint是从SourceOperator触发，然后依次向下游触发。

![1560303044681](/images/1560303044681.png)

#### 触发Checkpoint过程

•检查符合触发checkpoint的条件，例如如果禁止了周期性的checkpoint，尚未达到触发checkpoint的最小间隔等等，就直接return

•检查是否所有需要checkpoint和需要响应checkpoint的ACK（ack涉及到checkpoint的两阶段提交，后面会讲）的task都处于running状态，否则return

•如果都符合，那么执行checkpointID = checkpointIdCounter.getAndIncrement();以生成一个新的id，然后生成一个PendingCheckpoint。PendingCheckpoint是一个启动了的checkpoint，但是还没有被确认。等到所有的task都确认了本次checkpoint，那么这个checkpoint对象将转化为一个CompletedCheckpoint。

•定义一个超时callback，如果checkpoint执行了很久还没完成，就把它取消

•触发MasterHooks，用户可以定义一些额外的操作，用以增强checkpoint的功能（如准备和清理外部资源）

**核心触发逻辑：**

```java
@VisibleForTesting
public PendingCheckpoint triggerCheckpoint(
			long timestamp,
			CheckpointProperties props,
			@Nullable String externalSavepointLocation,
			boolean isPeriodic,
			boolean advanceToEndOfTime) throws CheckpointException {
    // 向Task发送消息触发Checkpoint
    for (Execution execution: executions) {
       if (props.isSynchronous()) {
           //触发Savepoint
          execution.triggerSynchronousSavepoint(checkpointID, timestamp, 		                       checkpointOptions, advanceToEndOfTime);
       } else {
           //触发Checkpoint
          execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
       }
    }
}
```

**JobManager向Task发送Checkpoint消息**

Execution的triggerCheckpoint方法，一个execution就是一个executionVertex的实际执行者。我们看一下这个方法：

```java
public void triggerCheckpoint(long checkpointId, long timestamp, CheckpointOptions checkpointOptions) {
   triggerCheckpointHelper(checkpointId, timestamp, checkpointOptions, false);
}

private void triggerCheckpointHelper(long checkpointId, long timestamp, CheckpointOptions checkpointOptions, boolean advanceToEndOfEventTime) {

		final CheckpointType checkpointType = checkpointOptions.getCheckpointType();
		if (advanceToEndOfEventTime && !(checkpointType.isSynchronous() && checkpointType.isSavepoint())) {
			throw new IllegalArgumentException("Only synchronous savepoints are allowed to advance the watermark to MAX.");
		}

		final LogicalSlot slot = assignedResource;

		if (slot != null) {
            //TaskManagerGateway是用来跟远程的TaskManager通讯的组件
			final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();
			//向TaskManager发送消息触发Checkpoint，TaskManager收到消息进行相应的处理，
            //触发Task的checkpoint操作
			taskManagerGateway.triggerCheckpoint(attemptId, getVertex().getJobId(), checkpointId, timestamp, checkpointOptions, advanceToEndOfEventTime);
		} else {
			LOG.debug("The execution has no slot assigned. This indicates that the execution is no longer running.");
		}
	}

```

**Task层面的Checkpoint准备**

Task类中的部分，该类创建了一个CheckpointMetaData的对象，并且生成了一个Runable匿名类用于执行checkpoint，然后以异步的方式触发了该Runable。

代码如下：

```java
public void triggerCheckpointBarrier(
			final long checkpointID,
			final long checkpointTimestamp,
			final CheckpointOptions checkpointOptions,
			final boolean advanceToEndOfEventTime) {
        
		final AbstractInvokable invokable = this.invokable;
		final CheckpointMetaData checkpointMetaData = new CheckpointMetaData(checkpointID, checkpointTimestamp);
        //检查状态是否为Running
		if (executionState == ExecutionState.RUNNING && invokable != null) {

			// build a local closure
			final String taskName = taskNameWithSubtask;
			final SafetyNetCloseableRegistry safetyNetCloseableRegistry =
				FileSystemSafetyNet.getSafetyNetCloseableRegistryForThread();

			Runnable runnable = new Runnable() {
				@Override
				public void run() {
					// set safety net from the task's context for checkpointing thread
					LOG.debug("Creating FileSystem stream leak safety net for {}", Thread.currentThread().getName());
					FileSystemSafetyNet.setSafetyNetCloseableRegistryForThread(safetyNetCloseableRegistry);

					try {
						boolean success = invokable.triggerCheckpoint(checkpointMetaData, checkpointOptions, advanceToEndOfEventTime);
						if (!success) {
							checkpointResponder.declineCheckpoint(
									getJobID(), getExecutionId(), checkpointID,
									new CheckpointDeclineTaskNotReadyException(taskName));
						}
					}
					catch (Throwable t) {
						if (getExecutionState() == ExecutionState.RUNNING) {
							failExternally(new Exception(
								"Error while triggering checkpoint " + checkpointID + " for " +
									taskNameWithSubtask, t));
						} else {
							LOG.debug("Encountered error while triggering checkpoint {} for " +
								"{} ({}) while being not in state running.", checkpointID,
								taskNameWithSubtask, executionId, t);
						}
					} finally {
						FileSystemSafetyNet.setSafetyNetCloseableRegistryForThread(null);
					}
				}
			};
			executeAsyncCallRunnable(
					runnable,
					String.format("Checkpoint Trigger for %s (%s).", taskNameWithSubtask, executionId),
					checkpointOptions.getCheckpointType().isSynchronous());
		}
		else {
			LOG.debug("Declining checkpoint request for non-running task {} ({}).", taskNameWithSubtask, executionId);

			// send back a message that we did not do the checkpoint
			checkpointResponder.declineCheckpoint(jobId, executionId, checkpointID,
					new CheckpointDeclineTaskNotReadyException(taskNameWithSubtask));
		}
	}
```

代码里的invokable事实上就是我们的StreamTask了。Task类实际上是将checkpoint委托给了更具体的类去执行，而StreamTask也将委托给更具体的类，直到业务代码。 

**StreamTask类中的CheckPoint的过程**

StreamTask的概要逻辑：

- 如果task还在运行，那就可以进行checkpoint。方法是先向下游所有出口广播一个Barrier，然后触发本task的State保存。
- 如果task结束了，那我们就要通知下游取消本次checkpoint，方法是发送一个CancelCheckpointMarker，这是类似于Barrier的另一种消息。

> 注意，从这里开始，整个执行链路上开始出现Barrier，可以和前面讲Fault Tolerant原理的地方结合看一下。 



```java
private boolean performCheckpoint(
			CheckpointMetaData checkpointMetaData,
			CheckpointOptions checkpointOptions,
			CheckpointMetrics checkpointMetrics,
			boolean advanceToEndOfTime) throws Exception {
		...
		synchronized (lock) {
			if (isRunning) {			
			...
			//下边三个步骤，看起来像是个原子操作实际上在实现的时候，
			//checkpoint barrier会尽可能的快速向下发送，避免下游等待checkpoint对齐消耗太多的时间,
			//影响数据流吞吐

			// 步骤1: Checkpoint前准备，实现中尽量的减少此处的处理
			operatorChain.prepareSnapshotPreBarrier(checkpointId);

			// 步骤2: 向下游发送Checkpoint barrier
			operatorChain.broadcastCheckpointBarrier(
					checkpointId,
					checkpointMetaData.getTimestamp(),
					checkpointOptions);

			// 步骤3: 生成状态快照，尽量采用异步模式，避免影响数据流处理
			checkpointState(checkpointMetaData, checkpointOptions, checkpointMetrics);

			result = true;
		}
		else {
			//告诉下游取消Checkpoint
			final CancelCheckpointMarker message = 
			new CancelCheckpointMarker(checkpointMetaData.getCheckpointId());
			Exception exception = null;

			for (RecordWriter<SerializationDelegate<StreamRecord<OUT>>> recordWriter : recordWriters) {
				...
				recordWriter.broadcastEvent(message);
				...
			}
		}
	}
	...
}
```
**checkpointState**生成快照

​		调用checkpointStreamOperator ，触发StreamOperator执行具体checkpoint的动作。

​		注册回调，当执行完checkpoint后，之后向JobManager发出CompletedCheckPoint消息，这也是fault tolerant两阶段提交的一部分 。

```java
private static final class CheckpointingOperation {

	public void executeCheckpointing() throws Exception {
		...
		try {
			//调用StreamOperator进行snapshot的入口
			for (StreamOperator<?> op : allOperators) {
				checkpointStreamOperator(op);
			}

			// we are transferring ownership over snapshotInProgressList for cleanup to the thread, active on submit
			AsyncCheckpointRunnable asyncCheckpointRunnable = new AsyncCheckpointRunnable(
				owner,
				operatorSnapshotsInProgress,
				checkpointMetaData,
				checkpointMetrics,
				startAsyncPartNano);

			owner.cancelables.registerCloseable(asyncCheckpointRunnable);
			//注册runnable，执行完checkpoint之后，向JM发出CompletedCheckpoint
			owner.asyncOperationsThreadPool.submit(asyncCheckpointRunnable);

		} catch (Exception ex) {
			// 清理释放资源
			for (OperatorSnapshotFutures operatorSnapshotResult : operatorSnapshotsInProgress.values()) {
				...
				 operatorSnapshotResult.cancel();
				...	
				}
			}
		
		}
	}
	...
}
```

​	

**StreamOperator中保存State和Barrier传递**

接着上一节的checkpointStreamOperator(op)方法往下讲。 顺便，前面也提到了，在进行checkpoint之前，operator初始化时，会执行一个initializeState方法，在该方法中，如果task是从失败中恢复的话，其保存的state也会被restore进来。

传递barrier是在进行本operator的statesnapshot之前完成的，我们先来看看其逻辑，其实和传递一条数据是类似的，就是生成一个CheckpointBarrier对象，然后向每个streamOutput写进去：

```java
public void broadcastCheckpointBarrier(long id, long timestamp, CheckpointOptions checkpointOptions) throws IOException {
   CheckpointBarrier barrier = new CheckpointBarrier(id, timestamp, checkpointOptions);
   for (RecordWriterOutput<?> streamOutput : streamOutputs) {
      streamOutput.broadcastEvent(barrier);
   }
}
```

**Operator捕获Barrier的过程**

在flink的流中，barrier流过时会触发checkpoint。捕获barrier的过程其实就是处理input数据的过程，对应着StreamInputProcessor.processInput()方法。处理barrier的过程在这段代码里没有体现，因为被包含在了StreamTaskInput.pollNextNullable()方法中。

```java
@Internal
public final class StreamTaskNetworkInput implements StreamTaskInput {
	@Override
	@Nullable
	public StreamElement pollNextNullable() throws Exception {

		while (true) {
			// get the stream element from the deserializer
			if (currentRecordDeserializer != null) {
				DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate);
				if (result.isBufferConsumed()) {
					currentRecordDeserializer.getCurrentBuffer().recycleBuffer();
					currentRecordDeserializer = null;
				}
				
				if (result.isFullRecord()) {
					return deserializationDelegate.getInstance();
				}
			}

			Optional<BufferOrEvent> bufferOrEvent = barrierHandler.pollNext();
			if (bufferOrEvent.isPresent()) {
				processBufferOrEvent(bufferOrEvent.get());
			} else {
				if (barrierHandler.isFinished()) {
					isFinished = true;
					checkState(barrierHandler.isAvailable().isDone(), "Finished BarrierHandler should be available");
					if (!barrierHandler.isEmpty()) {
						throw new IllegalStateException("Trailing data in checkpoint barrier handler.");
					}
				}
				return null;
			}
		}
	}
}
```

**Barrier具体的处理BarrierBuffer.pullNext()**

处理barrier也是个麻烦事，回想之前barrier的原理图，一个opertor必须收到从每个inputchannel发过来的同一序号的barrier之后才能发起本节点的checkpoint，如果有的channel的数据处理的快了，那该barrier后的数据还需要缓存起来，如果有的inputchannel被关闭了，那它就不会再发送barrier过来了。

当收到全部的barrier之后，就会触发notifyCheckpoint()，该方法又会调用StreamTask的triggerCheckpoint。

```java
@Override
public Optional<BufferOrEvent> pollNext() throws Exception {
   while (true) {
    ...
      else if (bufferOrEvent.getEvent().getClass() == CheckpointBarrier.class) {
         if (!endOfStream) {
            // process barriers only if there is a chance of the checkpoint completing
            processBarrier((CheckpointBarrier) bufferOrEvent.getEvent(), bufferOrEvent.getChannelIndex());
         }
      }
      else if (bufferOrEvent.getEvent().getClass() == CancelCheckpointMarker.class) {
         processCancellationBarrier((CancelCheckpointMarker) bufferOrEvent.getEvent());
      }
      else {
         if (bufferOrEvent.getEvent().getClass() == EndOfPartitionEvent.class) {
            processEndOfPartition();
         }
         return next;
      }
     ...
   }
}
```

#### Checkpoint保存State

1.Checkpoint的过程从Source算子开始，首先向下游发送barrier，然后自己进行checkpoint。

2.下游算子收到Barrier，触发本Operator的checkpoint，并继续想下游发送barrier。

**注意：**Operator只是将state保存到了StateBackend中，并没有进行持久化的操作。持久化的操作在AbstractStreamOperator中，由flink统一负责持久化。 

```java
//AbstractStreamOperator.java中的snapshotState负责将State持久化到外部存储中
...
//operator State保存
if (null != operatorStateBackend) {
				snapshotInProgress.setOperatorStateManagedFuture(
					operatorStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
}
//keyed State保存
if (null != keyedStateBackend) {
	snapshotInProgress.setKeyedStateManagedFuture(
		keyedStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
}
...

```

operatorStateBackend是怎么保存状态的呢？

•首先把各个算子的state做了一份深拷贝；

•然后以异步的方式执行了一个内部类的runnable，该内部类的run方法实现了一个模版方法，首先打开stream，然后写入数据，然后再关闭stream。

#### TM报告Checkpointpoint完成

当一个operator保存完checkpoint数据后，就会启动一个异步对象AsyncCheckpointRunnable，用以报告该检查点已完成，其具体逻辑在reportCompletedSnapshotStates中。这个方法把任务又最终委托给了RpcCheckpointResponder这个类：

```java

//RpcCheckpointResponder
@Override
public void acknowledgeCheckpoint(
      JobID jobID,
      ExecutionAttemptID executionAttemptID,
      long checkpointId,
      CheckpointMetrics checkpointMetrics,
      TaskStateSnapshot subtaskState) {
   //向CheckpointCoordinator发送消息报告Checkpoint完成
   checkpointCoordinatorGateway.acknowledgeCheckpoint(
      jobID,
      executionAttemptID,
      checkpointId,
      checkpointMetrics,
      subtaskState);
}
```

RpcCheckpointResponder逻辑是通过rpc的方式远程调JobManager的相关方法完成报告事件，底层通过akka实现的。

#### JobMaster响应完成Checkpoint事件

JobMaster转身就把任务又rpc给了CheckpointCoordinator.receiveAcknowledgeMessage()方法。

```java
//JobMaster.java,将事件转交给CheckpointCoordinator
@Override
public void acknowledgeCheckpoint(
    final JobID jobID,
    final ExecutionAttemptID executionAttemptID,
    final long checkpointId,
    final CheckpointMetrics checkpointMetrics,
    final TaskStateSnapshot checkpointState) {

    schedulerNG.acknowledgeCheckpoint(jobID, executionAttemptID, checkpointId, checkpointMetrics, checkpointState);
}
```

##### CheckpointCoordinator响应完成Checkpoint事件

1.之前提到，coordinator在触发checkpoint时，生成了一个PendingCheckpoint，保存了所有operator的id。

2.当PendingCheckpoint收到一个operator的完成checkpoint的消息时，它就把这个operator从未完成checkpoint的节点集合移动到已完成的集合。当所有的operator都报告完成了checkpoint时，CheckpointCoordinator会触发completePendingCheckpoint()方法，该方法做了以下事情：

- 把pendinCgCheckpoint转换为CompletedCheckpoint
- 把CompletedCheckpoint加入已完成的检查点集合，并从未完成检查点集合删除该检查点
- 再度向各个operator发出rpc，通知该检查点已完成

```java
//响应消息，判断checkpoint是否完成或者丢弃
public boolean receiveAcknowledgeMessage(AcknowledgeCheckpoint message, String taskManagerLocationInfo) throws CheckpointException {
    ...
    switch (checkpoint.acknowledgeTask(message.getTaskExecutionId(), message.getSubtaskState(), message.getCheckpointMetrics())) {
        case SUCCESS:

            if (checkpoint.isFullyAcknowledged()) {
                completePendingCheckpoint(checkpoint);
            }
            break;
     ...
    }
}

//向Task发送checkpoint完成消息
private void completePendingCheckpoint(PendingCheckpoint pendingCheckpoint) throws CheckpointException {
	...
	for (ExecutionVertex ev : tasksToCommitTo) {
		Execution ee = ev.getCurrentExecutionAttempt();
		if (ee != null) {
			ee.notifyCheckpointComplete(checkpointId, timestamp);
		}
	}
}

```

# Flink生成Graph

## Flink图总览

![1560325980049](/images/1560325980049.png)

JobGraph 之上除了 StreamGraph 还有 OptimizedPlan。OptimizedPlan 是由 BatchAPI 转换而来的。StreamGraph 是由 StreamAPI 转换而来的。

**为什么API不直接转换成JobGraph**？

因为，Batch和Stream的图结构和优化方法有很大的区别，比如 Batch有很多执行前的预分析用来优化图的执行，而这种优化并不普适于Stream，所以通过OptimizedPlan 来做 Batch的优化会更方便和清晰，也不会影响Stream。

### FlinkStreaming 4层图结构

![img](https://upload-images.jianshu.io/upload_images/5501600-416016b40d269ab8.png?imageMogr2/auto-orient/)

- **StreamGraph**

  对用户逻辑的映射，从DataStream API生成的最初的图。

  **其核心对象如下**

  1. **StreamNode**：用来代表 operator 的类，并具有所有相关的属性，如并发度、入边和出边等。

  2. **StreamEdge**：表示连接两个StreamNode的边。

- **JobGraph**

  在StreamGraph的基础上进行了一些优化，比如把一部分操作串成chain以提高效率。流和批的统一是JobGraph层面上实现的。

  **其核心对象如下**

  1. **JobVertex**：经过优化后符合条件的多个StreamNode可能会chain在一起生成一个JobVertex，即一个
      JobVertex包含一个或多个operator，JobVertex的输入是JobEdge，输出是IntermediateDataSet。

  2. **IntermediateDataSet**：表示JobVertex的输出，即经过operator处理产生的数据集。producer是
      JobVertex，consumer是JobEdge。

  3. **JobEdge**：代表了job graph中的一条数据传输通道。source 是 IntermediateDataSet，target 是 JobVertex。即数据通过JobEdge由IntermediateDataSet传递给目标JobVertex。

- **ExecutionGraph**

  为了调度存在的，加入了并行处理的概念。而在此基础上真正执行的是Task及其相关结构。

  JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。

  **其核心对象如下**

  1. **ExecutionJobVertex**：和JobGraph中的JobVertex一一对应。每一个ExecutionJobVertex都有和并发度一样多的 ExecutionVertex。

  2. **ExecutionVertex**：**表示****ExecutionJobVertex****的其中一个并发子任务**，输入是ExecutionEdge，输出是IntermediateResultPartition。

  3. **IntermediateResult**：和JobGraph中的IntermediateDataSet一一对应。**一个****IntermediateResult****包含多个****IntermediateResultPartition****，其个数等于该****operator****的并发度。** 

  4. **IntermediateResultPartition**：**表示****ExecutionVertex****的一个输出分区**，producer是ExecutionVertex，consumer是若干个ExecutionEdge。

  5. **ExecutionEdge**：表示ExecutionVertex的输入，source是IntermediateResultPartition，target是ExecutionVertex。**source****和****target****都只能是一个。** 

  6. **Execution**：是执行一个 ExecutionVertex 的一次尝试。当发生故障或者数据需要重算的情况下 ExecutionVertex 可能会有多个 ExecutionAttemptID。一个 Execution 通过 ExecutionAttemptID 来唯一标识。JM和TM之间关于 task 的部署和 task status 的更新都是通过 ExecutionAttemptID 来确定消息接受者。

- **物理执行图**

  并非flink的数据结构，而是程序开始执行后，各个task分布在不同的节点上，所形成的物理上的关系表示。JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。

  **其核心对象如下**

  1. **Task**：Execution被调度后在分配的 TaskManager 中启动对应的 Task。Task 包裹了具有用户执行逻辑的 operator。

  2. **ResultPartition**：代表由一个Task的生成的数据，和ExecutionGraph中的IntermediateResultPartition一一对应。

  3. **ResultSubpartition**：是ResultPartition的一个子分区。每个ResultPartition包含多个ResultSubpartition，**其数目要由下游消费** **Task** **数和** **DistributionPattern** **来决定。** 

  4. **InputGate**：代表Task的输入封装，和JobGraph中JobEdge一一对应。每个InputGate消费了一个或多个的ResultPartition。

  5. **InputChannel**：每个InputGate会包含一个以上的InputChannel，和ExecutionGraph中的ExecutionEdge一一对应，也和ResultSubpartition一对一地相连，即一个InputChannel接收一个ResultSubpartition的输出。

​	从JobGraph的图里可以看到，数据从上一个operator流到下一个operator的过程中，上游作为生产者提供了IntermediateDataSet，而下游作为消费者需要JobEdge。事实上，JobEdge是一个通信管道，连接了上游生产的dataset和下游的JobVertex节点。

​		在JobGraph转换到ExecutionGraph的过程中，主要发生了以下转变： 

- 加入了并行度的概念，成为真正可调度的图结构
- 生成了与JobVertex对应的ExecutionJobVertex，ExecutionVertex，与IntermediateDataSet对应的IntermediateResult和IntermediateResultPartition等，并行将通过这些类实现。

​	ExecutionGraph已经可以用于调度任务。我们可以看到，flink根据该图生成了一一对应的Task，每个task对应一个ExecutionGraph的一个Execution。Task用InputGate、InputChannel和ResultPartition对应了上面图中的IntermediateResult和ExecutionEdge。

## 生成StreamGraph



### 过程

在调用StreamExecutionEnvirment.execute()的时候开始触发StreamGraph的构建。

**入口**

```java
//StreamExecutionEnvironment.java

@Internal
public StreamGraph getStreamGraph() {
   if (transformations.size() <= 0) {
      throw new IllegalStateException("No operators defined in streaming topology. Cannot execute.");
   }
   return StreamGraphGenerator.generate(this, transformations);
}

```

**真正的构建过程**

```java
//StreamGraphGenerator.java中负责具体的StreamGraph的生成

public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
	return new StreamGraphGenerator(env).generateInternal(transformations);
}

//实际执行StreamGraph的生成
private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
	for (StreamTransformation<?> transformation: transformations) {
		transform(transformation);
	}
	return streamGraph;
}
```

遍历`transformations`集合，并对其每一个`StreamTransformation`调用`transform()`方法。

```java
Collection<Integer> transformedIds;
if (transform instanceof OneInputTransformation<?, ?>) {
   transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
   transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
} else if (transform instanceof SourceTransformation<?>) {
   transformedIds = transformSource((SourceTransformation<?>) transform);
} else if (transform instanceof SinkTransformation<?>) {
   transformedIds = transformSink((SinkTransformation<?>) transform);
} else if (transform instanceof UnionTransformation<?>) {
   transformedIds = transformUnion((UnionTransformation<?>) transform);
} else if (transform instanceof SplitTransformation<?>) {
   transformedIds = transformSplit((SplitTransformation<?>) transform);
} else if (transform instanceof SelectTransformation<?>) {
   transformedIds = transformSelect((SelectTransformation<?>) transform);
} else if (transform instanceof FeedbackTransformation<?>) {
   transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
} else if (transform instanceof CoFeedbackTransformation<?>) {
   transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
} else if (transform instanceof PartitionTransformation<?>) {
   transformedIds = transformPartition((PartitionTransformation<?>) transform);
} else if (transform instanceof SideOutputTransformation<?>) {
   transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
} else {
   throw new IllegalStateException("Unknown transformation: " + transform);
}
```

从上述代码可以看出，针对具体的某一种类型的`StreamTransformation`，会调用其相应的`transformXXX()`函数进行转换。`transformXXX()`首先会对`transform`的上游`transform`进行递归转换，确保上游的都已经完成了转化。然后通过`addOperator()`方法构造出`StreamNode`，通过`addEdge()`方法与上游的`transform`进行连接，构造出`StreamEdge`。

**单输入实体StreamTransformation的转换示例**

```java
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {

	Collection<Integer> inputIds = transform(transform.getInput());

	// 防止重复转换，如果已经转换过了则直接返回转换的结果
	if (alreadyTransformed.containsKey(transform)) {
		return alreadyTransformed.get(transform);
	}
	//确定共享slot组
	String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);
	//添加StreamNode到StreamGraph中
	streamGraph.addOperator(transform.getId(),
			slotSharingGroup,
			transform.getCoLocationGroupKey(),
			transform.getOperatorFactory(),
			transform.getInputType(),
			transform.getOutputType(),
			transform.getName());
	
	if (transform.getStateKeySelector() != null) {
		TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
		streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
	}
	//设定并行度
	streamGraph.setParallelism(transform.getId(), transform.getParallelism());
	streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());
	//添加StreamEdge，建立StreamNode之间的关联关系
	for (Integer inputId: inputIds) {
		streamGraph.addEdge(inputId, transform.getId(), 0);
	}

	return Collections.singleton(transform.getId());
}
```

1. 存储这个OneInputTransformation的上游Transformation的id,方便构造边，在这里递归，确保所有的上游Transformation都已经转化。

2. 确定共享的Slot组。

3. 添加Operator到StreamGraph中。

4. 设置StateKeySelector。

5. 设置并行度、最大并行度。

6. 构造StreamGrap的边。

**虚拟SteamTranform的生成示例**

如下DataStream.keyBy（），虚拟的StreamTransformation生成的时候不会转换为StreamNode，而是添加为虚拟节点。

```java
private <T> Collection<Integer> transformPartition(PartitionTransformation<T> partition) {
    StreamTransformation<T> input = partition.getInput();
    List<Integer> resultIds = new ArrayList<>();
    // 递归对该transformation的直接上游transformation进行转换，获得直接上游id集合
    Collection<Integer> transformedIds = transform(input);
    for (Integer transformedId: transformedIds) {
        int virtualId = StreamTransformation.getNewNodeId();
        // 添加一个虚拟分区节点VirtualPartitionNode，不会生成StreamNode
        streamGraph.addVirtualPartitionNode(transformedId, virtualId, partition.getPartitioner());
        resultIds.add(virtualId);
    }
  return resultIds;
}
```

从`transformPartition`函数的实现可以看出，对`transformPartition`的转换没有生成具体的`StreamNode`和`StreamEdge`，而是通过`streamGraph.addVirtualPartitionNode()`方法添加了一个虚拟节点。当`partition`的下游`transform`添加`edge`时（调用`streamGraph.addEdge()`），会把`partition`信息写入到`StreamEdge`中。

```java
private void addEdgeInternal(Integer upStreamVertexID, Integer downStreamVertexID, int typeNumber, StreamPartitioner<?> partitioner, List<String> outputNames, OutputTag outputTag) {
    // 当上游是sideOutput时，递归调用，并传入sideOutput信息
    if (virtualSideOutputNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualSideOutputNodes.get(virtualId).f0;
        if (outputTag == null) {
            outputTag = virtualSideOutputNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, null, outputTag);
    }
    // 当上游是select时，递归调用，并传入select信息 
    else if (virtualSelectNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualSelectNodes.get(virtualId).f0;
        if (outputNames.isEmpty()) {
            outputNames = virtualSelectNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
    }
    // 当上游是partition时，递归调用，并传入partitioner信息 
    else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
        int virtualId = upStreamVertexID;
        upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
        if (partitioner == null) {
            partitioner = virtualPartitionNodes.get(virtualId).f1;
        }
        addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
    }
    // 不是以上逻辑转换的情况，真正构建StreamEdge 
    else {
        StreamNode upstreamNode = getStreamNode(upStreamVertexID);
        StreamNode downstreamNode = getStreamNode(downStreamVertexID);
        // 没有指定partitioner时，会为其选择forward或者rebalance分区
        if (partitioner == null && upstreamNode.getParallelism() == downstreamNode.getParallelism()) {
            partitioner = new ForwardPartitioner<Object>();
        } else if (partitioner == null) {
            partitioner = new RebalancePartitioner<Object>();
        }
        // 创建StreamEdge，并将该StreamEdge添加到上游的输出，下游的输入
        StreamEdge edge = new StreamEdge(upstreamNode, downstreamNode, typeNumber, outputNames, partitioner, outputTag);
        getStreamNode(edge.getSourceId()).addOutEdge(edge);
        getStreamNode(edge.getTargetId()).addInEdge(edge);
    }
}
```

### 关键设计

#### 关键对象

**StreamTransformation**



**StreamOperator**



#### DataStream API到 StreamTransformation

```java
//DataStream.java

//编写代码是调用的map API
public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {

	TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
			Utils.getCallLocationName(), true);

	return transform("Map", outType, new StreamMap<>(clean(mapper)));
}

@PublicEvolving
public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

	// read the output type of the input Transform to coax out errors about MissingTypeInfo
	transformation.getOutputType();
    //封装成StreamTransformation
	OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
			this.transformation,
			operatorName,
			operator,
			outTypeInfo,
			environment.getParallelism());
    //封装成StreamOperator
	@SuppressWarnings({ "unchecked", "rawtypes" })
	SingleOutputStreamOperator<R> returnStream = new 	                        SingleOutputStreamOperator(environment, resultTransform);
    //注册到StreamExectionEnvirment中，后边生成StreamGraph会用到
	getExecutionEnvironment().addOperator(resultTransform);

	return returnStream;
}

```

> 其他的API调用的转换过程是类似的。

1. 首先，用户代码里定义的UDF会被当作其基类对待，然后交给StreamMap这个operator做进一步包装。事实上，每一个Transformation都对应了一个StreamOperator。
2. 由于map这个操作只接受一个输入，所以再被进一步包装为OneInputTransformation。
3. 最后，将该transformation注册到执行环境中，当执行上文提到的generate方法时，生成StreamGraph图结构。
4. StreamTransformation是个虚拟的概念，不会体现在运行时。在真正的运行时，是由StreamTask（OneInputStreamTask、TwoInputStreamTask、StreamIterationHead、StreamIterationTail）调用StreamOneInputProcessor和StreamTwoInputProcessor执行。

StreamInputProcessor.processInput()从InputGate读取数据，然后调用StreamOperator.processElement处理数据。

#### StreamFormation、StreamOperator、UDF的关系

![1560329270219](/images/1560329270219.png)

#### 实体StreamNode和虚拟StreamNode

并不是每一个 StreamTransformation 都会转换成runtime层中的物理操作。有一些只是逻辑概念，比如union、split/select、partition等。

虚拟的StreamNode不会变成实体的StreamNode，实体的StreamNode往往表达是用户实现的逻辑，此处的功能是不确定的，因业务而异，很可能会生成不同结构的StreamRecord的输出交给下游。而Split、union、partition这些逻辑是很明确的，不会改变StreamRecord，在执行的时候封装到StreamEdge。

如下图所示的转换树所示

![1560332101843](/images/1560332101843.png)



### 总结

 首先，根据用户通过Stream API自定义UDF编写的程序，即在DataStream上做的一系列转换（map、shuffle、window等），我们可以得到StreamTransformation集合。然后通过调用streamGraphGenerator.generate(env, transformations)，遍历transformations集合，并对其每一个StreamTransformation调用transform()方法，构造出StreamNode，并通过StreamEdge与上游的transformation进行连接，此处需要特别注意对逻辑转换（partition等）的处理，最后构造出StreamGraph。

## 生成JobGraph

JobGraph 的责任就是统一Batch和Stream的图，用来描述清楚一个拓扑图的结构，并且做了chaining的优化，chaining普适于Batch和Stream。

### 过程

**入口**

```java
//StreamGraph.java
@Override
public JobGraph getJobGraph(@Nullable JobID jobID) {
	// 迭代job暂时禁用Checkpoint
	return StreamingJobGraphGenerator.createJobGraph(this, jobID);
}
```

```java
//StreamGraphGenerator.java
@Override
public JobGraph getJobGraph(@Nullable JobID jobID) {
	// 迭代job暂时禁用Checkpoint
	return StreamingJobGraphGenerator.createJobGraph(this, jobID);
}

public static JobGraph createJobGraph(StreamGraph streamGraph, @Nullable JobID jobID) {
    return new StreamingJobGraphGenerator(streamGraph, jobID).createJobGraph();
}

```



```java
private JobGraph createJobGraph() {

	// 设置调度模式为Eager，Eager是Stream的调度模式，不同于Batch，
	//即一次性申请所有需要的资源
	jobGraph.setScheduleMode(ScheduleMode.EAGER);

	// Generate deterministic hashes for the nodes in order to identify them across
	// submission iff they didn't change.
	//为每个节点生成确定的hash id作为唯一标识
	Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

	// 为了向后保持兼容为每个节点生成老版本的Hash id
	List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
	for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
		legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
	}

	Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();
	//真正对StreamGrap进行转换，生成JobGraph图
	setChaining(hashes, legacyHashes, chainedOperatorHashes);

	setPhysicalEdges();
	//设置共享slotGroup
	setSlotSharingAndCoLocation();
	//配置checkpoint
	configureCheckpointing();
	//如果有之前的缓存的文件的配置，则重新读入
	JobGraphGenerator.addUserArtifactEntries(streamGraph.getEnvironment().getCachedFiles(), jobGraph);

	//设置执行环境配置
	jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());

	return jobGraph;
}
```

**JobGraph生成的具体逻辑**

从Source节点开始，遍历Source节点，创建OperatorChain

```java
//StreamGraphGenerator.java
private void setChaining(Map<Integer, byte[]> hashes, List<Map<Integer, byte[]>> legacyHashes, Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {
    //从所有的Source节点开始生成OperatorChain
    for (Integer sourceNodeId : streamGraph.getSourceIDs()) {
        createChain(sourceNodeId, sourceNodeId, hashes, legacyHashes, 0, chainedOperatorHashes);
    }
}
```



执行具体的Chain和JobVertex生成、JobEdge的关联、IntermediateResultSet。

```java
private List<StreamEdge> createChain(
		Integer startNodeId,
		Integer currentNodeId,
		Map<Integer, byte[]> hashes,
		List<Map<Integer, byte[]>> legacyHashes,
		int chainIndex,
		Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {

	if (!builtVertices.contains(startNodeId)) {

		List<StreamEdge> transitiveOutEdges = new ArrayList<StreamEdge>();

		List<StreamEdge> chainableOutputs = new ArrayList<StreamEdge>();
		List<StreamEdge> nonChainableOutputs = new ArrayList<StreamEdge>();
		//获取当前节点的出边，判断边是否符合operatorchain的条件，
		//分为两类：chainableOutputs,nonChainableOutputs
		for (StreamEdge outEdge : streamGraph.getStreamNode(currentNodeId).getOutEdges()) {
			if (isChainable(outEdge, streamGraph)) {
				chainableOutputs.add(outEdge);
			} else {
				nonChainableOutputs.add(outEdge);
			}
		}
		//对于chainable的边，递归调用createChain
		//返回值添加到transitiveOutEdges中
		for (StreamEdge chainable : chainableOutputs) {
			transitiveOutEdges.addAll(
					createChain(startNodeId, chainable.getTargetId(), hashes, legacyHashes, chainIndex + 1, chainedOperatorHashes));
		}
		//对于无法chain在一起的边，边的下游节点作为OperatorChain的Head节点，
		//进行递归调用，返回值添加到transitiveOutEdges中
		for (StreamEdge nonChainable : nonChainableOutputs) {
			transitiveOutEdges.add(nonChainable);
			createChain(nonChainable.getTargetId(), nonChainable.getTargetId(), hashes, legacyHashes, 0, chainedOperatorHashes);
		}

		List<Tuple2<byte[], byte[]>> operatorHashes =
			chainedOperatorHashes.computeIfAbsent(startNodeId, k -> new ArrayList<>());

		byte[] primaryHashBytes = hashes.get(currentNodeId);

		for (Map<Integer, byte[]> legacyHash : legacyHashes) {
			operatorHashes.add(new Tuple2<>(primaryHashBytes, legacyHash.get(currentNodeId)));
		}

		chainedNames.put(currentNodeId, createChainedName(currentNodeId, chainableOutputs));
		chainedMinResources.put(currentNodeId, createChainedMinResources(currentNodeId, chainableOutputs));
		chainedPreferredResources.put(currentNodeId, createChainedPreferredResources(currentNodeId, chainableOutputs));
		
		// 如果当前节点是起始节点，则直接创建JobVertex，否则返回一个空的SteamConfig
		StreamConfig config = currentNodeId.equals(startNodeId)
				? createJobVertex(startNodeId, hashes, legacyHashes, chainedOperatorHashes)
				: new StreamConfig(new Configuration());	
		// 将StreamNode中的配置信息序列化到StreamConfig中
		setVertexConfig(currentNodeId, config, chainableOutputs, nonChainableOutputs);
		// 再次判断，如果是chain的起始节点，执行connect()方法，创建JobEdge和IntermediateDataSet；
		//否则将当前节点的StreamConfig添加到chainedConfig中

		if (currentNodeId.equals(startNodeId)) {

			config.setChainStart();
			config.setChainIndex(0);
			config.setOperatorName(streamGraph.getStreamNode(currentNodeId).getOperatorName());
			config.setOutEdgesInOrder(transitiveOutEdges);
			config.setOutEdges(streamGraph.getStreamNode(currentNodeId).getOutEdges());

			for (StreamEdge edge : transitiveOutEdges) {
				connect(startNodeId, edge);
			}

			config.setTransitiveChainedTaskConfigs(chainedConfigs.get(startNodeId));

		} else {
			chainedConfigs.computeIfAbsent(startNodeId, k -> new HashMap<Integer, StreamConfig>());

			config.setChainIndex(chainIndex);
			StreamNode node = streamGraph.getStreamNode(currentNodeId);
			config.setOperatorName(node.getOperatorName());
			chainedConfigs.get(startNodeId).put(currentNodeId, config);
		}

		config.setOperatorID(new OperatorID(primaryHashBytes));
		
		if (chainableOutputs.isEmpty()) {
			config.setChainEnd();
		}
		return transitiveOutEdges;

	} else {
		return new ArrayList<>();
	}
}
```

`setChaining()`会依次对`source`调用`createChain()`方法，该方法会递归调用其下游节点，从而构建出`node chains`。`createChain()`会分析当前节点的出边，根据`Operator Chains`中的条件进行判断`isChainable()`，并将出边分成两类：`chainalbeOutputs`和`noChainableOutputs`，接着分别递归调用自身方法。之后会将`StreamNode`中的配置信息序列化到`StreamConfig`中。如果当前不是`chain`中的子节点，则会构建`JobVertex`和`JobEdge`相连。如果是`chain`中的子节点，则会将`StreamConfig`添加到该`chain`的`chainedConfigs`集合中。

### JobGraph生成示例

**以SocketWindowCount为例**

![img](./images/10507536-92418c0ca2da7a3d.png)

上图是StreamGraph

如上图所示，我们先给`4`个`StreamNode`节点进行编号，`Source`用`1`表示，`Flat Map`用`2`表示，`Trigger Window`用`3`表示，`Sink`用`4`表示；相应地，`3`条`StreamEdge`则分别用`1->2`，`2->3`，`3->4`表示。

**递归调用过程如下**：

- 递归始于`Source`，调用`createChain(1, 1)`，当前节点`1`的出边为`1->2`，不可`chain`，将边`1->2`直接加入`transitiveOutEdges`；
- 然后递归调用`createChain(2, 2)`，当前节点`2`的出边为`2->3`，同样的，不可`chain`，将边`2->3`直接加入`transitiveOutEdges`；
- 继续递归调用`createChain(3, 3)`，当前节点`3`的出边为`3->4`，要注意了，可`chain`，等着将下游`createChain()`的返回值加入`transitiveOutEdges`；
- 此处递归调用`createChain(3, 4)`，当前节点`4`没有出边，递归终止。

**递归结束条件：**

- 当前节点不再有出边集合，即`streamGraph.getStreamNode(currentId).getOutEdges()`为空
- 当前节点已经转换完成，即`builtVertices.contains(startNodeId)`为`false` 

递归调用过程中各种操作以及变量情况一览表如下：

| `creatChain()` | `getOutEdges()` | `chainable` | `nonChainAble` | `transitiveOutEdges` | `JobVertex`    | `connect()` |
| -------------- | --------------- | ----------- | -------------- | -------------------- | -------------- | ----------- |
| `(1, 1)`       | `1->2`          | 无          | `1->2`         | `1->2`               | `JobVertex`    | `Y`         |
| `(2, 2)`       | `2->3`          | 无          | `2->3`         | `2->3`               | `JobVertex`    | `Y`         |
| `(3, 3)`       | `3->4`          | `3->4`      | 无             | 无                   | `JobVertex`    | `Y`         |
| `(3, 4)`       | 无              | 无          | 无             | 无                   | `StreamConfig` | `N`         |



### 关键设计

#### 关键对象

####  OperatorChain

为了更高效地分布式执行，Flink会尽可能地将operator的subtask链接（chain）在一起形成task。每个task在一个线程中执行。将operators链接成task是非常有效的优化：它能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

如下图所示

![1560394979197](/images/1560394979197.png)

`isChainable()`，用来判断`StreamNode chain`，一共有`9`个条件：

- 下游节点的入边为`1` 
-  `StreamEdge`的下游节点对应的算子不为`null` 
-  `StreamEdge`的上游节点对应的算子不为`null` 
-  `StreamEdge`的上下游节点拥有相同的`slotSharingGroup`，默认都是`default` 
- 下游算子的连接策略为`ALWAYS` 
- 上游算子的连接策略为`ALWAYS`或者`HEAD` 
-  `StreamEdge`的分区类型为`ForwardPartitioner` 
- 上下游节点的并行度一致
- 当前`StreamGraph`允许`chain`

### 总结

首先，通过`streamGraph.getSourceIDs()`拿到`source`节点集合，紧接着依次从`source`节点开始遍历，判断`StreamNode Chain`，递归创建`JobVertex`，所以，其真正的处理顺序其实是从`sink`开始的。然后通过`connect()`遍历当前节点的物理出边`transitiveOutEdges`集合，创建`JobEdge`，建立当前节点与下游节点的联系，即`JobVertex`与`IntermediateDataSet`之间。

## 生成ExecutionGraph

与StreamGraph和JobGraph不同，ExecutionGraph并不是在我们的客户端程序生成，而是在服务端（JobManager处）生成的，顺便flink只维护一个JobManager。其入口代码是ExecutionGraphBuilder.buildGraph（...）。

### 过程

```java
//ExecutionGraphBuilder.java

public static ExecutionGraph buildGraph(){
    ...
    //对JobGraph中的JobVertex节点进行拓扑排序，得到List<JobVertex>
    List<JobVertex> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();
    executionGraph.attachJobGraph(sortedTopology);  //构建ExecutionGraph的核心方法
    ...
}
```



由上可知，`attachJobGraph()`方法是构建`ExecutionGraph`图结构的核心方法。

```java
public void attachJobGraph(List<JobVertex> topologiallySorted){
    ...
    //遍历排好序的Job Vertex
    for (JobVertex jobVertex : topologiallySorted) {
        ...
        // 创建ExecutionJobVertex，并添加到ExecutionGraph中
        ExecutionJobVertex ejv = new ExecutionJobVertex(this, jobVertex, 1, rpcCallTimeout, globalModVersion, createTimestamp);
        //将创建的ExecutionJobVertex与前置的IntermediateResult连接起来
        ejv.connectToPredecessors(this.intermediateResults);
        ...
    }
    ...
}
```

`attachJobGraph()`方法主要完成的两件事情：

- 将`JobVertex`封装成`ExecutionJobVertex`
- 把节点通过`ExecutionEdge`连接

### 关键设计

#### 关键对象

- **ExecutionJobVertex**

ExecutionJobVertex方法，用来将一个个JobVertex封装成ExecutionJobVertex，并依次创建ExecutionVertex、Execution、IntermediateResult、IntermediateResultPartition，用于丰富ExecutionGraph。

在ExecutionJobVertex的构造函数中，首先是依据对应的JobVertex的并发度，生成对应个数的ExecutionVertex。其中，一个ExecutionVertex代表着一个ExecutionJobVertex的并发子task。然后是将原来JobVertex的中间结果IntermediateDataSet转化为ExecutionGraph中的IntermediateResult。

**核心的逻辑在ExecutionJobVertex的构造方法中**

1. 是并行度相关的设置    

2. 序列化后的TaskInformation，这个信息很重要，后面deploy的时候会将TaskInformation分发到具体的Task中。 

3. ExecutionVertex列表，按照JobVertex并行度设置      

4. slots创建intermediate results，这是由当前operator的出度确定的，如果当前operator只向下游一个operator输出，则为1。注意一个IntermediateResult包含多个IntermediateResultPartition。

5. Slot sharing和coLocation相关代码     

6. 根据job vertex的并行度，创建对应的ExecutionVertex列表。即，一个JobVertex/ExecutionJobVertex代表的是一个operator，而具体的ExecutionVertex则代表了每一个Task

7. sanity check,检查intermediate result partition和Execution Vertices之间有没有重复的引用

- **ExecutionVertex**

`ExecutionVertex`的构造函数中，首先会创建`IntermediateResultPartition`，并通过`IntermediateResult.setPartition( )`建立`IntermediateResult`和`IntermediateResultPartition`之间的关系；然后生成`Execution`，并配置资源相关。

#### 上下游连接

新创建的`ExecutionJobVertex`调用`ejv.connectToPredecessor()`方法，按照不同的分发策略连接上游，其参数为上游生成的`IntermediateResult`集合。其中，根据`JobEdge`中两种不同的`DistributionPattern`属性，分别调用`connectPointWise()`或者`connectAllToAll( )`方法，创建`ExecutionEdge`，将`ExecutionVertex`和上游的`IntermediateResultPartition`连接起来。

**连接策略**

`DistributionPattern.POINTWISE`策略，该策略用来连接当前`ExecutionVertex`与上游的`IntermediateResultPartition`。首先，获取上游`IntermediateResult`的`partition`数，用`numSources`表示，以及此`ExecutionJobVertex`的并发度，用`parallelism`表示；然后，根据其并行度的不同，分别创建`ExecutionEdge`。共分`3`种情况：

(1)  如果并发数等于`partition`数，则一对一进行连接。如下图所示：
 即`numSources == parallelism`

![img](/images/10507536-9274be995a382392.png)

(2)  如果并发数大于`partition`数，则一对多进行连接。如下图所示：
即`numSources < parallelism`，且`parallelism % numSources == 0`

![img](/images/10507536-76cf412fd1531087.png)



 即`numSources < parallelism`，且`parallelism % numSources != 0`

![img](/images/10507536-75a151d59a3981bf.png)

(3) 如果并发数小于`partition`数，则多对一进行连接。如下图所示：
即`numSources > parallelism`，且`numSources % parallelism == 0`

![img](/images/10507536-646304409b42b06d.png)

即`numSources > parallelism`，且`numSources % parallelism != 0`

![img](/images/10507536-9955c154f6e1bff6.png)

### 总结

将`JobGraph`按照拓扑排序后得到一个`JobVertex`集合，遍历该`JobVertex`集合，即从`source`开始，将`JobVertex`封装成`ExecutionJobVertex`，并依次创建`ExecutionVertex`、`Execution`、`IntermediateResult`和`IntermediateResultPartition`。然后通过`ejv.connectToPredecessor()`方法，创建`ExecutionEdge`，建立当前节点与其上游节点之间的联系，即连接`ExecutionVertex`和`IntermediateResultPartition`。

# Flink调度

## 调度模式

Apache Flink内部提供了两种调度模式，分别为：**LAZY_FROM_SOURCES**(批处理）,**EAGER**（流计算）。

```java
public enum ScheduleMode {

	/** 批的lazy模式，从Source开始，下游的Task的输入数据准备好之后，调度下游的Task开始执行*/
	LAZY_FROM_SOURCES,

	/** 流的全量模式，一次性申请所有的资源 */
	EAGER;
}
```

### LAZY_FROM_SOURCES模式

**ExecutionJobVertex**代表某个**operation**，如**map**。在这里根据**ExecutionGraph**中**ExecutionJobVertex**的顺序来依次初始化。当使用到当前**task**时，再去调度。

**LAZY_FROM_SOURCES**理解为，一个一个**operator**逐个完成，比较适合批处理模式，这种模式使得每一种**operator**都能最大限度的利用集群资源。

```java
//ExecutionGraph.java
private CompletableFuture<Void> scheduleLazy(SlotProvider slotProvider) {

		final ArrayList<CompletableFuture<Void>> schedulingFutures = new ArrayList<>(numVerticesTotal);
		// simply take the vertices without inputs.
		for (ExecutionJobVertex ejv : verticesInCreationOrder) {
			if (ejv.getJobVertex().isInputVertex()) {
				final CompletableFuture<Void> schedulingJobVertexFuture = ejv.scheduleAll(
					slotProvider,
					allowQueuedScheduling,
					LocationPreferenceConstraint.ALL, // since it is an input vertex, the input based location preferences should be empty
					Collections.emptySet());

				schedulingFutures.add(schedulingJobVertexFuture);
			}
		}

		return FutureUtils.waitForAll(schedulingFutures);
	}
```



### EAGER模式

如果采用**EAGER**方式的话，是先调用**allocateResourcesForAll**来分配所需的所有资源，然后才是把所有的**task**部署到对应的TaskManager上。

```java
//ExecutionGraph.java

private CompletableFuture<Void> scheduleEager(SlotProvider slotProvider, final Time timeout) {
	assertRunningInJobMasterMainThread();
	checkState(state == JobStatus.RUNNING, "job is not running currently");

	// Important: reserve all the space we need up front.
	// that way we do not have any operation that can fail between allocating the slots
	// and adding them to the list. If we had a failure in between there, that would
	// cause the slots to get lost
	final boolean queued = allowQueuedScheduling;

	// collecting all the slots may resize and fail in that operation without slots getting lost
	final ArrayList<CompletableFuture<Execution>> allAllocationFutures = new ArrayList<>(getNumberOfExecutionJobVertices());

	final Set<AllocationID> allPreviousAllocationIds =
		Collections.unmodifiableSet(computeAllPriorAllocationIdsIfRequiredByScheduling());

	// allocate the slots (obtain all their futures
	for (ExecutionJobVertex ejv : getVerticesTopologically()) {
		// these calls are not blocking, they only return futures
		Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
			slotProvider,
			queued,
			LocationPreferenceConstraint.ALL,
			allPreviousAllocationIds,
			timeout);

		allAllocationFutures.addAll(allocationFutures);
	}

	// this future is complete once all slot futures are complete.
	// the future fails once one slot future fails.
	final ConjunctFuture<Collection<Execution>> allAllocationsFuture = FutureUtils.combineAll(allAllocationFutures);

	return allAllocationsFuture.thenAccept(
		(Collection<Execution> executionsToDeploy) -> {
			for (Execution execution : executionsToDeploy) {
				try {
					execution.deploy();
				} catch (Throwable t) {
					throw new CompletionException(
						new FlinkException(
							String.format("Could not deploy execution %s.", execution),
							t));
				}
			}
		})
		// Generate a more specific failure message for the eager scheduling
		.exceptionally(
			(Throwable throwable) -> {
				final Throwable strippedThrowable = ExceptionUtils.stripCompletionException(throwable);
				final Throwable resultThrowable;
				if (strippedThrowable instanceof TimeoutException) {
					int numTotal = allAllocationsFuture.getNumFuturesTotal();
					int numComplete = allAllocationsFuture.getNumFuturesCompleted();

					String message = "Could not allocate all requires slots within timeout of " +
						timeout + ". Slots required: " + numTotal + ", slots allocated: " + numComplete +
							", previous allocation IDs: " + allPreviousAllocationIds;

					StringBuilder executionMessageBuilder = new StringBuilder();

					for (int i = 0; i < allAllocationFutures.size(); i++) {
						CompletableFuture<Execution> executionFuture = allAllocationFutures.get(i);

						try {
							Execution execution = executionFuture.getNow(null);
							if (execution != null) {
								executionMessageBuilder.append("completed: " + execution);
							} else {
								executionMessageBuilder.append("incomplete: " + executionFuture);
							}
						} catch (CompletionException completionException) {
							executionMessageBuilder.append("completed exceptionally: " + completionException + "/" + executionFuture);
						}

						if (i < allAllocationFutures.size() - 1) {
							executionMessageBuilder.append(", ");
						}
					}

					message += ", execution status: " + executionMessageBuilder.toString();

					resultThrowable = new NoResourceAvailableException(message);
				} else {
					resultThrowable = strippedThrowable;
				}

				throw new CompletionException(resultThrowable);
			});
}
```



# Flink执行Task

## 状态迁移

### Job的状态迁移



![1560398153227](/images/1560398153227.png)

### Task的状态迁移

![1560398176914](/images/1560398176914.png)

## 生命周期

### Task生命周期

```java
//状态初始化
TASK::setInitialState
//开始执行
TASK::invoke
	    //初始化
	    create basic utils (config, etc) and load the chain of operators
	    setup-operators
	    task-specific-init
	    initialize-operator-states
   	    open-operators
	    //死循环处理数据
	    run
	    //销毁阶段
	    close-operators
	    dispose-operators
	    task-specific-cleanup
	    common-cleanup

```

### Operator生命周期

```java
   //初始化阶段
    OPERATOR::setup
        UDF::setRuntimeContext
    OPERATOR::initializeState
    OPERATOR::open
        UDF::open
    // 处理阶段(called on every element/watermark)
    OPERATOR::processElement
        UDF::run
    OPERATOR::processWatermark
    // 检查点阶段(called asynchronously on every checkpoint)
    OPERATOR::snapshotState      
    // 销毁阶段
    OPERATOR::close
        UDF::close
    OPERATOR::dispose
```

## Job的启动

### JobManager启动Task



### TaskManager启动Task

#### StreamTask的执行逻辑

在真正的运行时，是由

```java
StreamTask（OneInputStreamTask、TwoInputStreamTask、StreamIterationHead、StreamIterationTail）

-> StreamOneInputProcessor或StreamTwoInputProcessor

-> StreamInputProcessor.processInput()从InputGate读取数据

-> OneInputStreamOperator.processElement（）、processWatermark（）处理数据
```

> TowInputStreamOperator同理，额外需要判断调用processElement1还是processElement2



## 关键对象

### TaskManager

TaskManager是flink中资源管理的基本组件，是所有执行任务的基本容器，提供了内存管理、IO管理、通信管理等一系列功能。

- **MemoryManager**

​       JVM普遍存在着存储对象密度低、大内存时GC对系统影响大等问题。所以flink自己抽象了一套内存管理机制，将所有对象序列化后放在自己的MemorySegment上进行管理。

- **IOManager**

​       flink通过IOManager管理磁盘IO的过程，提供了同步和异步两种写模式，又进一步区分了block、buffer和bulk三种读写方式。

- **NettyShuffleManager**（以前叫**NetworkEnvironment**）

​        TaskManager的网络 IO 组件，包含了追踪中间结果和数据交换的数据结构。它的构造器会统一将配置的内存先分配出来，抽象成 NetworkBufferPool 统一管理内存的申请和释放。意思是说，在输入和输出数据时，不管是保留在本地内存，等待chain在一起的下个操作符进行处理，还是通过网络把本操作符的计算结果发送出去，都被抽象成了NetworkBufferPool。

### StreamTask

![1560406578603](/images/1560406578603.png)

# Flink的数据抽象和数据交换过程

## Flink内存

### 大数据平台面临的内存管理的问题

基于 JVM 的数据分析引擎都需要面对将大量数据存到内存中，这就不得不面对 JVM 存在的几个问题：

1. Java 对象存储密度低。一个只包含 boolean 属性的对象占用了16个字节内存：对象头占了8个，boolean 属性占了1个，对齐填充占了7个。而实际上只需要一个bit（1/8字节）就够了。
2. Full GC 会极大地影响性能，尤其是为了处理更大数据而开了很大内存空间的JVM来说，GC 会达到秒级甚至分钟级。
3. OOM 问题影响稳定性。OutOfMemoryError是分布式计算框架经常会遇到的问题，当JVM中所有对象大小超过分配给JVM的内存大小时，就会发生OutOfMemoryError错误，导致JVM崩溃，分布式框架的健壮性和性能都会受到影响。

所以目前，越来越多的大数据项目开始自己管理JVM内存了，像 Spark、Flink、HBase，为的就是获得像 C 一样的性能以及避免 OOM 的发生。本文将会讨论 Flink 是如何解决上面的问题的，主要内容包括内存管理、定制的序列化工具、缓存友好的数据结构和算法、堆外内存、JIT编译优化等。

Flink 并不是将大量对象存在堆上，而是将对象都序列化到一个预分配的内存块上，这个内存块叫做 `MemorySegment`，它代表了一段固定长度的内存（默认大小为 32KB），也是 Flink 中最小的内存分配单元，并且提供了非常高效的读写方法。你可以把 MemorySegment 想象成是为 Flink 定制的 `java.nio.ByteBuffer`。它的底层可以是一个普通的 Java 字节数组（`byte[]`），也可以是一个申请在堆外的 `ByteBuffer`。每条记录都会以序列化的形式存储在一个或多个`MemorySegment`中。

### Flink内存布局

Flink 中的 Worker 名叫 TaskManager，是用来运行用户代码的 JVM 进程。TaskManager 的堆内存主要被分成了三个部分：



![1560406627219](/images/1560406627219.png)

- **Network Buffers:** 一定数量的32KB大小的 buffer，主要用于数据的网络传输。在 TaskManager 启动的时候就会分配。默认数量是 2048 个，可以通过 `taskmanager.network.numberOfBuffers` 来配置。
- **Memory Manager Pool:** 这是一个由 `MemoryManager` 管理的，由众多`MemorySegment`组成的超大集合。Flink 中的算法（如 sort/shuffle/join）会向这个内存池申请 MemorySegment，将序列化后的数据存于其中，使用完后释放回内存池。默认情况下，池子占了堆内存的 70% 的大小。
- **Remaining (Free) Heap:** 这部分的内存是留给用户代码以及 TaskManager 的数据结构使用的。因为这些数据结构一般都很小，所以基本上这些内存都是给用户代码使用的。从GC的角度来看，可以把这里看成的新生代，也就是说这里主要都是由用户代码生成的短期对象。

**注意**

> Memory Manager Pool 主要在Batch模式下使用。在Steaming模式下，该池子不会预分配内存，也不会向该池子请求内存块。也就是说该部分的内存都是可以给用户代码使用的。不过社区是打算在 Streaming 模式下也能将该池子利用起来。

### 堆内内存管理

Flink 采用类似 DBMS 的 sort 和 join 算法，直接操作二进制数据，从而使序列化/反序列化带来的开销达到最小。所以 Flink 的内部实现更像 C/C++ 而非 Java。如果需要处理的数据超出了内存限制，则会将部分数据存储到硬盘上。如果要操作多块MemorySegment就像操作一块大的连续内存一样，Flink会使用逻辑视图（`AbstractPagedInputView`）来方便操作。下图描述了 Flink 如何存储序列化后的数据到内存块中，以及在需要的时候如何将数据存储到磁盘上。

从上面我们能够得出 Flink 积极的内存管理以及直接操作二进制数据有以下几点好处：

1. **减少GC压力。**显而易见，因为所有常驻型数据都以二进制的形式存在 Flink 的`MemoryManager`中，这些`MemorySegment`一直呆在老年代而不会被GC回收。其他的数据对象基本上是由用户代码生成的短生命周期对象，这部分对象可以被 Minor GC 快速回收。只要用户不去创建大量类似缓存的常驻型对象，那么老年代的大小是不会变的，Major GC也就永远不会发生。从而有效地降低了垃圾回收的压力。另外，这里的内存块还可以是堆外内存，这可以使得 JVM 内存更小，从而加速垃圾回收。
2. **避免了OOM。**所有的运行时数据结构和算法只能通过内存池申请内存，保证了其使用的内存大小是固定的，不会因为运行时数据结构和算法而发生OOM。在内存吃紧的情况下，算法（sort/join等）会高效地将一大批内存块写到磁盘，之后再读回来。因此，`OutOfMemoryErrors`可以有效地被避免。
3. **节省内存空间。**Java 对象在存储上有很多额外的消耗（如上一节所谈）。如果只存储实际数据的二进制内容，就可以避免这部分消耗。
4. **高效的二进制操作 & 缓存友好的计算。**二进制数据以定义好的格式存储，可以高效地比较与操作。另外，该二进制形式可以把相关的值，以及hash值，键值和指针等相邻地放进内存中。这使得数据结构可以对高速缓存更友好，可以从 L1/L2/L3 缓存获得性能的提升）。

### 走向堆外内存

Flink 基于堆内存的内存管理机制已经可以解决很多JVM现存问题了，为什么还要引入堆外内存？

1. 启动超大内存（上百GB）的JVM需要很长时间，GC停留时间也会很长（分钟级）。使用堆外内存的话，可以极大地减小堆内存（只需要分配Remaining Heap那一块），使得 TaskManager 扩展到上百GB内存不是问题。
2. 高效的 IO 操作。堆外内存在写磁盘或网络传输时是 zero-copy，而堆内存的话，至少需要 copy 一次。
3. 堆外内存是进程间共享的。也就是说，即使JVM进程崩溃也不会丢失数据。这可以用来做故障恢复（Flink暂时没有利用起这个，不过未来很可能会去做）。

但是强大的东西总是会有其负面的一面，不然为何大家不都用堆外内存呢。

1. 堆内存的使用、监控、调试都要简单很多。堆外内存意味着更复杂更麻烦。
2. Flink 有时需要分配短生命周期的 `MemorySegment`，这个申请在堆上会更廉价。
3. 有些操作在堆内存上会快一点点。

Flink用通过`ByteBuffer.allocateDirect(numBytes)`来申请堆外内存，用 `sun.misc.Unsafe` 来操作堆外内存。

基于 Flink 优秀的设计，实现堆外内存是很方便的。Flink 将原来的 `MemorySegment` 变成了抽象类，并生成了两个子类。`HeapMemorySegment` 和 `HybridMemorySegment`。从字面意思上也很容易理解，前者是用来分配堆内存的，后者是用来分配堆外内存**和堆内存**的。是的，你没有看错，后者既可以分配堆外内存又可以分配堆内存。为什么要这样设计呢？

首先假设`HybridMemorySegment`只提供分配堆外内存。在上述堆外内存的不足中的第二点谈到，Flink 有时需要分配短生命周期的 buffer，这些buffer用`HeapMemorySegment`会更高效。那么当使用堆外内存时，为了也满足堆内存的需求，我们需要同时加载两个子类。这就涉及到了 JIT 编译优化的问题。因为以前 `MemorySegment` 是一个单独的 final 类，没有子类。JIT 编译时，所有要调用的方法都是确定的，所有的方法调用都可以被去虚化（de-virtualized）和内联（inlined），这可以极大地提高性能（MemroySegment的使用相当频繁）。然而如果同时加载两个子类，那么 JIT 编译器就只能在真正运行到的时候才知道是哪个子类，这样就无法提前做优化。实际测试的性能差距在 2.7 被左右。

**Flink 使用了两种方案**

1. **方案1：只能有一种 MemorySegment 实现被加载**

   代码中所有的短生命周期和长生命周期的MemorySegment都实例化其中一个子类，另一个子类根本没有实例化过（使用工厂模式来控制）。那么运行一段时间后，JIT 会意识到所有调用的方法都是确定的，然后会做优化。

2. **方案2：提供一种实现能同时处理堆内存和堆外内存**

   这就是 `HybridMemorySegment` 了，能同时处理堆与堆外内存，这样就不需要子类了。这里 Flink 优雅地实现了一份代码能同时操作堆和堆外内存。这主要归功于 `sun.misc.Unsafe`提供的一系列方法，如getLong方法：

```java
sun.misc.Unsafe.getLong(Object reference, long offset)
```

- 如果reference不为空，则会取该对象的地址，加上后面的offset，从相对地址处取出8字节并得到 long。这对应了堆内存的场景。
- 如果reference为空，则offset就是要操作的绝对地址，从该地址处取出数据。这对应了堆外内存的场景。

**为什么只使用HybridMemorySegment**

​	Flink为MemorySegment提供了两个实现类：

- **HeapMemorySegment**

- **HybridMemorySegment**

  区别在于前者只能分配堆内存，而后者能用来分配堆内和堆外内存。事实上，Flink框架里，只使用了后者。

​		如果HybridMemorySegment只能用于分配堆外内存的话，似乎更合常理。但是在JVM的世界中，如果一个方法是一个虚方法，那么每次调用时，JVM都要花时间去确定调用的到底是哪个子类实现的该虚方法（方法重写机制，不明白的去看JVM的invokeVirtual指令），也就意味着每次都要去翻方法表；而如果该方法虽然是个虚方法，但实际上整个JVM里只有一个实现（就是说只加载了一个子类进来），那么JVM在运行一段时间后会感知到，进行去虚化处理，省略掉查询方法表的步骤，能够大大提升性能。但是只分配堆内或者堆外内存不能满足我们的需要，所以就出现了HybridMemorySegment同时可以分配两种内存的设计。

​		byte[]数组和directmemory是最底层的存储，那么memorysegment就是在其上覆盖的一层统一抽象。它定义了一系列抽象方法，用于控制和底层内存的交互。

## 数据抽象

### MemorySegment

​		MemorySegment**就是Flink的内存抽象。默认情况下，一个MemorySegment可以被看做是一个32KB大的内存块的抽象。这块内存既可以是堆内内存(Java的byte[] 数组)，也可以是堆外内存（基于Netty的DirectByteBuffer）。

**MemorySegment关键属性**

```java
UNSAFE : 用来对堆/非堆内存进行操作，是JVM的非安全的API
BYTE_ARRAY_BASE_OFFSET : 二进制字节数组的起始索引，相对于字节数组对象
LITTLE_ENDIAN ： 布尔值，是否为小端对齐（涉及到字节序的问题）
heapMemory : 如果为堆内存，则指向访问的内存的引用，否则若内存为非堆内存，则为null
address : 字节数组对应的相对地址（若heapMemory为null，即可能为off-heap内存的绝对地址，后续会详解）
addressLimit : 标识地址结束位置（address+size）
size : 内存段的字节数
```



> **Big endian与Little endian**
>
> 字节序的问题来自于PowerPC系列CPU和Intel的x86系列CPU。PowerPC系列采用big endian方式存储数据，而x86系列则采用little endian方式存储数据。Big endian是指低地址存放最高有效字节（MSB），而Little endian则是低地址存放最低有效字节（LSB）。
>
> 如下例所示：
>
> Big Endian
>
>    低地址                                            高地址
>
>  ----------------------------------------------------------------------------->
>
>    |     12     |      34    |     56      |     78    |
>
> Little Endian 
>
>    低地址                                            高地址
>
>    ----------------------------------------------------------------------------->
>
>    |     78     |      56    |     34      |     12    |



### ByteBuffer

​        Flink在数据从operator内的数据对象在向TaskManager上转移，预备被发给下个节点的过程中，使用的抽象或者说内存对象是Buffer 。

​       Flink的这个Buffer接口主要是一种flink层面用于传输数据和事件的统一抽象，其实现类是NetworkBuffer，是对MemorySegment的包装。Flink在各个TaskManager之间传递数据时，使用的是这一层的抽象。

​       Buffer的底层是MemorySegment，这可能不是JVM所管理的，所以为了知道什么时候一个Buffer用完了可以回收，Flink引入了引用计数的概念，当确认这个buffer没有人引用，就可以回收这一片MemorySegment。

### NetworkBufferPool

​       在每个**TaskManager**上只有一个，负责所有子task的内存管理。其实例化时就会尝试获取所有可由它管理的内存（对于堆内存来说，直接获取所有内存并放入老年代，并令用户对象只在新生代存活，可以极大程度的减少Full GC）。

​       **NetworkBufferPool**只是个工厂，实际的内存池是**LocalBufferPool** 。每个task有一个和其他task隔离的**LocalBufferPool**池。

NetworkBufferPool会计算自己所拥有的所有内存分片数，在分配新的内存池时对每个内存池应该占有的内存分片数重分配，步骤是：

1.  首先，从整个工厂管理的内存片中拿出所有的内存池所需要的最少Buffer数目总和
2.  如果正好分配完，就结束
3.  其次，把所有的剩下的没分配的内存片，按照每个LocalBufferPool内存池的剩余想要容量大小进行按比例分配
4.  剩余想要容量大小是这么个东西：如果该内存池至少需要3个buffer，最大需要10个buffer，那么它的剩余想要容量就是7

**LocalBufferPool**



### Record&RecordWriter

最底层内存抽象是**MemorySegment**，用于数据传输的是**Buffer**，那么，承上启下对接从Java对象转为Buffer的中间对象是什么呢？是**StreamRecord** 。

**StreamRecord<T>**是个Warp类，里面保存了原始的Java对象。另外，还保存了一个**timestamp**。 

**RecordWriter**类负责将**StreamRecord**进行序列化，调用**SpaningRecordSerializer**，再调用**BufferBuilder**写入**MemorySegment**中（每个Task都有自己的LocalBufferPool，LocalBufferPool中包含了多个MemorySegment）。

Flink Types中的每一种类型都实现了序列化的write方法。

RecordWriter将StreamRecord序列化完成之后，会根据flushAlways参数决定是否立即将数据进行推送，相当于1条记录发送一次，这样做延迟最低，但是吞吐量会下降，Flink默认的做法是单独启动一个线程，每隔一个固定时间flush一次所有的Channel，本质上是一种mini-batch（与spark的mini-batch不同）。

### InputGate



## 数据交换

### 数据传递

#### 数据传递的方式

- 本地线程之间的数据传递
- 跨网络的数据交换
- 同一线程的Operator之间的交换（Operator chain）

在flink中，数据处理的业务逻辑位于UDF ProcessFunction.processElement中，每一个ProcessFunction绑定与1个Operator中，Operator调用UDF处理数据完毕之后，需要将数据交给下一个Operator。那么Flink是如何衔接Operator之间的数据传递的，答案就是Collector接口。

Collector的类继承关系如下

![1560408117101](/images/1560408117101.png)

#### 本地线程间的数据传递

Operator FlatMap 所在线程 与 下游 Operator sum() 所在线程间的通信为例。这两个task线程共享同一个Buffer pool,通过wait()/notifyAll来同步。 Buffer和Netty中的ByteBuf功能类似，可以看作是一块共享的内存。inputGate负责读取Buffer或Event。

**交互过程**

1.  当没有Buffer可以消费时，Operator sum()所在的线程阻塞(通过inputGate中的inputChannelWithData.wait()方法阻塞) 
2.  当FlatMap所在线程写入结果数据到ResultSubPartition,并flush到buffer后 
3.  会唤醒Operator sum()所在的线程（通过inputChannelWithData.notifyAll()方法唤醒）。 
4.  线程被唤醒后会从Buffer中读取数据，经反序列化后，传递给Operator中的用户代码逻辑处理。

如下图所示

![1560408192767](/images/1560408192767.png)

#### 跨网络的数据传递

远程线程的Operator数据传递与本地线程类似。不同点在于，当没有Buffer可以消费时，会通过PartitionRequestClient向Operator FlatMap所在的进程发起RPC请求。远程的PartitionRequestServerHandler接收到请求后，读取ResultPartition管理的Buffer。并返回给Client。

如下图所示

![1560408373518](/images/1560408373518.png)

**RPC通讯过程**

远程数据转递RPC通信基于Netty实现， 图为Client端的RPC请求发送过程。PartitionRequestClient发出请求，交由Netty写到对应的socket。Netty读取Socket数据，解析Response后交由NetworkClientHandler处理。

![1560408446707](/images/1560408446707.png)



#### 同一线程的Operator之间的数据传递

以Operator sum() 和Operator sink:Print to std 为例。 这两个Operator在同一个线程中运行，数据不需要经过序列化和写到多线程共享的buffer中， Operator sum()通过Collector发送数据后，直接调用Operator sink的processElement方法传递数据。

如下图所示

![1560408523823](/images/1560408523823.png)

### 数据的写入和读取

#### Buffer的读取和写入

##### Buffer读取

1.Task线程启动后，通过while循环调用StreamInputProcessor.processInput()消费和发送数据,上文所指的“线程阻塞在inputGate中的inputChannelWithData.wait()”这段逻辑也是在StreamInputProcessor.processInput中发生的。Task启动到Buffer读取的调用栈如下图所示,图例只展示了核心内容，省略了一些逻辑。

2.在调用栈的后部，ResultSubpartition的子类PipelinedSubPartition通过BufferConsumer来读取Buffer。

![1560408708930](/images/1560408708930.png)

##### Buffer写入

StreamInputProcessor调用CheckPointBarrierHandler.getNextNonBlocked阻塞直到获取到StreamRecord后，会调用StreamFlatMap.processElement(),内部调用FlatMapFunction.flatMap(T value, Collector<O> out) 。该方法包含两个参数， value为反序列化后的StreamRecord ， Collecotr负责把数据发送到下游。 用户的代码逻辑获取到value后进行处理，处理结果通过Collector发送。

部分Collector的实现类如下所示

![1560408776958](/images/1560408776958.png)

**Buffer写入涉及的类**



![1560408825476](/images/1560408825476.png)

FlatMap和sum()是线程间通信，数据的发送最后会委托给实现了Collector接口的RecordWriterOutput。在RecordWriterOutput内部, 序列化逻辑由TypeSerializer实现,数据发送逻辑由StreamRecordWriter实现。而StreamRecordWriter的包含了ChannelSelector , ResultPartitionWriter这两个对象，其中ChannelSelector相当于NIO概念中的Selector,负责选择Channel,即选择通向数据发送的目标对象的Buffer通道。ResultPartitionWriter提供了两类方法，flush数据到buffer和添加buffer的消费者BufferConsumer, BufferConsumer用于读取ResultPartitionWriter写入的buffer,提供可消费的Buffer视图。ResultPartitionWriter的主要实现类是ResultPartition。

> <font color='red'>注意：此处需要重新编写，因为现有版本已经不同了</font>
>
> 1.StreamRecordWriter与RecordWriter在当前版本中已经何为一体。
>
> 2.ResultPartition是一个复杂的类,它代表单个task生产的结果数据块 , 一个ResultPartition是一系列Buffer实例的集合。这些buffer会被分配到一个或多个ResultSubpartition实例中。即，一个ResultPartition可能被分成多个子集，然后根据数据分发类型交由consuming task消费。ResultSubpartition的主要实现类是PipelinedSubpartition , 该类提供了通知功能，当有新的数据写入buffer时，会回调BufferAvailabilityListener的notifyDataAvailable()方法。

**Buffer可用通知**

在此例中，上下游在同一个进程中，使用的是LocalInputChannel, LocalInputChannel实际调用SingleInputGate.notifyChannelNonEmpty(InputChannel channel) ，这个方法调用inputChannelsWithData.notifyAll , 唤醒阻塞在inputChannelsWithData对象实例的所有线程。上文提到的阻塞在CheckPointBarrierHandler.getNextNonBlocked()方法的线程也会被唤醒，返回数据。

![1560409206438](/images/1560409206438.png)

####  Netty的数据写入和读取

##### 数据写入Netty的过程

```java
-> ResultPartition.flushAll  

-> AbstractChannelHandlerContext.fireUserEventTriggered

-> PartittionRequesetQueue.enqueueAvailableReader

-> PartittionRequesetQueue.writeAndFlushNextMessageIfPossible
```

至此数据真实的写入netty channel，发送给下游，

-> AbstractChannelHandlerContext.fireUserEventTriggered

-> PartittionRequesetQueue.enqueueAvailableReader

-> PartittionRequesetQueue.writeAndFlushNextMessageIfPossible

##### 从netty读取数据





# Flink背压



# Flink部署

## 单机部署
## StandAlone模式
## Yarn模式
## K8s模式