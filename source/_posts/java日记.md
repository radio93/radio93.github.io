---
title: java日记
date: 2020-11-09 19:47:33
tags:
- java
---

##### 1，要优先使用基本类型而不是装箱基本类型，要当心无意识的自动装箱

比如下面这段代码

```java
import cn.hutool.core.date.DateUtil;

public class Test {
    public static void main(String[] args) {
        System.out.println(DateUtil.now());
        sum();
        System.out.println(DateUtil.now());

    }

    public static long sum() {
        Long sum = 0L;
        for (long i = 0; i < Integer.MAX_VALUE; i++) {
            sum += i;
        }
        return sum;
    }
}
```

我们运行一下

```java
2020-11-09 19:57:12
2020-11-09 19:57:20
```

这段程序是没有问题的，慢的原因在哪，在`Long sum = 0L`这里，意味着程序构造了大约 2^31 个多余的 Long 实例（大约每次往 Long sum 中增加 long 时构造一个实例） 。将sum 的声明从 Long 改成 long ，我们再试一下

```java
2020-11-09 19:59:44
2020-11-09 19:59:45
```

##### 2，避免创建不必要的对象

比如下面这个例子

```java
public class Test {
    public static void main(String[] args) {
        System.out.println(isNumber("312"));

    }
    static boolean isNumber(String s) {
        return s.matches("^[0-9]*$");
    }
}
```

一个正则表达式，判断传过来的字符串是不是数字，这个方案看起来没有问题，但是如果这个方法使用的非常频繁，那么这种写法就不太合适了，我们先来看看`matcher`这个方法

```java
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

再往里走

```java
public static boolean matches(String regex, CharSequence input) {
    Pattern p = Pattern.compile(regex);
    Matcher m = p.matcher(input);
    return m.matches();
}
```

再往里走

```java
public static Pattern compile(String regex) {
    return new Pattern(regex, 0);
}
```

它在内部为正则表达式创建了一个 `Pattern` 实例，却只用了1次，之后就可以进行垃圾回收了，创建 `Pattenr`实例的成本很高 ，因为需要将正则表达式编译成一个有限状态机（ finite state machine）。

为了提升性能，应该显式地将正则表达式编译成一个 `Pattern` 实例（不可变），让它成为类初始化的一部分，并将它缓存起来，每当调用 `isNumber`方法的时候就重用同一个实例：

```java
import java.util.regex.Pattern;

public class Test {
    private static final Pattern ISNUM = Pattern.compile("^[0-9]*$");
    public static void main(String[] args) {
        System.out.println(isNumber("312"));

    }
    static boolean isNumber(String s) {
        return ISNUM.matcher(s).matches();
    }
}
```

##### 3，覆盖equals必须覆盖hashCode

在每个覆盖了equals方法的类中，也必须覆盖hashCode方法。如果不这样做的话，就会违反Object.hashCode的通用约定，从而导致该类无法结合所有基于散列的集合一起正常运作，这样的集合包括HashMap、HashSet和Hashtable

1. 只要对象的equals方法的比较操作所用到的信息未被修改，那么对同一个对象调用多次其hashCode返回值不变
2. 若两个对象通过equals得到是相等的，那么调用这两个对象任意一个对象的hashCode方法产生整数结果一样
3. 若两个对象通过equals得到是不相等的，那么调用这两个对象任意一个对象的hashCode方法产生的结果也可能相等,但是从提高散列表(hash table)的性能分析，给不相等的对象产生不同的结果会更好

##### 4，Lambda 优先于匿名类，和 java.util.function使用

举个例子，我们看下面这个代码

```java
package effectiveJava.enumTest;

public enum OpemrationOld {
    PLUS("+") {
        public double apply(double x, double y) {
            return x + y;
        }
    },
    MINUS("-") {
        public double apply(double x, double y) {
            return x - y;
        }
    },
    TIMES("*") {
        public double apply(double x, double y) {
            return x * y;
        }
    },
    DIVIDE("/") {
        public double apply(double x, double y) {
            return x / y;
        }
    };
    private final String symbol;

    OpemrationOld(String symbol) {
        this.symbol = symbol;
    }

    public abstract double apply(double x, double y);
}

class testold {
    public static void main(String[] args) {
        double value = OpemrationOld.DIVIDE.apply(1,2);
        System.out.println(value);
    }
}
```

这个代码表示对任意两个数进行加减乘除运算，jdk1.8新增了Lambda表达式后，我们就可以简写这段代码

```java
package effectiveJava.enumTest;

import java.util.function.DoubleBinaryOperator;

public enum Opemration {
    PLUS("+", (x, y) -> x + y),
    MINUS("-", (x, y) -> x - y),
    TIMES("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);


    private final String symbol;
    private final DoubleBinaryOperator op;

    Opemration(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override
    public String toString() {
        return symbol;
    }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}

class test {
    public static void main(String[] args) {
        double value = Opemration.DIVIDE.apply(1, 2);
        System.out.println(value);
    }
}
```

这里我们提一下`DoubleBinaryOperator`这个接口，这个接口是java.util.function包中的，也属于jdk1.8新增的包，用来支持 Java的函数式编程。

那这个接口是什么意思呢 表示：代表了作用于两个double值操作符的操作，并且返回了一个double值的结果。

这个包下提供了非常多的函数

![image-20201110202033145](https://radio93.oss-cn-beijing.aliyuncs.com/gitbub/image-20201110202033145.png?x-oss-process=style/radio93)

| 序号 | **接口**                    | **描述**                                                     |
| ---- | --------------------------- | ------------------------------------------------------------ |
| 1    | **BiConsumer<T,U>**         | 代表了一个接受两个输入参数的操作，并且不返回任何结果         |
| 2    | **BiFunction<T,U,R>**       | 代表了一个接受两个输入参数的方法，并且返回一个结果           |
| 3    | **BinaryOperator<T>**       | 代表了一个作用于于两个同类型操作符的操作，并且返回了操作符同类型的结果 |
| 4    | **BiPredicate<T,U>**        | 代表了一个两个参数的boolean值方法                            |
| 5    | **BooleanSupplier**         | 代表了boolean值结果的提供方                                  |
| 6    | **Consumer<T>**             | 代表了接受一个输入参数并且无返回的操作                       |
| 7    | **DoubleBinaryOperator**    | 代表了作用于两个double值操作符的操作，并且返回了一个double值的结果。 |
| 8    | **DoubleConsumer**          | 代表一个接受double值参数的操作，并且不返回结果。             |
| 9    | **DoubleFunction<R>**       | 代表接受一个double值参数的方法，并且返回结果                 |
| 10   | **DoublePredicate**         | 代表一个拥有double值参数的boolean值方法                      |
| 11   | **DoubleSupplier**          | 代表一个double值结构的提供方                                 |
| 12   | **DoubleToIntFunction**     | 接受一个double类型输入，返回一个int类型结果。                |
| 13   | **DoubleToLongFunction**    | 接受一个double类型输入，返回一个long类型结果                 |
| 14   | **DoubleUnaryOperator**     | 接受一个参数同为类型double,返回值类型也为double 。           |
| 15   | **Function<T,R>**           | 接受一个输入参数，返回一个结果。                             |
| 16   | **IntBinaryOperator**       | 接受两个参数同为类型int,返回值类型也为int 。                 |
| 17   | **IntConsumer**             | 接受一个int类型的输入参数，无返回值 。                       |
| 18   | **IntFunction<R>**          | 接受一个int类型输入参数，返回一个结果 。                     |
| 19   | **IntPredicate**            | 接受一个int输入参数，返回一个布尔值的结果。                  |
| 20   | **IntSupplier**             | 无参数，返回一个int类型结果。                                |
| 21   | **IntToDoubleFunction**     | 接受一个int类型输入，返回一个double类型结果 。               |
| 22   | **IntToLongFunction**       | 接受一个int类型输入，返回一个long类型结果。                  |
| 23   | **IntUnaryOperator**        | 接受一个参数同为类型int,返回值类型也为int 。                 |
| 24   | **LongBinaryOperator**      | 接受两个参数同为类型long,返回值类型也为long。                |
| 25   | **LongConsumer**            | 接受一个long类型的输入参数，无返回值。                       |
| 26   | **LongFunction<R>**         | 接受一个long类型输入参数，返回一个结果。                     |
| 27   | **LongPredicate**           | 接受一个long输入参数，返回一个布尔值类型结果。               |
| 28   | **LongSupplier**            | 无参数，返回一个结果long类型的值。                           |
| 29   | **LongToDoubleFunction**    | 接受一个long类型输入，返回一个double类型结果。               |
| 30   | **LongToIntFunction**       | 接受一个long类型输入，返回一个int类型结果。                  |
| 31   | **LongUnaryOperator**       | 接受一个参数同为类型long,返回值类型也为long。                |
| 32   | **ObjDoubleConsumer<T>**    | 接受一个object类型和一个double类型的输入参数，无返回值。     |
| 33   | **ObjIntConsumer<T>**       | 接受一个object类型和一个int类型的输入参数，无返回值。        |
| 34   | **ObjLongConsumer<T>**      | 接受一个object类型和一个long类型的输入参数，无返回值。       |
| 35   | **Predicate<T>**            | 接受一个输入参数，返回一个布尔值结果。                       |
| 36   | **Supplier<T>**             | 无参数，返回一个结果。                                       |
| 37   | **ToDoubleBiFunction<T,U>** | 接受两个输入参数，返回一个double类型结果                     |
| 38   | **ToDoubleFunction<T>**     | 接受一个输入参数，返回一个double类型结果                     |
| 39   | **ToIntBiFunction<T,U>**    | 接受两个输入参数，返回一个int类型结果。                      |
| 40   | **ToIntFunction<T>**        | 接受一个输入参数，返回一个int类型结果。                      |
| 41   | **ToLongBiFunction<T,U>**   | 接受两个输入参数，返回一个long类型结果。                     |
| 42   | **ToLongFunction<T>**       | 接受一个输入参数，返回一个long类型结果。                     |
| 43   | **UnaryOperator<T>**        | 接受一个参数为类型T,返回值类型也为T。                        |

我们用35  **Predicate<T>**  接受一个输入参数，返回一个布尔值结果。来举个例子

```java
package effectiveJava.functionTest;

import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

public class Java8Tester {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        // Predicate<Integer> predicate = n -> true
        // n 是一个参数传递到 Predicate 接口的 test 方法
        // n 如果存在则 test 方法返回 true

        System.out.println("输出所有数据:");
        eval(list, n -> true);


        // Predicate<Integer> predicate1 = n -> n%2 == 0
        // n 是一个参数传递到 Predicate 接口的 test 方法
        // 如果 n%2 为 0 test 方法返回 true

        System.out.println("输出所有偶数:");
        eval(list, n -> n % 2 == 0);

        // Predicate<Integer> predicate2 = n -> n > 3
        // n 是一个参数传递到 Predicate 接口的 test 方法
        // 如果 n 大于 3 test 方法返回 true

        System.out.println("输出大于 3 的所有数字:");
        eval(list, n -> n > 3);
    }


    public static void eval(List<Integer> list, Predicate<Integer> predicate) {
        for (Integer i : list) {
            if (predicate.test(i)) {
                System.out.println(i);
            }
        }
    }
}

```

当然是用Lambda后，`eval`就可以简写为

```java
public static void eval(List<Integer> list, Predicate<Integer> predicate) {
    list.stream().filter(predicate).forEach(System.out::println);
}
```

java.util.Function 中共有43个接口。别指望能够全部记住它们，但是如果能记住其中6个基础接口，必要时就可以推断出其余接口了。

1. 基础接口作用于对象引用类型
2. Operator 接口代表其结果与参数类型一致的函数
3. Predicate 接口代表带有一个参数 并返回一个 boolean 的函数
4. Function 接口代表其参数与返回的类型不一致的函数
5. Supplier 接口代表没有参数并且返回（或“提供”）一个值的函数
6. Consumer 表的是带有一个函数但不返回任何值的函数，相当于消费掉了其参数

比如下面这个代码

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
list.stream().filter(a -> a > 6).forEach(System.out::println);
```

这是个很常见的用来打印大于6的值，我们来看下这个`filter`接口。

```java
	/**
     * Returns a stream consisting of the elements of this stream that match
     * the given predicate.
     *
     * <p>This is an <a href="package-summary.html#StreamOps">intermediate
     * operation</a>.
     *
     * @param predicate a <a href="package-summary.html#NonInterference">non-interfering</a>,
     *                  <a href="package-summary.html#Statelessness">stateless</a>
     *                  predicate to apply to each element to determine if it
     *                  should be included
     * @return the new stream
     */
	Stream<T> filter(Predicate<? super T> predicate);
```

这里用到的就是我们的函数式编程。再比如

```java
//求最大值
Optional<Integer> max1 = list.stream().reduce(Integer::max);
```

```
Optional<T> reduce(BinaryOperator<T> accumulator);
```

等等，java8给我们提供了大量的示例，告诉我们要尽量使用函数式编程。

jdk8对于许多常用的类都扩展了一些面向函数，lambda表达式，方法引用的功能，使得java面向函数编程更为方便。其中Map.merge方法就是其中一个，merge方法有三个参数，**key**：map中的键，**value**：使用者传入的值，**remappingFunction**：BiFunction函数接口(该接口接收两个值，执行自定义功能并返回最终值)。当map中不存在指定的key时，便将传入的value设置为key的值，当key存在值时，执行一个方法该方法接收key的旧值和传入的value，执行自定义的方法返回最终结果设置为key的值。

举个例子，当map中存在某个key，那么我们把value取出来加上新值，再存进去；传统办法是先判断，如果不存在key，则直接存，如果存在，则取出值加上新值，再把值重新put进去。当使用了merge后，我们的操作就简便多了

```java
package effectiveJava.functionTest;

import java.util.HashMap;
import java.util.Map;

public class MapMergeTest {

    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        map.put("id", 1);
        map.merge("id", 1, (oldValue, newValue) -> oldValue + newValue);
        map.merge("name", 1, (oldValue, newValue) -> oldValue + newValue);
        System.out.println(map.toString());
    }
}
```

运行结果：

```java
{name=1, id=2}
```

与匿名类相比， Lambda 的主要优势在于更加简洁。Java 提供了生成比 Lambda 更简洁函数对象的方法：**方法引用（method reference）**。

比如`map.merge("id", 1, (oldValue, newValue) -> oldValue + newValue);`这段代码我们就可以简写为

```java
map.merge("id", 1, Integer::sum);
```

记住一句话：**只要方法引用更加简洁、清晰，就用方法引用；如果方法引用并不简洁，就坚持使用 Lambda。**

##### 5，Stream

>`Stream`将要处理的元素集合看作一种流，在流的过程中，借助`Stream API`对流中的元素进行操作，比如：筛选、排序、聚合等。

>`Optional`类是一个可以为`null`的容器对象。如果值存在则`isPresent()`方法会返回`true`，调用`get()`方法会返回该对象。

我们先举个实例

```java
package effectiveJava.functionTest;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.junit.Test;

import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class SteamTest {
    static List<Person> personList = new ArrayList<>();

    public static void list() {
        personList.add(new Person("Tom", 8900, 20, "male", "New York"));
        personList.add(new Person("Jack", 7000, 20, "male", "Washington"));
        personList.add(new Person("Lily", 7800, 20, "female", "Washington"));
        personList.add(new Person("Anni", 8200, 20, "female", "New York"));
        personList.add(new Person("Owen", 9500, 20, "male", "New York"));
        personList.add(new Person("Alisa", 7900, 20, "female", "New York"));
    }
    
@Data
@AllArgsConstructor
@NoArgsConstructor
class Person implements Cloneable {
    private String name;  // 姓名
    private int salary; // 薪资
    private int age; // 年龄
    private String sex; //性别
    private String area;  // 地区

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

1. 筛选（filter）

   ```java
   @Test
   public void testFilter() {
       List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
   
       list.stream().filter(a -> a > 6).forEach(System.out::println);
   
       Optional<Integer> first = list.stream().filter(a -> a > 6).findFirst();
   
       Optional<Integer> any = list.stream().filter(a -> a > 6).findAny();
   
       // 是否包含符合特定条件的元素
       boolean anyMatch = list.stream().anyMatch(x -> x < 6);
       System.out.println("匹配第一个值：" + first.get());
       System.out.println("匹配任意一个值：" + any.get());
       System.out.println("是否存在大于6的值：" + anyMatch);
   }
   ```

2. 聚合（max/min/count）

   ```java
   @Test
   public void testMinMaxCount() {
       List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
   	//最小值
       Optional<Integer> min = list.stream().filter(a -> a > 6).min(Integer::compareTo);
       System.out.println(min.get());
   	//最大值
       Optional<Integer> max = list.stream().filter(a -> a > 6).max(Comparator.comparingInt(a -> a));
       System.out.println(max.get());
   	//长度
       long count = list.stream().filter(a -> a > 6).count();
       System.out.println(count);
   }
   ```

3. 映射（map/flatMap）

   ```java
   /**
    * map：接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素。
    * flatMap：接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流。
    */
   @Test
   public void testMap() {
   
       List<String> list = Arrays.asList("abc", "def", "ghi", "jkl");
   
       List<String> strList = list.stream().map(String::toUpperCase).collect(Collectors.toList());
       System.out.println(strList.toString());
   
       List<Integer> numList = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
       List<Integer> newNumList = numList.stream().map(a -> a + 3).collect(Collectors.toList());
       System.out.println(newNumList.toString());
   
       list();
       //不改变
       List<Person> newPersonList = personList.stream().map(a -> {
           Person p = null;
           try {
               p = (Person) a.clone();
           } catch (CloneNotSupportedException e) {
               e.printStackTrace();
           }
           p.setSalary(a.getSalary() + 10000);
           return p;
       }).collect(Collectors.toList());
       System.out.println(personList.toString());
       System.out.println(newPersonList.toString());
   
       //改变
       personList.stream().map(a -> {
           a.setSalary(a.getSalary() + 10000);
           return a;
       }).collect(Collectors.toList());
       System.out.println(personList.toString());
   
   
       List<String> flatList = Arrays.asList("m,k,l,a", "1,3,5,7");
   
       List<String> newFlatList = flatList.stream().flatMap(a -> {
           String[] strArr = a.split(",");
           return Arrays.stream(strArr);
       }).collect(Collectors.toList());
       System.out.println(flatList.toString());
       System.out.println(newFlatList.toString());
   }
   ```

4. 归约（reduce）

   ```java
   /**
    * reduce 归约
    */
   @Test
   public void testReduce() {
       List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
   
       //求和
       Optional<Integer> sum = list.stream().reduce(Integer::sum);
       System.out.println(sum.get());
   
       //求最大值
       Optional<Integer> max1 = list.stream().reduce((a, b) -> a > b ? a : b);
       Optional<Integer> max2 = list.stream().reduce(Integer::max);
       Integer max3 = list.stream().reduce(100, Integer::max);
       System.out.println(max1.get());
       System.out.println(max2.get());
       System.out.println(max3);
   }
   ```

5. 收集（collect）

   ```java
   @Test
   public void testCollect() {
       list();
       List<Person> newList = personList.stream().filter(a -> a.getSalary() > 8000).collect(Collectors.toList());
       System.out.println(newList.toString());
   
       List<String> nameList = personList.stream().filter(a -> a.getSalary() > 8000).map(Person::getName).collect(Collectors.toList());
       System.out.println(nameList.toString());
   
       Map<String, Person> nameMap = personList.stream().collect(Collectors.toMap(Person::getName, b -> b, (oldValue, newValue) -> newValue));
       System.out.println(nameMap.toString());
   }
   ```

6. 统计（count/averaging）

   ```java
   @Test
   public void testAveraging() {
       list();
       //求和
       Long count = personList.stream().collect(Collectors.counting());
       //平均值
       double average = personList.stream().collect(Collectors.averagingDouble(Person::getSalary));
       //最大值
       Optional<Integer> max = personList.stream().map(Person::getSalary).collect(Collectors.maxBy(Integer::compareTo));
       //求和
       Integer sum = personList.stream().collect(Collectors.summingInt(Person::getSalary));
       //统计
       DoubleSummaryStatistics statistics = personList.stream().collect(Collectors.summarizingDouble(Person::getSalary));
       statistics.getMax();
       statistics.getMin();
       statistics.getCount();
       statistics.getAverage();
       statistics.getSum();
   }
   ```

7. 分组（partitioningBy/groupingBy）

   ```java
   @Test
   public void testGroupingBy() {
       list();
       Map<Boolean, List<Person>> map = personList.stream().collect(Collectors.partitioningBy(a -> a.getSalary() > 8000));
       Map<String, List<Person>> sexMap = personList.stream().collect(Collectors.groupingBy(Person::getSex));
   
       Map<String, Map<String, List<Person>>> tmp =
           personList.stream().collect(Collectors.groupingBy(Person::getSex, Collectors.groupingBy(Person::getArea)));
   }
   ```

8. 接合（joining）

   ```java
   @Test
   public void testJoin() {
       list();
       String str = personList.stream().map(Person::getName).collect(Collectors.joining(","));
       System.out.println(str);
   }
   ```

9. 排序（sorted）

   ```java
   /**
    * 排序
    * sorted()：自然排序，流中元素需实现Comparable接口
    * sorted(Comparator com)：Comparator排序器自定义排序
    */
   @Test
   public void testSorted() {
       list();
       //正序
       List<Person> newList = personList.stream().sorted(Comparator.comparingInt(Person::getSalary)).collect(Collectors.toList());
       System.out.println(newList.toString());
       //倒序
       List<Integer> newList2 = personList.stream().map(Person::getSalary).sorted(Comparator.comparingInt(a -> (int) a).reversed()).collect(Collectors.toList());
       System.out.println(newList2.toString());
   }
   ```

10. 提取/组合

    ```java
    @Test
    public void testDistinct() {
        String[] arr1 = {"a", "b", "c", "d"};
        String[] arr2 = {"d", "e", "f", "g"};
    
        List<String> strList = new ArrayList<>(Arrays.asList(arr1));
    
        Stream<String> stream1 = Stream.of(arr1);
        Stream<String> stream2 = Stream.of(arr2);
    
        // concat:合并两个流
        // distinct：去重
        List<String> list = Stream.concat(stream1, stream2).distinct().collect(Collectors.toList());
        System.out.println(list);
    
        // limit：限制从流中获得前n个数据
        List<String> list2 = strList.stream().limit(2).collect(Collectors.toList());
        System.out.println(list2.toString());
    
        // skip：跳过前n个数据
        List<String> list3 = strList.stream().skip(1).limit(2).collect(Collectors.toList());
        System.out.println(list3.toString());
    
        // iterate：遍历
        List<Integer> list4 = Stream.iterate(1, x -> x + 2).skip(1).limit(5).collect(Collectors.toList());
        System.out.println(list4);
    }
    ```

##### 6，同步访问共享的可变数据

举个例子

```java
package effectiveJava.functionTest;

import java.util.concurrent.TimeUnit;

public class StopThread {
    private static boolean stopRequest;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            int i = 0;
            while (!stopRequest) {
                i++;
            }
        });
        t.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequest = true;
    }
}
```

你可能期待这个程序运行大约一秒钟左右，之后主线程将 stopRequest 设置为 true ，致使后台线程的循环终止。但是实际上，这个程序永远不会终止：因为后台 线程永远在循环！

问题在于，由于没有同步，就不能保证后台线程何时‘看到’主线程对 stopRequest 的值所做的改变。没有同步，虚拟机将以下代码：

```java
while (!stopRequest) {
    i++;
}
```

转变成这样：

```java
if (!stopRequest){
    while(true){
		i++;    
	}
}
```

这种优化称作**提升（ hoisting ）**，正是 OpenJDK Server VM的工作 结果是一个**活性失败 (liveness failure ）**：这个程序并没有得到提升。

修正这个问题的一种方式是同步访问`stopRequest`域。

```java
package effectiveJava.functionTest;

import java.util.concurrent.TimeUnit;

public class StopThread {
    private static volatile boolean stopRequest;

    private static synchronized void setStopRequest() {
        stopRequest = true;
    }

    private static synchronized boolean getStopRequest() {
        return stopRequest;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            int i = 0;
            while (!getStopRequest()) {
                i++;
            }
        });

        t.start();
        TimeUnit.SECONDS.sleep(1);
        setStopRequest();
    }
}
```

注意，我们的**读和写操作都要同步**，否则无法保证同步起作用。

还有一种方式就是`volatile`关键字，虽然 volatile 修饰符不执行互斥访问，但它可以保证任何一个线程在读取该域的时候都将看到最近刚刚被写入的值：

```java
package effectiveJava.functionTest;

import java.util.concurrent.TimeUnit;

public class StopThread {
    private static volatile boolean stopRequest;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            int i = 0;
            while (!stopRequest) {
                i++;
            }
        });

        t.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequest = true;
    }
}
```

在使用volatile关键字的时候，我们需要特别注意，不能使用i++（增量操作符）的操作，因为这个操作**不是原子的**。这个操作域中执行两项操作：首先它读取值，然后写回一个新值，相当于原来的值再加上1。如果第二个线程在第一个线程读取旧值和写回新值期间读取这个域，第二个线程就会与第一个线程一起看到同一个值，并返回相同的序列号，这就是**安全性失败（ safety failure ）**：这个程序会计算出错误的结果。修复方法是用synchronized来代替volatile。当然最好的办法是替换成原子类java.util.concurrent.atomic。

```java
Atomiclang i =new Atomiclong(); 
i.getAndincrement();
```

总而言之， **当多个线程共享可变数据的时候，每个读或者写数据的线程都必须执行同步。**