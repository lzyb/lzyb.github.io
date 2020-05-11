---
title: Java8 Stream流
date: 2017-05-11 10:01:04
categories: 
- Java
tags: 
- Stream流
- Java8
---
## Java 8 Stream流
### Java 8 Stream流概述
虽然JAVA8中的stream API与JAVA I/O中的InputStream和OutputStream在名字上比较类似，但是其实是另外一个东西，Stream API是JAVA函数式编程中的一个重要组成部分。
<!--more-->
### Streams如何工作？
stream是一个可以对单列集合中的元素执行各种计算操作的一个元素序列。
```Java
public static void main(String[] args) {

List<String> myList = Arrays.asList("a1", "a2", "b1", "c2", "c1");
myList.stream()
      .filter(s -> s.startsWith("c"))
      .map(String::toUpperCase)
      .sorted()
      .forEach(System.out::println);
      // C1
      // C2
}

```
stream包含中间（intermediate operations）和最终（terminal operation）两种形式的操作。中间操作（intermediate operations）的返回值还是一个stream，因此可以通过链式调用将中间操作（intermediate operations）串联起来。最终操作（terminal operation）只能返回void或者一个非stream的结果。在上述例子中：`filter`, `map` ，`sorted`是中间操作，而`forEach`是一个最终操作。更多关于stream的中可用的操作可以查看java doc。上面例子中的链式调用也被称为操作管道流。

大多数流操作都接受某种lambda表达式参数，这是一个指定操作确切行为的功能接口。这些操作大多数都必须是无干扰的和无状态的。这意味着什么？

当函数不修改流的基础数据源时，它是无干扰的，例如，在上面的示例中，没有lambda表达式myList通过添加或删除集合中的元素来进行修改。

当操作的执行是确定性的时，函数是无状态的，例如，在上面的示例中，lambda表达式不依赖于外部变量的任何可变变量或状态，这些变量或状态可能在执行期间发生变化。

### 不同类型的流
可以从各种数据源（尤其是集合）创建流。列表和集合支持新方法，`stream()`并`parallelStream()`可以创建顺序流或并行流。并行流能够在多个线程上运行，并且将在本教程的后续部分中介绍。现在，我们关注顺序流：
```Java
Arrays.asList("a1", "a2", "a3")
    .stream()
    .findFirst()
    .ifPresent(System.out::println);  // a1
```
`stream()`在对象列表上调用该方法将返回常规对象流。但是我们不必创建集合即可使用流，如我们在下一个代码示例中看到的那样：
```Java
Stream.of("a1", "a2", "a3")
    .findFirst()
    .ifPresent(System.out::println);  // a1
```
仅用于`Stream.of()`从一堆对象引用创建流。

除了常规对象流之外，Java 8还附带了特殊的流，用于处理原始数据类型`int`，`long`以及`double`。您可能已经猜到了`IntStream`，`LongStream`和`DoubleStream`。

IntStreams可以使用以下方法替换常规的for循环`IntStream.range()`：
```Java
IntStream.range(1, 4)
    .forEach(System.out::println);

// 1
// 2
// 3
```
所有这些原始流都像常规对象流一样工作，但有以下区别：原始流使用专用的lambda表达式，例如`IntFunction`代替`Function`或`IntPredicate`代替`Predicate`。基本流支持其他终端聚合操作`sum()`和`average()`：
```Java
Arrays.stream(new int[] {1, 2, 3})
    .map(n -> 2 * n + 1)
    .average()
    .ifPresent(System.out::println);  // 5.0
```
有时将常规对象流转换为原始流很有用，反之亦然。为此，对象流支持特殊的映射操作`mapToInt()`，`mapToLong()`并且`mapToDouble`：
```Java
Stream.of("a1", "a2", "a3")
    .map(s -> s.substring(1))
    .mapToInt(Integer::parseInt)
    .max()
    .ifPresent(System.out::println);  // 3
```
原始流可以通过以下方式转换为对象流mapToObj()：
```Java
IntStream.range(1, 4)
    .mapToObj(i -> "a" + i)
    .forEach(System.out::println);

// a1
// a2
// a3
```
这是一个组合的示例：双精度流首先映射到int流，然后映射到字符串对象流：
```Java
Stream.of(1.0, 2.0, 3.0)
    .mapToInt(Double::intValue)
    .mapToObj(i -> "a" + i)
    .forEach(System.out::println);

// a1
// a2
// a3
```
### 处理订单号
既然我们已经学习了如何创建和使用不同类型的流，那么让我们更深入地了解如何在后台处理流操作。

中间操作的一个重要特征是懒惰。查看以下缺少终端操作的示例：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return true;
    });
    ```
执行此代码段时，没有任何内容打印到控制台。这是因为仅当存在终端操作时才执行中间操作。

让我们通过终端操作扩展以上示例forEach：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return true;
    })
    .forEach(s -> System.out.println("forEach: " + s));
执行此代码段将在控制台上产生所需的输出：
```
```Java
filter:  d2
forEach: d2
filter:  a2
forEach: a2
filter:  b1
forEach: b1
filter:  b3
forEach: b3
filter:  c
forEach: c
```
结果的顺序可能令人惊讶。天真的方法是在流的所有元素上一个接一个地水平执行操作。但是，每个元素都沿链垂直移动。然后，第一个字符串“ d2”通过，`filter`然后`forEach`才处理第二个字符串“ a2”。

这种行为可以减少在每个元素上执行的实际操作数，如下例所示：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .anyMatch(s -> {
        System.out.println("anyMatch: " + s);
        return s.startsWith("A");
    });

// map:      d2
// anyMatch: D2
// map:      a2
// anyMatch: A2
```
谓词应用于给定输入元素后，该操作`anyMatch`将`true`立即返回。对于通过“ A2”的第二个元素，这是正确的。由于流链是垂直执行的，`map`因此在这种情况下只需执行两次。因此，`map`将尽可能少地调用而不是映射流的所有元素。

#### 为什么为了事项
下一个示例包括两个中间操作`map`和`filter`和终端操作`forEach`。让我们再次检查这些操作是如何执行的：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("A");
    })
    .forEach(s -> System.out.println("forEach: " + s));

// map:     d2
// filter:  D2
// map:     a2
// filter:  A2
// forEach: A2
// map:     b1
// filter:  B1
// map:     b3
// filter:  B3
// map:     c
// filter:  C
```
您可能已经猜到了两者，map并且filter基础集合中的每个字符串都被调用了五次，而forEach仅被调用了一次。

如果更改操作顺序（移至filter链的开头），则可以大大减少实际的执行次数：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));

// filter:  d2
// filter:  a2
// map:     a2
// forEach: A2
// filter:  b1
// filter:  b3
// filter:  c
```
现在，`map`仅调用一次，因此操作管道对于大量输入元素的执行速度要快得多。组成复杂的方法链时，请记住这一点。

让我们通过一个额外的操作扩展上述示例`sorted`：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .sorted((s1, s2) -> {
        System.out.printf("sort: %s; %s\n", s1, s2);
        return s1.compareTo(s2);
    })
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));
    ```
排序是一种特殊的中间操作。这是所谓的有状态操作，因为为了对元素集合进行排序，您必须在排序期间保持状态。

执行此示例将得到以下控制台输出：
```Java
sort:    a2; d2
sort:    b1; a2
sort:    b1; d2
sort:    b1; a2
sort:    b3; b1
sort:    b3; d2
sort:    c; b3
sort:    c; d2
filter:  a2
map:     a2
forEach: A2
filter:  b1
filter:  b3
filter:  c
filter:  d2
```
首先，对整个输入集合执行排序操作。换句话说，`sorted`是水平执行的。因此，在这种情况下`sorted`，对输入集合中每个元素的多个组合调用了八次。

我们可以通过重新排序链来再次优化性能：
```Java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .sorted((s1, s2) -> {
        System.out.printf("sort: %s; %s\n", s1, s2);
        return s1.compareTo(s2);
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));

// filter:  d2
// filter:  a2
// filter:  b1
// filter:  b3
// filter:  c
// map:     a2
// forEach: A2
```
在此示例`sorted`中，因为`filter`将输入集合简化为一个元素而从未被调用。因此，对于较大的输入集合，性能会大大提高。

### 重用流
Java 8流无法重用。调用任何终端操作后，流就立即关闭：
```Java
Stream<String> stream =
    Stream.of("d2", "a2", "b1", "b3", "c")
        .filter(s -> s.startsWith("a"));

stream.anyMatch(s -> true);    // ok
stream.noneMatch(s -> true);   // exception
```
在同一流上调用`noneMatchafte`r会`anyMatch`导致以下异常：
```Java
java.lang.IllegalStateException: stream has already been operated upon or closed
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:229)
    at java.util.stream.ReferencePipeline.noneMatch(ReferencePipeline.java:459)
    at com.winterbe.java8.Streams5.test7(Streams5.java:38)
    at com.winterbe.java8.Streams5.main(Streams5.java:28)
    ```
为了克服此限制，我们必须为要执行的每个终端操作创建一个新的流链，例如，我们可以创建一个流提供程序以构造一个已经设置了所有中间操作的新流：
```Java
Supplier<Stream<String>> streamSupplier =
    () -> Stream.of("d2", "a2", "b1", "b3", "c")
            .filter(s -> s.startsWith("a"));

streamSupplier.get().anyMatch(s -> true);   // ok
streamSupplier.get().noneMatch(s -> true);  // ok
```
每次调用都会`get()`构造一个新的流，我们可以保存该流以调用所需的终端操作。

### 高级操作
流支持许多不同的操作。我们已经了解了最重要的操作，例如`filter`或`map`。我留给您发现所有其他可用的操作（请参阅Stream Javadoc）。相反，让我们更深入地了解了更复杂的操作`collect`，`flatMap`和`reduce`。

本节中的大多数代码示例都使用以下人员进行演示：
```Java
class Person {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return name;
    }
}

List<Person> persons =
    Arrays.asList(
        new Person("Max", 18),
        new Person("Peter", 23),
        new Person("Pamela", 23),
        new Person("David", 12));
```
#### 收集
收集是到流中的元素转换为不同的种类的结果，例如一个非常有用的终端操作`List`，`Set`或`Map`。收集接受C`ollector`由四个不同的操作组成的：供应商，累加器，合并器和装订器。乍一看，这听起来超级复杂，但是好地方是Java 8通过`Collectors`该类支持各种内置的收集器。因此，对于最常见的操作，您不必自己实现收集器。

让我们从一个非常常见的用例开始：
```Java
List<Person> filtered =
    persons
        .stream()
        .filter(p -> p.name.startsWith("P"))
        .collect(Collectors.toList());

System.out.println(filtered);    // [Peter, Pamela]
```
如您所见，从流的元素构造列表非常简单。需要一个集合而不是列表-只需使用`Collectors.toSet()`。

下一个示例按年龄对所有人进行分组：
```Java
Map<Integer, List<Person>> personsByAge = persons
    .stream()
    .collect(Collectors.groupingBy(p -> p.age));

personsByAge
    .forEach((age, p) -> System.out.format("age %s: %s\n", age, p));

// age 18: [Max]
// age 23: [Peter, Pamela]
// age 12: [David]
```
收集器功能极为丰富。您还可以在信息流的元素上创建汇总，例如，确定所有人的平均年龄：
```Java
Double averageAge = persons
    .stream()
    .collect(Collectors.averagingInt(p -> p.age));

System.out.println(averageAge);     // 19.0
```
如果您对更全面的统计感兴趣，则汇总收集器将返回一个特殊的内置汇总统计对象。因此，我们可以简单地确定人员的最小，最大和算术平均年龄以及总数和计数。
```Java
IntSummaryStatistics ageSummary =
    persons
        .stream()
        .collect(Collectors.summarizingInt(p -> p.age));

System.out.println(ageSummary);
// IntSummaryStatistics{count=4, sum=76, min=12, average=19.000000, max=23}
```
下一个示例将所有人连接成一个字符串：
```Java
String phrase = persons
    .stream()
    .filter(p -> p.age >= 18)
    .map(p -> p.name)
    .collect(Collectors.joining(" and ", "In Germany ", " are of legal age."));

System.out.println(phrase);
// In Germany Max and Peter and Pamela are of legal age.
```
联接收集器接受定界符以及可选的前缀和后缀。

为了将流元素转换为映射，我们必须指定如何映射键和值。请记住，映射的键必须唯一，否则将`IllegalStateException`抛出。您可以选择将合并功能作为附加参数传递来绕过异常：
```Java
Map<Integer, String> map = persons
    .stream()
    .collect(Collectors.toMap(
        p -> p.age,
        p -> p.name,
        (name1, name2) -> name1 + ";" + name2));

System.out.println(map);
// {18=Max, 23=Peter;Pamela, 12=David}
```
现在我们知道一些最强大的内置收集器，让我们尝试构建自己的特殊收集器。我们希望将流中的所有人转换为单个字符串，该字符串包含所有用|竖线字符分隔的大写字母名称。为了实现这一点，我们通过创建了一个新的收集器Collector.of()。我们必须通过收集器的四个要素：供应商，累加器，组合器和修整器。
```Java
Collector<Person, StringJoiner, String> personNameCollector =
    Collector.of(
        () -> new StringJoiner(" | "),          // supplier
        (j, p) -> j.add(p.name.toUpperCase()),  // accumulator
        (j1, j2) -> j1.merge(j2),               // combiner
        StringJoiner::toString);                // finisher

String names = persons
    .stream()
    .collect(personNameCollector);

System.out.println(names);  // MAX | PETER | PAMELA | DAVID
```
由于Java中的字符串是不可变的，因此我们需要一个帮助器类，`StringJoiner`以便让收集器构造我们的字符串。供应商最初使用适当的定界符构造此类StringJoiner。累加器用于将每个人的大写名称添加到StringJoiner。组合器知道如何将两个StringJoiners合并为一个。在最后一步，修整器从StringJoiner构造所需的String。

#### FlatMap 
我们已经学习了如何通过使用map操作将流的对象转换为另一种对象。映射是有限的，因为每个对象只能精确地映射到另一个对象。但是，如果我们想将一个对象转换成多个其他对象，或者根本不转换呢？这就是`flatMap`救援的地方。

FlatMap将流的每个元素转换为其他对象的流。因此，每个对象都将转换为零，一个或多个由流支持的其他对象。然后，将这些流的内容放入`flatMap`操作的返回流中。

在看到`flatMap`实际效果之前，我们需要一个适当的类型层次结构：
```Java
class Foo {
    String name;
    List<Bar> bars = new ArrayList<>();

    Foo(String name) {
        this.name = name;
    }
}

class Bar {
    String name;

    Bar(String name) {
        this.name = name;
    }
}
```
接下来，我们利用关于流的知识来实例化几个对象：
```Java
List<Foo> foos = new ArrayList<>();

// create foos
IntStream
    .range(1, 4)
    .forEach(i -> foos.add(new Foo("Foo" + i)));

// create bars
foos.forEach(f ->
    IntStream
        .range(1, 4)
        .forEach(i -> f.bars.add(new Bar("Bar" + i + " <- " + f.name))));
```
现在我们有了三个foo的列表，每个foo包含三个小节。

FlatMap接受一个必须返回对象流的函数。因此，为了解析每个foo的bar对象，我们只需传递适当的函数：
```Java
foos.stream()
    .flatMap(f -> f.bars.stream())
    .forEach(b -> System.out.println(b.name));

// Bar1 <- Foo1
// Bar2 <- Foo1
// Bar3 <- Foo1
// Bar1 <- Foo2
// Bar2 <- Foo2
// Bar3 <- Foo2
// Bar1 <- Foo3
// Bar2 <- Foo3
// Bar3 <- Foo3
```
如您所见，我们已经成功地将三个foo对象的流转换为九个bar对象的流。

最后，以上代码示例可以简化为单个流操作管道：
```Java
IntStream.range(1, 4)
    .mapToObj(i -> new Foo("Foo" + i))
    .peek(f -> IntStream.range(1, 4)
        .mapToObj(i -> new Bar("Bar" + i + " <- " f.name))
        .forEach(f.bars::add))
    .flatMap(f -> f.bars.stream())
    .forEach(b -> System.out.println(b.name));
```
FlatMap也可用于`Optional` Java 8中引入的类。Optionals `flatMap`操作返回另一种类型的可选对象。因此，它可以用来防止令人讨厌的`null`检查。

考虑这样一个高度分层的结构：
```Java
class Outer {
    Nested nested;
}

class Nested {
    Inner inner;
}

class Inner {
    String foo;
}
```
为了解析`foo`外部实例的内部字符串，您必须添加多个null检查以防止可能的NullPointerExceptions：
```Java
Outer outer = new Outer();
if (outer != null && outer.nested != null && outer.nested.inner != null) {
    System.out.println(outer.nested.inner.foo);
}
```
通过使用可选`flatMap`操作可以获得相同的行为：
```Java
Optional.of(new Outer())
    .flatMap(o -> Optional.ofNullable(o.nested))
    .flatMap(n -> Optional.ofNullable(n.inner))
    .flatMap(i -> Optional.ofNullable(i.foo))
    .ifPresent(System.out::println);
```
每次调用时，如果存在或不存在，则`flatMap`返回一个`Optional`包装所需对象的包装null。

#### 减少
归约运算将流的所有元素组合为单个结果。Java 8支持三种不同的`reduce`方法。第一个将元素流简化为该流的一个元素。让我们看看如何使用此方法确定最大的人：
```Java
persons
    .stream()
    .reduce((p1, p2) -> p1.age > p2.age ? p1 : p2)
    .ifPresent(System.out::println);    // Pamela
```
该`reduce`方法接受`BinaryOperator`累加器功能。`BiFunction`在这种情况下，实际上这是两个操作数共享相同类型的地方`Person`。`BiFunction`就像，`Function`但是接受两个参数。示例函数比较两个人的年龄，以便返回最大年龄的人。

第二种`reduce`方法接受身份值和`BinaryOperator`累加器。可使用此方法来构造一个新人员，并使用流中所有其他人员的姓名和年龄进行汇总：
```Java
Person result =
    persons
        .stream()
        .reduce(new Person("", 0), (p1, p2) -> {
            p1.age += p2.age;
            p1.name += p2.name;
            return p1;
        });

System.out.format("name=%s; age=%s", result.name, result.age);
// name=MaxPeterPamelaDavid; age=76
```
第三种`reduce`方法接受三个参数：标识值，`BiFunction`累加器和type的组合器函数`BinaryOperator`。由于身份值类型不限于该`Person`类型，因此我们可以利用此归约法确定所有人的年龄总和：
```Java
Integer ageSum = persons
    .stream()
    .reduce(0, (sum, p) -> sum += p.age, (sum1, sum2) -> sum1 + sum2);

System.out.println(ageSum);  // 76
```
如您所见，结果是76，但是到底发生了什么？让我们通过一些调试输出扩展上面的代码：
```Java
Integer ageSum = persons
    .stream()
    .reduce(0,
        (sum, p) -> {
            System.out.format("accumulator: sum=%s; person=%s\n", sum, p);
            return sum += p.age;
        },
        (sum1, sum2) -> {
            System.out.format("combiner: sum1=%s; sum2=%s\n", sum1, sum2);
            return sum1 + sum2;
        });

// accumulator: sum=0; person=Max
// accumulator: sum=18; person=Peter
// accumulator: sum=41; person=Pamela
// accumulator: sum=64; person=David
```
如您所见，累加器功能完成了所有工作。首先使用初始标识值0和第一人称Max进行调用。在接下来的三个步骤中`sum`，根据最后一个步骤的年龄，人员不断增加，总年龄达到76岁。

等待扫管?？组合器永远不会被调用？并行执行相同的流将揭秘：
```Java
Integer ageSum = persons
    .parallelStream()
    .reduce(0,
        (sum, p) -> {
            System.out.format("accumulator: sum=%s; person=%s\n", sum, p);
            return sum += p.age;
        },
        (sum1, sum2) -> {
            System.out.format("combiner: sum1=%s; sum2=%s\n", sum1, sum2);
            return sum1 + sum2;
        });

// accumulator: sum=0; person=Pamela
// accumulator: sum=0; person=David
// accumulator: sum=0; person=Max
// accumulator: sum=0; person=Peter
// combiner: sum1=18; sum2=23
// combiner: sum1=23; sum2=12
// combiner: sum1=41; sum2=35
```
并行执行此流将导致完全不同的执行行为。现在实际上调用了合并器。由于累加器是并行调用的，因此需要组合器来汇总单独的累加值。

在下一章中，让我们更深入地研究并行流。

### 并行数据流
可以并行执行流，以提高大量输入元素上的运行时性能。并行流使用`ForkJoinPool`可通过静态`ForkJoinPool.commonPool()`方法获得的公共变量。基础线程池的大小最多使用五个线程-取决于可用物理CPU内核的数量：
```Java
ForkJoinPool commonPool = ForkJoinPool.commonPool();
System.out.println(commonPool.getParallelism());    // 3
```
在我的机器上，默认情况下，公共池的并行度为3。可以通过设置以下JVM参数来减小或增大此值：
```Java
-Djava.util.concurrent.ForkJoinPool.common.parallelism=5
```
集合支持`parallelStream()`创建元素并行流的方法。或者，您可以`parallel()`在给定流上调用中间方法，以将顺序流转换为并行对应流。

为了低估并行流的并行执行行为，下一个示例将有关当前线程的信息打印到`sout`：
```Java
Arrays.asList("a1", "a2", "b1", "c2", "c1")
    .parallelStream()
    .filter(s -> {
        System.out.format("filter: %s [%s]\n",
            s, Thread.currentThread().getName());
        return true;
    })
    .map(s -> {
        System.out.format("map: %s [%s]\n",
            s, Thread.currentThread().getName());
        return s.toUpperCase();
    })
    .forEach(s -> System.out.format("forEach: %s [%s]\n",
        s, Thread.currentThread().getName()));
```
通过研究调试输出，我们应该更好地了解哪些线程实际用于执行流操作：
```Java
filter:  b1 [main]
filter:  a2 [ForkJoinPool.commonPool-worker-1]
map:     a2 [ForkJoinPool.commonPool-worker-1]
filter:  c2 [ForkJoinPool.commonPool-worker-3]
map:     c2 [ForkJoinPool.commonPool-worker-3]
filter:  c1 [ForkJoinPool.commonPool-worker-2]
map:     c1 [ForkJoinPool.commonPool-worker-2]
forEach: C2 [ForkJoinPool.commonPool-worker-3]
forEach: A2 [ForkJoinPool.commonPool-worker-1]
map:     b1 [main]
forEach: B1 [main]
filter:  a1 [ForkJoinPool.commonPool-worker-3]
map:     a1 [ForkJoinPool.commonPool-worker-3]
forEach: A1 [ForkJoinPool.commonPool-worker-3]
forEach: C1 [ForkJoinPool.commonPool-worker-2]
```
如您所见，并行流利用通用中所有可用线程`ForkJoinPool`来执行流操作。在连续运行中，输出可能会有所不同，因为实际使用特定线程的行为是不确定的。

让我们通过附加的流操作扩展该示例`sort`：
```Java
Arrays.asList("a1", "a2", "b1", "c2", "c1")
    .parallelStream()
    .filter(s -> {
        System.out.format("filter: %s [%s]\n",
            s, Thread.currentThread().getName());
        return true;
    })
    .map(s -> {
        System.out.format("map: %s [%s]\n",
            s, Thread.currentThread().getName());
        return s.toUpperCase();
    })
    .sorted((s1, s2) -> {
        System.out.format("sort: %s <> %s [%s]\n",
            s1, s2, Thread.currentThread().getName());
        return s1.compareTo(s2);
    })
    .forEach(s -> System.out.format("forEach: %s [%s]\n",
        s, Thread.currentThread().getName()));
```
起初结果可能看起来很奇怪：
```Java
filter:  c2 [ForkJoinPool.commonPool-worker-3]
filter:  c1 [ForkJoinPool.commonPool-worker-2]
map:     c1 [ForkJoinPool.commonPool-worker-2]
filter:  a2 [ForkJoinPool.commonPool-worker-1]
map:     a2 [ForkJoinPool.commonPool-worker-1]
filter:  b1 [main]
map:     b1 [main]
filter:  a1 [ForkJoinPool.commonPool-worker-2]
map:     a1 [ForkJoinPool.commonPool-worker-2]
map:     c2 [ForkJoinPool.commonPool-worker-3]
sort:    A2 <> A1 [main]
sort:    B1 <> A2 [main]
sort:    C2 <> B1 [main]
sort:    C1 <> C2 [main]
sort:    C1 <> B1 [main]
sort:    C1 <> C2 [main]
forEach: A1 [ForkJoinPool.commonPool-worker-1]
forEach: C2 [ForkJoinPool.commonPool-worker-3]
forEach: B1 [main]
forEach: A2 [ForkJoinPool.commonPool-worker-2]
forEach: C1 [ForkJoinPool.commonPool-worker-1]
```
似乎`sort`只在主线程上顺序执行。实际上，`sort`在并行流上，在后台使用了新的Java 8方法`Arrays.parallelSort()`。如Javadoc中所述，此方法决定数组的长度是排序是顺序执行还是并行执行：

*如果指定数组的长度小于最小粒度，则使用适当的Arrays.sort方法对其进行排序。*

回到上`reduce`一节的示例。我们已经发现，组合器函数仅在并行流中调用，而不在顺序流中调用。让我们看看实际涉及到哪些线程：
```Java
List<Person> persons = Arrays.asList(
    new Person("Max", 18),
    new Person("Peter", 23),
    new Person("Pamela", 23),
    new Person("David", 12));

persons
    .parallelStream()
    .reduce(0,
        (sum, p) -> {
            System.out.format("accumulator: sum=%s; person=%s [%s]\n",
                sum, p, Thread.currentThread().getName());
            return sum += p.age;
        },
        (sum1, sum2) -> {
            System.out.format("combiner: sum1=%s; sum2=%s [%s]\n",
                sum1, sum2, Thread.currentThread().getName());
            return sum1 + sum2;
        });
```
控制台输出显示，累加器和合并器函数在所有可用线程上并行执行：
```Java
accumulator: sum=0; person=Pamela; [main]
accumulator: sum=0; person=Max;    [ForkJoinPool.commonPool-worker-3]
accumulator: sum=0; person=David;  [ForkJoinPool.commonPool-worker-2]
accumulator: sum=0; person=Peter;  [ForkJoinPool.commonPool-worker-1]
combiner:    sum1=18; sum2=23;     [ForkJoinPool.commonPool-worker-1]
combiner:    sum1=23; sum2=12;     [ForkJoinPool.commonPool-worker-2]
combiner:    sum1=41; sum2=35;     [ForkJoinPool.commonPool-worker-2]
```
总之，可以说并行流可以为具有大量输入元素的流带来不错的性能提升。但是，请记住，像一些并行流操作`reduce`，并`collect`需要额外的计算（组合操作）时，依次执行其中不需要。

此外，我们了解到所有并行流操作共享相同的JVM范围的common `ForkJoinPool`。因此，您可能要避免实施缓慢的阻塞流操作，因为这有可能减慢应用程序中严重依赖并行流的其他部分的速度。

https://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/#different-kind-of-streams 的翻译版

