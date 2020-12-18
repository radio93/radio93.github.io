---
title: Stream
date: 2020-12-18 14:33:27
tags:
- java
---

Java 8 API添加了一个新的抽象称为流Stream，可以让你以一种声明的方式处理数据。什么意思呢，就是说Stream是一个把集合转换为流来进行操作的一个抽象概念。Stream为我们提供了高效，整洁的代码，不再像之前各种遍历，各种判断，校验。

Stream使用了一种类似于sql查询的方式来对集合进行抽象，他把我们的集合数据当做一个流，放在管道里面，并在每个节点对集合进行操作，比如limit，skip等等。

java 8 为我们提供了两种流

- Stream：串行流
- parallelStream：并行流，因为使用parallelStream要开放多线程和进行多线程间的切换，会消耗额外的资源，而且不是线程安全的，要进行额外的同步加锁操作。所以遍历时如果没有数据库操作的话，是不建议使用并行流的，还是使用stream和普通遍历吧。

为了给下面的例子做准备，我们先定义好一个集合类，下面对这个集合进行操作。

```java
@AllArgsConstructor
@Data
static class People {
    private Integer id;
    private String name;
    private String englishName;
    private Integer age;
    private String sex;
    private Integer orderNo;

}

private static List<People> list = new ArrayList<>(Arrays.asList(
    new People(1, "名字1", "Radio", 10, "男", 1),
    new People(2, "名字2", "MaLi", 20, "男", 3),
    new People(3, "名字3", "Filter", 10, "男", 2),
    new People(4, "名字4", "Abc", 10, "男", 6),
    new People(5, "名字5", "Name", 13, "男", 5),
    new People(6, "名字6", "English", 20, "男", 4),
    new People(1, "名字1", "Radio", 10, "男", 1)
));
```

##### forEach ： 输出

```java
@Test
public void testForEach() {
    list.stream().forEach(a -> System.out.println(a.toString()));
    //forEach用来对流处理完后，输出流；如果仅仅是使用循环功能，不建议使用stream.forEach；使用原生的forEach即可，即list.forEach。
    Random random = new Random();
    random.ints().limit(10).forEach(System.out::println);
}
```

##### fileter：过滤

```java
@Test
public void testFilter() {
    List<Integer> ids = Arrays.asList(1, 2);
    List<People> list1 = list.stream().filter(a -> ids.contains(a.getId())).collect(Collectors.toList());
    System.out.println(list1);
    //注意其和removeIf的区别，removeIf内部的表达式如果成立，则删除掉，而filter表达式成立则输出。这也是stream流操作的特点
    //list.removeIf(a -> !ids.contains(a.getId()));
    //查询第一个

    People people1 = list.stream().filter(a -> ids.contains(a.getId())).findFirst().get();
    System.out.println(people1);
    //查询任意一个
    People people2 = list.stream().filter(a -> ids.contains(a.getId())).findAny().get();
    System.out.println(people2);
    //注意一般这种写法有风险，get很可能会报空指针，如果list为空的话，所以我们推荐下面这种方法
    Optional<People> people3 = list.stream().filter(a -> ids.contains(a.getId())).findFirst();
    //people3.isPresent()用来判断是否存在有效数据
    if (people3.isPresent()) {
        People people = people3.get();
    }
    //是否包含
    boolean flag = list.stream().anyMatch(a -> ids.contains(a.getId()));
    System.out.println(flag);

    //查询数量
    long num = list.stream().filter(a -> ids.contains(a.getId())).count();
    System.out.println(num);
}
```

##### max/min：最大/最小

```java
@Test
public void testCount() {
    //查询年龄最大的人
    Optional<People> people1 = list.stream().max(Comparator.comparingInt(People::getAge));
    if (people1.isPresent()) {
        System.out.println(people1.get());
    }

    //查询年龄最小的人
    Optional<People> people2 = list.stream().min(Comparator.comparingInt(People::getAge));
    if (people2.isPresent()) {
        System.out.println(people2.get());
    }
}
```

##### reduce：归纳

```java
@Test
public void testReduce() {
    //求所有人的年龄和
    Optional<Integer> sum1 = list.stream().map(People::getAge).reduce(Integer::sum);
    sum1.isPresent();
    System.out.println(sum1);
    //和固定值比较大小，输出最大值
    Integer sum2 = list.stream().map(People::getAge).reduce(10, Integer::max);
    System.out.println(sum2);
}
```

##### sorted：排序

```java
@Test
public void testSorted() {
    //正序
    List<People> list1 = list.stream().sorted(Comparator.comparingInt(People::getOrderNo)).collect(Collectors.toList());
    //倒序
    List<People> list2 = 		list.stream().sorted(Comparator.comparingInt(People::getOrderNo).reversed())
        .collect(Collectors.toList());
}
```

##### limit：限制(这俩结合使用，可以做假分页)

```java
@Test
public void testLimit() {
    //获取指定数量
    List<People> list1 = list.stream().limit(2).collect(Collectors.toList());
    //跳过指定数量
    List<People> list2 = list.stream().skip(2).collect(Collectors.toList());
}
```

##### count：统计

```java
@Test
public void testAvg() {
    //前提list不能为null
    long count = list.stream().collect(Collectors.counting());
    long count1 = list.stream().count();
    long count2 = list.size();
    System.out.println(count + "," + count1 + "," + count2);
    //年龄平均值
    double avg = list.stream().collect(Collectors.averagingInt(People::getAge));
    System.out.println(avg);
    //最大值
    Optional<Integer> max1 = list.stream().map(People::getAge).collect(Collectors.maxBy(Integer::compareTo));
    Optional<Integer> max2 = list.stream().map(People::getAge).max(Integer::compareTo);
    System.out.println(max1 + "," + max2);
    //求和
    Integer sum1 = list.stream().collect(Collectors.summingInt(People::getAge));
    int sum2 = list.stream().mapToInt(People::getAge).sum();
    System.out.println(sum1 + "," + sum2);
    //统计
    DoubleSummaryStatistics statistics = list.stream().collect(Collectors.summarizingDouble(People::getAge));
    statistics.getMax();
    statistics.getMin();
    statistics.getCount();
    statistics.getAverage();
    statistics.getSum();
}
```

##### grouping：分组

```java
@Test
public void testGrouping() {
    //分组，按年龄分组
    Map<Integer, List<People>> peopleMap1 = list.stream().collect(Collectors.groupingBy(People::getAge));
    //按是否符合要求来分组，例子为按年龄是否大于10来分组
    Map<Boolean, List<People>> peopleMap2 = list.stream().collect(Collectors.partitioningBy(a -> a.getAge() > 10));

    //按年龄分组 只取ID
    Map<Integer, List<Integer>> peopleMap3 = list.stream().collect(
        Collectors.groupingBy(People::getAge, Collectors.mapping(People::getId, Collectors.toList())));
    //复杂分组，先按年龄分组，再按性别分组
    Map<Integer, Map<String, List<People>>> peopleMap4 =
        list.stream().collect(Collectors.groupingBy(People::getAge, Collectors.groupingBy(People::getSex)));

}
```

##### joining：合并

```java
@Test
public void testJoining() {
    //合并
    String str = list.stream().map(People::getName).collect(Collectors.joining(","));
    System.out.println(str);//名字1,名字2,名字3,名字4,名字5,名字6,名字1
}
```

##### map：映射

```java
@Test
public void testMap() {
    //map：接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素
    List<Integer> ids1 = list.stream().map(People::getId).collect(Collectors.toList());
    //去重复
    List<Integer> ids2 = list.stream().map(People::getId).distinct().collect(Collectors.toList());
    //转set去重复
    Set<Integer> ids3 = list.stream().map(People::getId).collect(Collectors.toSet());

    //给每个人的年龄加10岁
    list.stream().map(a -> {
        a.setAge(a.getAge() + 10);
        return a;
    }).collect(Collectors.toList());
    //上面方法等价于下面这行代码
    //peek的操作是返回一个新的stream的，且设计的初衷是用来debug调试的，因此使用steam.peek()必须对流进行一次处理再产生一个新的stream。
    list.stream().peek(a -> a.setAge(a.getAge() + 10)).collect(Collectors.toList());

    List<String> flatList = Arrays.asList("a,b,c,d", "e,f,g,h");
    //flatMap：接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流
    List<String> newFlatList = flatList.stream().flatMap(a -> {
        String[] strArr = a.split(",");
        return Arrays.stream(strArr);
    }).collect(Collectors.toList());
    System.out.println(flatList.toString());
    System.out.println(newFlatList.toString());
}
```

##### collect：收集

```java
@Test
public void testCollect() {
    //该合并函数有两个参数，第一个参数为当前重复key 之前对应的值，第二个为当前重复key 现在数据的值。
    Map<String, People> nameMap = list.stream().collect(Collectors.toMap(People::getName, b -> b, 
                                                                         (oldValue, newValue) -> newValue));
    System.out.println(nameMap.toString());
}
```

iterate：遍历

```java
@Test
public void testDistinct() {
    String[] arr1 = {"a", "b", "c", "d"};
    String[] arr2 = {"d", "e", "f", "g"};

    List<String> strList = new ArrayList<>(Arrays.asList(arr1));

    //of方法：有两个overload方法，一个接受变长参数，一个接口单一值
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

    //generator方法：生成一个无限长度的Stream，其元素的生成是通过给定的Supplier（这个接口可以看成一个对象的工厂，每次调用返回一个给定类型的对象）
    Stream.generate(() -> Math.random());
    Stream.generate(Math::random);

    // iterate：遍历
    // iterate方法：也是生成无限长度的Stream，和generator不同的是，其元素的生成是重复对给定的种子值(seed)调用用户指定函数来生成的。
    //其中包含的元素可以认为是：seed
    //先获取一个无限长度的正整数集合的Stream，然后取出前10个打印。
    // 切记一定要使用limit方法，不然会无限打印下去。
    List<Integer> list4 = Stream.iterate(1, x -> x + 2).skip(1).limit(10).collect(Collectors.toList());
    System.out.println(list4);
}
```









