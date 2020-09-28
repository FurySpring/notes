## stream

简介：Java8集合中的Stream相当于高级版的Iterator，它可以通过Lambda表达式对集合进行各种非常便利、高效的聚合操作（Aggregate Operation），或者大批量数据操作（Bulk Data Operation）。

#### 1.Stream操作分类

官方将Stream中的操作分为两大类：中间操作（Intermediate Operations）和终结操作（Terminal Operations）。中间操作只对操作记录，即只返回一个流，不会计算，而终结操作才实现了计算。

中间操作又可以分为无状态（Stateless）与有状态（Stateful）操作，前者是指元素在的处理不受之前元素的影响，后者是指该操作只有拿到所有元素之后才能继续下去。

终结操作又可以分为短路（Short-circuiting）与非短路（Unshort-circuiting）操作，前者是指遇到某些符合条件的元素就可以得到最终结果，后者是指必须处理完所有元素才能得到最终结果。操作分类详情如下图：

![img](https://static001.geekbang.org/resource/image/ea/94/ea8dfeebeae8f05ae809ee61b3bf3094.jpg)



我们通常还会将中间操作成为兰操作，也正是由这种懒操作结合终结操作、数据源构成的处理管道（Pipeline）实现了Stream的高效。



#### 2.Stream源码实现

Steam包是由哪些的主要结构类组合而成的，各个类的职责是什么。

![img](https://static001.geekbang.org/resource/image/fc/00/fc256f9f8f9e3224aac10b2ee8940e00.jpg)

Base Stream和Stream为最顶端的接口类。Base Stream主要定义了流的基本接口方法，例如，spliterator、isParallel等；Stream则定义了一些流的常用操作方式，例如，map、filter等。

Reference Pipeline是一个结构类，它通过定义内部类组装了各种操作流。它定义了Head、StatelessOp、StatefulOp三个内部类，实现了BaseStream与Stream的接口方法。

Sink接口是定义每个Stream操作之间的协议，它包含begin(), end(), cancellationRequested(), accpt()四个方法。ReferencePipeline最终会将整个Stream流操作组装成一个调用链，而这条调用链上的各个Stream操作的上下关系就是通过Sink接口协议来定义实现的。



#### 3.Steam操作叠加

一个Stream的各个操作是由处理管道组装，并统一完成数据处理的。在jdk中每次的终端操作会以使用阶段（Stage）命名。

管道通常是由ReferencePipeline类实现的，前面提到Steam包结构时，讲到ReferencePipeline包含了Head、StatelessOp、StatefulOp三种内部类。

Head类主要用来定义数据源操作，在我们初次调用names.stream()方法时，会初次加载Head对象，此时加载数据源操作；接着加载的是中间操作，分别为无状态中间操作StatelessOp对象和有状态操作StatefulOp对象，此时的Stage并没有执行，而是通过AbstractPipeline生成了一个中间操作Stage链表；当我们调用终结操作时，会生成一个最终的Stage，通过这个Stage触发之前的中间操作，从最后一个Stage开始，递归生成一个Sink链。如下图所示：

![img](https://static001.geekbang.org/resource/image/f5/19/f548ce93fef2d41b03274295aa0a0419.jpg)



下面通过一个例子来感受下Stream的操作分类是如何实现高效迭代大数据集合的。

```java
List<String> names = Arrays.asList(" 张三 ", " 李四 ", " 王老五 ", " 李三 ", " 刘老四 ", " 王小二 ", " 张四 ", " 张五六七 ");

String maxLenStartWithZ = names.stream()
    	            .filter(name -> name.startsWith(" 张 "))
    	            .mapToInt(String::length)
    	            .max()
    	            .toString();
```

分析：

首先，names是ArrayList集合，所以names.stream()方法将会调用集合类基础接口Collection的Stream方法：

```java
default Stream<E> stream() {
	return StreamSupport.stream(apliterator(), false);
}
```



然后，Stream方法就会调用StreamSupport类的Stream方法，方法中初始化了一个ReferencePipeline的Head内部类对象：

```java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
	Objects.requiresNonNull(spliterator);
	return new ReferencePipeline.Head<>(spliterator,
                        StreamOpFlag.fromCharacteristics(spliterator),
                        parallel);
}
```



再掉用filter和map方法，这两个方法都是无状态的中间操作，所以执行filter和map操作时，并没有在进行任何操作，而是分别创建了一个Stage来标识用户的每一次操作。

而通常情况下Stream的操作又需要一个回调函数，所以一个完整的stage是由数据来源/操作/回调函数组成的三元组来表示。如下所示，分别是ReferencePipeline的filter方法和map方法：

```java
	@Override
	public final Stream<P_OUT> filter(Predicate<? super P_OUT> predicate) {
        Objects.requireNonNull(predicate);
        return new StatelessOp<P_OUT, P_OUT>(this, StreamShape.REFERENCE,
                                            StreamOpFlag.NOT_SIZED) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) {
                return new Sink.ChainedReference<P_OUT, P_OUT>(sink) {
                    @Override
                    public void begin(long size) {
                        downstream.begin(-1);
                    }
                    
                    @Override
                    public void accept(P_OUT u) {
                        if (predicate.test(u))
                            downstream.accept(u);
                    }
                };
            }
        };
    }
```

```java
	@Override
	@SuppressWarnings("unchecked")
	public final <R> Stream<R> map(function<? super P_OUT, ? extends R> mapper) {
        Objects.requireNonNull(mapper);
        return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                        StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
                return new Sink.ChainedReference<P_OUT，R>(sink) {
                    @Override
                    public void accept(P_OUT u) {
                        downstream.accept(mapper.apply(u));
                    }
                };
            }
        };
    }
```



new StatelessOp将会掉用父类AbstractPipeline的构造函数，这个构造函数将前后的Stage联系起来，生成一个Stage链表：

```java
AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
    if (previousStage.linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    previousStage.linkedOrConsumed = true;
    // 将当前的stage的next指针指向之前的stage
    previousStage.nextStage = this;
    // 赋值当前stage当全局变量previousStage
    this.previousStage = previousStage;
    this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
    this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
    this.sourceStage = previousStage.sourceStage;
    if (opIsStateful())
        sourceStage.sourceAnyStateful = true;
    this.depth = previousStage.depth + 1;
}
```

因为在创建每一个Stage时，都会包含一个opWrapSink()方法，该方法会把一个操作的具体实现封装在Sink类中，Sink采用（处理 -> 转发）的模式来叠加操作。

当执行max方法时，会调用ReferencePipeline的max方法，此时由于max方法是终结操作，所以会创建一个TerminalOp操作，同时创建一个ReducingSink，并且将操作封装在Sink类中。

```java
	@Override
	public final Optional<P_OUT> max(Comparator<? super P_OUT> comparator) {
        return reduce(BinaryOperator.maxBy(comparator));
    }
```



最后，掉用AbstractPipeline的wrapSink方法，该方法会掉用opWrapSink生成一个Sink链表，Sink链表中的每一个Sink都封装了一个操作的具体实现。

```java
	@Override
	@SuppressWarnings("unchecked")
	final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
        Objects.requiresNonNull(sink);
        
        for (@SuppressWarnings("rawtypes") AbstractPipeline p = AbstractPipeline.this; p.depth > 0; p = p.previousStage) {
            sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
        }
        return (Sink<P_IN>) sink;
    }
```

当Sink链表生成完成后，Stream开始执行，通过spliterator迭代集合，执行Sink链表中的具体操作。

```java
	@Override
	final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        Objects.requireNonNull(spliterator);
        
        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
            wrappedSink.begin(spliterator.getExactSizeIfKnown());
            spliterator.forEachRemaining(wrappedSink);
            wrappedSink.end();
        } else {
            copyIntoWithCancel(wrappedSink, spliterator);
        }
    }
```

Java8 中的Spliterator的forEachRemaining会迭代集合，每迭代一次，都会执行一次filter操作，如果filter操作通过，就会触发map操作，然后将结果放入到临时数组object中，再进行下一次的迭代。完成中间操作后，就会触发终结操作max。



这就是串行处理方式，那么Stream的另一种处理数据的方式又是怎么操作的呢？



#### 4.Stream并行处理

Stream处理数据的方式有两种，穿行处理和并行处理。要实现并行处理，只需要在例子的代码中新增一个parallel()方法，代码如下：

```java
List<String> names = Arrays.asList(" 张三 ", " 李四 ", " 王老五 ", " 李三 ", " 刘老四 ", " 王小二 ", " 张四 ", " 张五六七 ");

String maxLenStartWithZ = names.stream()
                    .parallel()
    	            .filter(name -> name.startsWith(" 张 "))
    	            .mapToInt(String::length)
    	            .max()
    	            .toString();

```

Stream的并行处理在执行终结操作之前，跟串行处理的实现是一样的。而在掉用终结方法之后，实现方式就有点不一样，会调用TerminalOp的evaluateParallel方法并行处理。

```java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
    assert getOutputShape() == terminalOp.inputShape();
    if (linkedConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    linkedOrConsumed = true;
    
    return isParallel() 
        ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
        : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
}
```

这里的并行处理指的是，Stream结合了ForkJoin框架，对Stream处理进行了分片，Spliterator中的estimateSize方法会估算出分片的数据量。

通过预估的数据量获取最小处理单元的阈值，如果当前分片大小大于最小处理单元的阈值，就继续切分集合。每个分片都将会生成一个Sink链表，当所有分片操作完成后，ForkJoin框架将会合并分片任何结果集。



#### 合理使用Stream

在循环迭代次数较少的情况下，常规的迭代方式性能反而更好；在单核CPU服务器配置环境中，也是常规迭代方式更有优势；而在大数据循环迭代中，如果服务器是多核CPU的情况下，Stream的并行迭代优势明显。所以我们在平时处理大数据的集合时，应该尽量考虑将应用部署在多喝CPU环境下，并且使用Stream的并行迭代方式处理。



## 总结

纵观 Stream 的设计实现，非常值得我们学习。从大的设计方向上来说，Stream 将整个操作分解为了链式结构，不仅简化了遍历操作，还为实现并行计算打下了基础。

从小的分类方向上来说，Stream 将遍历元素的操作和对元素的计算分为中间操作和终结操作，而中间操作又根据元素之间状态有无干扰分为有状态和无状态操作，实现了链结构中的不同阶段。

**在串行处理操作中，**Stream 在执行每一步中间操作时，并不会做实际的数据操作处理，而是将这些中间操作串联起来，最终由终结操作触发，生成一个数据处理链表，通过 Java8 中的 Spliterator 迭代器进行数据处理；此时，每执行一次迭代，就对所有的无状态的中间操作进行数据处理，而对有状态的中间操作，就需要迭代处理完所有的数据，再进行处理操作；最后就是进行终结操作的数据处理。

**在并行处理操作中，**Stream 对中间操作基本跟串行处理方式是一样的，但在终结操作中，Stream 将结合 ForkJoin 框架对集合进行切片处理，ForkJoin 框架将每个切片的处理结果 Join 合并起来。最后就是要注意 Stream 的使用场景。



## 思考题

这里有一个简单的并行处理案例，请你找出其中存在的问题。

```java
// 使用一个容器装载 100 个数字，通过 Stream 并行处理的方式将容器中为单数的数字转移到容器 parallelList
List<Integer> integerList= new ArrayList<Integer>();
 
for (int i = 0; i <100; i++) {
      integerList.add(i);
}
 
List<Integer> parallelList = new ArrayList<Integer>() ;
integerList.stream()
           .parallel()
           .filter(i->i%2==1)
           .forEach(i->parallelList.add(i));
 
```

