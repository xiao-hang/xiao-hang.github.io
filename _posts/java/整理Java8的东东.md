## Java8的东东-掉头发整理跑路必备Java8新特性罒ω罒

### 参考

> 先把参考文章贴出来，这些文章都很好的。各种具体的详情，请查看参考原文会更具体。
>
> 如果只是当做工具API查阅的话，可以直接按目录查找。

* 博客园-[JAVA8新特性(吐血整理)](https://www.cnblogs.com/liuxiaozhi23/p/10880147.html)
* 并发编程网-[Java 8 特性 – 终极手册](http://ifeve.com/java-8-features-tutorial/)
* 详细各个接口介绍【系列文章】【灰常赞的】https://blog.csdn.net/qq_28410283/article/details/80625482
* 常用操作：https://blog.csdn.net/wjavadog/article/details/107737755
* 静态类介绍：https://blog.csdn.net/qq_28410283/article/details/80952768
* 时间API参考详细文章【超级详细，推荐看以下参考文章】：

  * **https://cloud.tencent.com/developer/article/1649969**
  * **https://cloud.tencent.com/developer/article/1649882**
* https://blog.csdn.net/w605283073/article/details/92418504
* https://www.cnblogs.com/dreamroute/p/13706784.html
* https://www.jianshu.com/p/b3c4dd85901e

---

### 说明

虽然工作上已经升级到Java8挺久的了，但是还没有，完整使用上它的一些特性。

尤其是**Lambda 表达式**，一些的写法灰常的简洁，虽然效率上并不一定很高。╮(╯_╰)╭ 

> (毕竟源码上还是常规操作的封装语法糖，但它优雅啊 罒ω罒)

---

### Lambda表达式和函数式接口【重点】

> 也叫做闭包,它允许我们将一个函数当作方法的参数（传递函数），或者说把代码当作数据，这是每个函数式编程者熟悉的概念。

#### 语法格式定义

<span style="color:red;" >**(parameters) -> expression** 或  **(parameters) ->{statements; }**</span>

函数接口是一种只有一个方法的接口，像这样地，函数接口可以隐式地转换成lambda表达式。

<span style="color:red;" >**函数接口注解@FunctionalInterface**</span>，java里所有现存的接口都已经加上

> java.lang.Runnable 和java.util.concurrent.Callable是函数接口两个最好的例子

#### 简单的栗子

```java
Arrays.asList( "a", "b", "d" ).forEach( e -> System.out.println( e ) );
```

> <span style="color:violet;" >(方法参数)->{方法体}，就是一个定义匿名方法的东东。</span>

---

---

### 函数式接口：【重点】

> Lambda的语法主要的作用就是作为定义一个函数。
>
> **而真正让Java8的lambda用起来的，则是各种提供的函数接口与高效好用的新API。**

#### **常用接口：**

#### **Supplier** :【提供者】

仅提供get方法，可以看做一个创建对象的工厂。

* ```java
  //使用Lambda表达式提供Supplier接口实现，返回OK字符串
  Supplier<String> stringSupplier = ()->"OK";
  //使用方法引用提供Supplier接口实现，返回空字符串
  Supplier<String> supplier = String::new;
  System.out.println( stringSupplier.get());
  //Supplier是提供一个数据的接口。这里我们实现获取一个随机数
  Supplier<Integer> random = ()->ThreadLocalRandom.current().nextInt();
  System.out.println(random.get());
  ```

#### **Predicate**:【断言】

* ```java
  //Predicate接口是输入一个参数，返回布尔值。
  // 数字类型 判断值是否大于5
  Predicate<Integer> predicate = x -> x > 5;
  System.out.println(predicate.test(10));	//true
  // 字符串的非空判断
  Predicate<String> predicateStr = x -> null == x || "".equals(x);
  System.out.println(predicateStr.test(""));	//true
  //接口默认方法还有：and ,or , negate 作为链式调用时使用。
  Predicate<String> predicateStr2 = x -> x == "0";
  System.out.println((predicateStr.and(predicateStr2).test("1")));	//false
  ```

#### **Consumer**:【消费】

* ```java
  //Consumer接口是消费一个数据。我们通过andThen方法组合调用两个Consumer，输出两行abcdefg
  Consumer<String> println = System.out::println;
  println.andThen(println).accept("test");
  ```

* 接口其实只有`accept`方法，并且**无返回值**的。是作为消费者的存在，不需要返回的。

* `andThen`方法，则作为消息，往后传递给下一个消费者进行处理的默认方法。

#### **Function** : 【方法】

* ```java
  //Function接口是输入一个数据，计算后输出一个数据。我们先把字符串转换为大写，然后通过andThen组合另一个Function实现字符串拼接
  //定义：public interface Function<T, R>
  Function<String, String> upperCase = String::toUpperCase;
  Function<String, String> duplicate = s -> s.concat(s);
  System.out.println(upperCase.andThen(duplicate).apply("test"));  //TESTTEST
  ```

* 说明：这个接口，是要传入的泛型T参数，然后做业务操作，然后泛型R；

* `andthen`方法：先做本接口的apply操作，再做传入的Function类型的参数的apply操作

* `compose`与`andthen`方法顺序相反

#### **BinaryOperator**：【二元操作符】

* ```java
   //BinaryOperator是输入两个同类型参数，输出一个同类型参数的接口。这里我们通过方法引用获得一个整数加法操作，通过Lambda表达式定义一个减法操作，然后依次调用
   BinaryOperator<Integer> add = Integer::sum;
   BinaryOperator<Integer> subtraction = (a, b) -> a - b;
   System.out.println(subtraction.apply(add.apply(1, 2), 3));// 3
   ```

* 源码中提供默认方法：`minBy`，`maxBy` ： 作为比较器，创建二元操作,返回最大最小值；

* ```java
   Comparator<Integer> comparator = (t1, t2) -> t1.compareTo(t2);
   BinaryOperator<Integer> bi = BinaryOperator.minBy(comparator);
   System.out.println(bi.apply(5, 6));  // 5
   ```

#### **UnaryOperator**：【一元运算】

* ```java
   UnaryOperator<Integer> dda = x -> x + 1;
   System.out.println(dda.apply(10));// 11
   UnaryOperator<String> ddb = x -> x + 1;
   System.out.println(ddb.apply("aa"));// aa1
   ```

* 与function接口不同，感觉就只类型的定义了，此接口参数与返回为同类型的。

#### **BiConsumer**：【消费】

* 与Consumer的不同之处在与，接受的参数为两个。栗子：作为遍历map的方法就很好用。

* ```java
   Map<String, String> map = new HashMap<>();
   map.put("a", "a");
   map.put("b", "b");
   map.forEach((k, v) -> {
       System.out.println(k);
       System.out.println(v);
   });
   // 可以看一下这个方法的定义，需要的是BiConsumer接口的实现
   // default void forEach(BiConsumer<? super K, ? super V> action) { 
   ```

> \(^o^)/~  ok 就这些了。
>
> 如果没有自己去封装一些公共的操作对象等，这些接口很少会去写的。
>
> 但是，了解这些接口，很方便来调用Java各种API中的方法。看一下实现的接口是以上哪一个，就能知道如何使用了。毕竟，封装好的API，尤其是集合的，以上接口用的很多的。

---

### Stream  接口操作【重点】

> 【流】是Java API 新增的东东，用于处理集合等数据的迭代，作为数据处理工厂。

Stream的接口定义中，包含了30+的方法，涉及到了各种的操作，灰常的多。这些就是要学的东东了！！

#### 创建流

* Arrays.stream，我们可以通过Arrays的静态方法，传入一个泛型数组，创建一个流

  * ```java
    String[] dd = { "a", "b", "c" };
    Arrays.stream(dd).forEach(System.out::print);// abc
    ```

* Stream.of ： Stream的静态方法，传入一个泛型数组，或者多个参数，创建一个流，内部也是调用`Arrays.stream`的。

  * ```java
    Stream.of(dd).forEach(System.out::print);// abc
    ```

* **Collection.stream** ,集合的默认方法可以创建流的。**包括继承Collection的接口，如：Set，List，Map，SortedSet 等，大多的集合都是实现这个接口的**

  * ```java
    Arrays.asList(dd).stream().forEach(System.out::print);// abc
    ```

* Stream.iterate ：静态方法，迭代器的形式，创建数据流。

  * ```java
    // 查看其接口定义
    public static<T> Stream<T> iterate(final T seed, final UnaryOperator<T> f) {
     .......
        public boolean hasNext() {
            return true;
         }
         public T next() {  // 主要就是这个方法，无限的迭代
             return t = (t == Streams.NONE) ? seed : f.apply(t);
         }
     .......
    // 以上，可以了解：传入两个参数，
         // 一个泛型T对象，表示数据的起始，
         // 一个函数式接口 **UnaryOperator** 【一元运算接口】
    // 并且hasNext：true;迭代器，会一直执行下去，创建的数据集合的值为泛型T对象；
    // 这样一直创建无限个对象的流，也成为无限流；
    ```

  * ```java
    // 使用栗子：
    Stream.iterate(0, x -> x + 1).limit(10).forEach(System.out::print);// 0123456789
    ```

* Stream.generate ：静态方法；

  * ```java
    // 方法定义：
    public static<T> Stream<T> generate(Supplier<T> s) {
            Objects.requireNonNull(s);
            return StreamSupport.stream(
                    new StreamSpliterators.InfiniteSupplyingSpliterator.OfRef<>(Long.MAX_VALUE, s), false);
        }
    // 可以看到，传入一个函数式接口 Supplier 【提供者】 ,也是无限生成对象的集合流，也是一个无限流；
    ```

  * ```java
    // 使用栗子：
    Stream.generate(() -> "x").limit(10).forEach(System.out::print);// xxxxxxxxxx
    ```

好的，主要的创建方式就这些了。

> 这里有一个栗子，是说明这种流的操作，在一些数据处理上有很好的优势的。
>
> 应用场景：我们再做B端系统的时候，会遇到很多的统计类的需求，会用到百度的echarts插件，比如曲线图，在x抽，固定的况下（按月统计 1号-31号，或者按年统计1月-12月，或者按天24个小时的刻度），那么我就需要创建一个这个数组，或者集合，代码如下：
>
> ```java
> public static void main(String[] args) {
> 		System.out.println(buildList(100));
> 		System.out.println(buildIterate(100));
> 	}
> public static List<Integer> buildList(final int size) {
> 	List<Integer> list = new ArrayList<>(size);
> 	for (int i = 1; i <= size; i++) {
> 		list.add(i);
> 	}
> 	return list;
> }
> public static List<Integer> buildIterate(final int size) {
> 	return Stream.iterate(1, x -> ++x).limit(size).collect(Collectors.toList());
> }
> ```
> 所以，可以看到，这种流式的写法，还是很优雅的 罒ω罒  

---

#### 操作

> 这个操作指的就是，Stream接口定义中的各种操作了。

在所有操作中，可以分为中间操作和终端操作。

* **中间操作**：**返回对象本身`Stream<T>` ,可以连接操作。**

  * ```java
    //栗子：
    StringBuilder sb = new StringBuilder();
    sb.append("a").append("b").append("c");
    ```

* **终端操作**：返回最终接口，非本身对象。比如：forEach等。

---

**接下来，就具体各种的方法介绍定义了。**

##### 【中间操作】

##### filter : 【过滤器】【中间】

定义：`Stream<T> filter(Predicate<? super T> predicate);`

参数：Predicate 【断言接口】【单个参数，返回布尔值】

栗子：

```java
String[] dd = { "a", "b", "c" };
Stream<String> stream = Arrays.stream(dd);
stream.filter(str -> str.equals("a")).forEach(System.out::println);//返回字符串为a的值
```

---

##### map : 【编排】【中间】

> 这里的中文名字：【编排】，不好取，我觉得当做编排比较符合我的认知。

说明：处理流中的每一个数据的时候，会先进行一次该方法操作。

场景：数据编排，类型转换，改变组装数据等。

定义：`<R> Stream<R> map(Function<? super T, ? extends R> mapper);`

参数：Function 【参数T：泛型参数，返回R：泛型参数】

栗子：

```java
Integer[] dd = { 1, 2, 3 };
Stream<Integer> stream = Arrays.stream(dd);
// stream.map(str -> Integer.toString(str)).forEach(str -> {  // 等同简写
stream.map(str -> {
    System.out.println("xx:"+str);// 1 ,2 ,3
    System.out.println("xx:"+str.getClass());// class java.lang.String
    return Integer.toString(str);
}).forEach(str -> {
    System.out.println(str);// 1 ,2 ,3
    System.out.println(str.getClass());// class java.lang.String
});
// 结构
// xx:1
// xx:class java.lang.Integer
// 1
// class java.lang.String
// xx:2
// xx:class java.lang.Integer
// 2
// class java.lang.String
```

---

##### flatMap：【平整编排】

说明：处理单个数据，转为流输出的操作。

场景：嵌套集合，在处理时，需要把原子数据，做为流输出处理的场景。

定义：`<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);`

参数：Function 【参数T：泛型参数，返回R：Stream流；】

栗子：

```java
String[] strs = { "aaa", "bbb", "ccc" };
Arrays.stream(strs).map(str -> str.split("")).forEach(System.out::println);
// Ljava.lang.String;@53d8d10a 流中的单个元素为数组：[a,a,a] 这一整个对象，类型是数组;
Arrays.stream(strs).map(str -> str.split("")).flatMap(Arrays::stream).forEach(System.out::println);
// aaabbbccc   这里的处理 把 [a,a,a] 平整为 单个元素流  a->a->a 这个样子的，类型为string;
Arrays.stream(strs).map(str -> str.split("")).flatMap(str -> Arrays.stream(str)).forEach(System.out::println);
// aaabbbccc  与上面那个以上的
// 结果：
// [Ljava.lang.String;@27bc2616
// [Ljava.lang.String;@3941a79c
// [Ljava.lang.String;@506e1b77
// a
// a
// a
// b
// b
// b
// ......
```

---

##### distinct :【去重】

定义：`Stream<T> distinct();`

栗子：

```java
Arrays.asList(3, 1, 2, 1).stream().distinct().forEach(System.out::println);
// 3 1 2
```

---

##### sorted：【排序】

定义：

```java
Stream<T> sorted();
Stream<T> sorted(Comparator<? super T> comparator);
```

参数：空参数或者比较器Comparator

说明：由于栗子中排序用到了**比较器**，在栗子之后，还需要**单独说明一下的**。有几种的情况。

栗子：

```java
// 如果数据流为数字等基础类型，本生可以比较的那种 ，则直接使用空参数排序
Arrays.asList(3, 1, 2, 1).stream().distinct().sorted().forEach(System.out::println);
// 结果： 1 2 3 
// 如果数据为，集合或者对象 进行排序的话，则需要使用带参数的
list.stream().map(emp -> emp.getSalary()).sorted().forEach(System.out::println);
list.stream().sorted(Comparator.comparing(Emp::getName)).forEach(System.out::println);
// 结果：
// 1000.0    这里是使用map做了编排，只把对比的值作为流处理，而非对象本身
// 2000.0
// 4000.0
// LambdaTest$Emp@1e80bfe8  这里虽然是排序了，但是流中的数据，依旧是对象本身
// LambdaTest$Emp@66a29884
// LambdaTest$Emp@4769b07b
    
// 类定义  与下面多个栗子公用 
public static List<Emp> list = new ArrayList<>();
	static {
	    list.add(new Emp("xiaoHong1", 20, 1000.0));
        list.add(new Emp("xiaoHong4", 35, 4000.0));
        list.add(new Emp("xiaoHong2", 25, 2000.0));
//        list.add(new Emp("xiaoHong3", 30, 3000.0));
//        list.add(new Emp("xiaoHong5", 38, 5000.0));
//        list.add(new Emp("xiaoHong6", 45, 9000.0));
//        list.add(new Emp("xiaoHong7", 55, 10000.0));
//        list.add(new Emp("xiaoHong8", 42, 15000.0));
	}

	public static class Emp {
		private String name;
 
		private Integer age;
 
		private Double salary;
 
		public Emp(String name, Integer age, Double salary) {
			super();
			this.name = name;
			this.age = age;
			this.salary = salary;
		}
 
		public String getName() {
			return name;
		}
 
		public void setName(String name) {
			this.name = name;
		}
 
		public Integer getAge() {
			return age;
		}
 
		public void setAge(Integer age) {
			this.age = age;
		}
 
		public Double getSalary() {
			return salary;
		}
 
		public void setSalary(Double salary) {
			this.salary = salary;
		}
 
	}

```

> 这里一小部分，跳到比较器上面，记录一下，存在的写法。

###### Comparator.comparing：【比较器排序】

使用的是Comparator.comparing 进行排；【**此方法可以再流处理中的sort直接使用**】

这个静态方法创建 `Comparator`的定义如下：

```java
    public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor)
    {
        Objects.requireNonNull(keyExtractor);
        return (Comparator<T> & Serializable)
            (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
    }
//第二组定义：双参数
    static <T, U> Comparator<T> comparing(Function<? super T, ? extends U> var0, Comparator<? super U> var1) {
        Objects.requireNonNull(var0);
        Objects.requireNonNull(var1);
        return (Comparator)((Serializable)((var2x, var3) -> {
            return var1.compare(var0.apply(var2x), var0.apply(var3));
        }));
    }
```

这里参数为`Function` 接口，并且支持lambda的写法。所以就可以写成这个样子：**【栗子】**

```java
list.stream().sorted(Comparator.comparing(Emp::getName)).forEach(System.out::println);
// 或者：    *.sorted(Comparator.comparing(e -> e.getName()))  
// 第二组定义 ：双参数
list.stream().sorted( Comparator.comparing(
                Emp::getName, (s1, s2) -> {
                    return s2.compareTo(s1);
                })).forEach(e->System.out.println(e.getName()));
// 由于这里是：s2.compareTo(s1)  则结果为，倒序：
// xiaoHong4
// xiaoHong2
// xiaoHong1
```



###### Collections 【集合本身排序】

**定义为：**

```java
    public static <T> void sort(List<T> list, Comparator<? super T> c) {
        list.sort(c);
    }
// 调用的为list接口的sort的方法。
// 看一下这个list.sort方法的定义：
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```

可以看到，这里的排序为`list`本身接口，并且无返回参数，是本身集合LIST的排序。

这里的定义中，可以看到`Object[] a = this.toArray()` 因此存在使用2个参数的情况。

**栗子：**

```java
list.sort((o1, o2) -> o1.getName().compareTo(o2.getName()));
Collections.sort(list,(o1, o2) -> o1.getName().compareTo(o2.getName()));
list.stream().forEach(e->System.out.println(e.getName()));
// 效果是一样的 无所谓  
// 当然，这里的参数是<Comparator> ，当然也是支持这种写法的了：
list.sort(Comparator.comparing(e -> e.getName()));
list.sort(Comparator.comparing(Employee::getName));
```

> 好的，比较器排序的东西，就这么多了，如果还有其他的写法啥的，也是我不知道的了 ╮(╯_╰)╭ 

---

##### peek：【对对象的进行操作】

定义：`Stream<T> peek(Consumer<? super T> action);`

参数：`<Consumer>` 消费者的接口，对单个流中的对象进行操作，毕竟此接口为单参数无返回的。

栗子：

```java
// list.add(new Emp("xiaoHong4", 35, 4000.0));
list.stream().filter(emp -> {
    return emp.getAge() > 30;
}).peek(emp -> {
    emp.setSalary(emp.getSalary() * 1.5);
}).forEach(e->System.out.println(e.getSalary()));
// 结果：6000.0 
```

---

##### limit：【截断-取前N个】

##### skip：【截断-忽略前N个】

> 这两个就一起了吧，都是在创建流的时候做限制使用，当然，其他情况也是可以用的。

栗子：

```java
// 数字从1开始迭代（无限流），下一个数字，是上个数字+1，忽略前5个 ，并且只取10个数字
// 原本1-无限，忽略前5个，就是1-5数字，不要，从6开始，截取10个，就是6-15
Stream.iterate(1, x -> ++x).skip(5).limit(10).forEach(System.out::println);
// 在普通流处理中使用，也是一样的意思
Arrays.asList(3, 1, 2, 1).stream().distinct().sorted().limit(2).forEach(System.out::println);
```

---

##### 【终端操作】

##### 【遍历】：forEach 和 forEachOrdered

定义：

```java
void forEach(Consumer<? super T> action);  
void forEachOrdered(Consumer<? super T> action);
```

两个接口的定义是一样的。

参数：`<consumer>` 【消费者】的接口类型。 

栗子： 【这栗子就比较简单了，看看就好】

```java
List<String> strs = Arrays.asList("a", "b", "c");
        strs.stream().forEachOrdered(System.out::print);//abc
        System.out.println();
        strs.stream().forEach(System.out::print);//abc
        System.out.println();
        strs.parallelStream().forEachOrdered(System.out::print);//abc
        System.out.println();
        strs.parallelStream().forEach(System.out::print);//bca
// parallelStream 属于并行流；
// forEachOrdered表示严格按照顺序取数据，forEach在并行中，随机排列了；
```

> 这个也可以看出来，在并行的程序中，如果对处理之后的数据，没有顺序的要求，使用forEach的效率，肯定是要更好的

---

##### 【转数组】: toArray

> 这好像懒得说明了，直接看个例子吧，反正就字面意思。都差不多的。

```java
List<String> strs = Arrays.asList("a", "b", "c");
String[] dd = strs.stream().toArray(str -> new String[strs.size()]);
String[] dd1 = strs.stream().toArray(String[]::new);
Object[] obj = strs.stream().toArray();
String[] dd2 = strs.toArray(new String[strs.size()]);
Object[] obj1 = strs.toArray();
// 结果是一样额  ╮(╯_╰)╭ ["a", "b", "c"] 
```

---

##### 【最值】：min,max,findFirst,findAny

定义：

```java
	Optional<T> min(Comparator<? super T> comparator);  
    Optional<T> max(Comparator<? super T> comparator);
	Optional<T> findFirst();    
    Optional<T> findAny();
```

> 可以看到这里，最大最小值，需要的参数和排序是一样的`<Comparatoe>` 【比较器】

栗子：

```java
List<String> strs = Arrays.asList("d", "b", "a", "c", "a","d");
Optional<String> min = strs.stream().min(Comparator.comparing(Function.identity()));
Optional<String> max = strs.stream().max((o1, o2) -> o1.compareTo(o2));
System.out.println(String.format("min:%s; max:%s", min.get(), max.get()));// min:a; max:d
Optional<String> aa = strs.stream().filter(str -> !str.equals("a")).findFirst();
Optional<String> bb = strs.stream().filter(str -> !str.equals("a")).findAny();
Optional<String> aa1 = strs.parallelStream().filter(str -> !str.equals("a")).findFirst();
Optional<String> bb1 = strs.parallelStream().filter(str -> !str.equals("a")).findAny();
System.out.println(aa.get() + "===" + bb.get());// d===d
System.out.println(aa1.get() + "===" + bb1.get());// d===b or d===c or d===d 【最后一个我是没测到过】
```

<span style="color:yellow;" >**注意：这里有一个问题，就是在findAny方法，是可以返回`Null`的。**</span>

```java
// 所以，findAny的后续方法需要注意
strs.parallelStream().filter(str -> str.equals("v")).findAny().get(); // 这种写法是不行，会报错的
// 建议需要这么写
strs.parallelStream().filter(str -> !str.equals("v")).findAny().orElseGet(()->"null"); 
strs.parallelStream().filter(str -> !str.equals("v")).findAny().orElse（defaultVal); // 这样也行
```

---

##### 【长度与匹配】:count,-Match

定义：

```java
// 与lise接口size一样的，返回的是集合流的元素长度。
	long count();     
// 一下参数为【断言】
    boolean anyMatch(Predicate<? super T> predicate);  // 判断的条件里，任意一个元素成功，返回true
    boolean allMatch(Predicate<? super T> predicate);  // 判断条件里的元素，所有的都是，返回true
    boolean noneMatch(Predicate<? super T> predicate); // 判断条件里的元素，所有的都不是，返回true
```

栗子：

```java
List<String> strs = Arrays.asList("a", "a", "a", "a", "b");
        boolean aa = strs.stream().anyMatch(str -> str.equals("a"));
        boolean bb = strs.stream().allMatch(str -> str.equals("a"));
        boolean cc = strs.stream().noneMatch(str -> str.equals("a"));
        long count = strs.stream().filter(str -> str.equals("a")).count();
        System.out.println(aa);// TRUE
        System.out.println(bb);// FALSE
        System.out.println(cc);// FALSE
        System.out.println(count);// 4
```

---

##### 【归约-折叠】：reduce

介绍：reduce 是一种归约操作，将流归约成一个值的操作叫做归约操作，用函数式编程语言的术语来说，这种称为折叠（fold）；

<span style="color:yellow;" >**注意：这里需要注意，2，3个参数的方法，并行流的情况。【例子中有解释】**</span>

定义：

```java
T reduce(T identity, BinaryOperator<T> accumulator);
Optional<T> reduce(BinaryOperator<T> accumulator);

<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);

// 接口实现
@Override
public final P_OUT reduce(final P_OUT identity, final BinaryOperator<P_OUT> accumulator) {
    return evaluate(ReduceOps.makeRef(identity, accumulator, accumulator));
}

@Override
public final Optional<P_OUT> reduce(BinaryOperator<P_OUT> accumulator) {
    return evaluate(ReduceOps.makeRef(accumulator));
}

@Override
public final <R> R reduce(R identity, BiFunction<R, ? super P_OUT, R> accumulator, BinaryOperator<R> combiner) {
    return evaluate(ReduceOps.makeRef(identity, accumulator, combiner));
}
```

参数：T identity：表示初始值。接口BinaryOperator：表示二元计算规则。

栗子：

```java
List<Integer> numbers = Stream.iterate(1, x -> x + 1).limit(100).collect(Collectors.toList());
Integer aa = 0;
for (Integer i : numbers) {
    aa += i;
}
Optional<Integer> dd1 = numbers.stream().reduce( (a, b) -> a + b);
Integer dd2 = numbers.stream().reduce(100, (a, b) -> a + b);
Integer dd3 = numbers.stream().reduce(100,(a, b) -> a + b ,(a, b) -> a - b);

Optional<Integer> dx1 = numbers.parallelStream().reduce((a, b) -> a + b);
Integer dx2 = numbers.parallelStream().reduce(100,(a, b) -> a + b);
Integer dx3 = numbers.parallelStream().reduce(100,(a, b) -> a + b,(a, b) -> a - b);

System.out.println(aa);
System.out.println("=======================");
System.out.println(dd1.get());
System.out.println(dd2);
System.out.println(dd3);    // 单线程，所有第三个参数是无效的
System.out.println("=======================");
System.out.println(dx1.get());
// 这里每个线程的计算都是100开始，所以每个线程的值都会+100  所以有几十个线程的话就多了N*100了
System.out.println(dx2);  
// 第三个参数用于合并多线的结果，这里是(a, b) -> a - b，线程相减操作
System.out.println(dx3);       

// 结果：
//5050
//===================
//5050
//5150
//5150
//=====================
//5050
//8650
//0
```

---

##### 【收集器】：collect

定义：

```java
<R> R collect(Supplier<R> supplier,BiConsumer<R, ? super T> accumulator,BiConsumer<R, R> combiner);
<R, A> R collect(Collector<? super T, A, R> collector);
```

> 第一个仨参数的略过，反正用不到。就用单参数的这个就够了。

参数：单参数，`<Collector>` ，这个类是有**静态工厂类**的：`Collectors` ;

栗子：具体的方法直接代码上看看就懂了的罒ω罒

* 结果转换：list，set，map，count，sum，aveAges
* 综合处理：max,min,sum ,Average
* 字符拼接：前后缀字符拼接每一流数据。
* 最值获取：方法参数不一样，按需使用吧
* 归约操作，分组操作

```java
// ============================================================================================
// 转list
List<String> names = list.stream().map(emp -> emp.getName()).collect(Collectors.toList());
// 转set
Set<String> address = list.stream().map(emp -> emp.getName()).collect(Collectors.toSet());
// 转map，需要指定key和value，Function.identity()表示当前的Emp对象本身
Map<String, Emp> map = list.stream().collect(Collectors.toMap(Emp::getName, Function.identity()));
// 计算元素中的个数
Long count = list.stream().collect(Collectors.counting());
// 数据求和 summingInt summingLong，summingDouble
Integer sumAges = list.stream().collect(Collectors.summingInt(Emp::getAge));
// 平均值 averagingInt,averagingDouble,averagingLong
Double aveAges = list.stream().collect(Collectors.averagingInt(Emp::getAge));
// ============================================================================================
// 综合处理的，求最大值，最小值，平均值，求和操作
// summarizingInt，summarizingLong,summarizingDouble
IntSummaryStatistics intSummary = list.stream().collect(Collectors.summarizingInt(Emp::getAge));
System.out.println(intSummary.getAverage());// 26.666666666666668
System.out.println(intSummary.getMax());// 35
System.out.println(intSummary.getMin());// 20
System.out.println(intSummary.getSum());// 80
// ============================================================================================
// 连接字符串,当然也可以使用重载的方法，加上一些前缀，后缀和中间分隔符
String strEmp = list.stream().map(emp -> emp.getName()).collect(Collectors.joining());
String strEmp1 = list.stream().map(emp -> emp.getName()).collect(Collectors.joining("-中间的分隔符-"));
String strEmp2 = list.stream().map(emp -> emp.getName()).collect(Collectors.joining("-中间的分隔符-", "前缀*", "&后缀"));
System.out.println(strEmp);// xiaoHong1xiaoHong4xiaoHong2
// xiaoHong1-中间的分隔符-xiaoHong4-中间的分隔符-xiaoHong2
System.out.println(strEmp1);
// 前缀*xiaoHong1-中间的分隔符-xiaoHong4-中间的分隔符-xiaoHong2&后缀
// ============================================================================================
 // maxBy 按照比较器中的比较结果刷选 最大值
 Optional<Integer> maxAge = list.stream().map(emp -> emp.getAge())
         .collect(Collectors.maxBy(Comparator.comparing(Function.identity())));
 // 最小值
 Optional<Integer> minAge = list.stream().map(emp -> emp.getAge())
         .collect(Collectors.minBy(Comparator.comparing(Function.identity())));
 System.out.println("max:" + maxAge);  // max:Optional[35]
 System.out.println("min:" + minAge);  // min:Optional[20]
// ============================================================================================
// 归约操作
Optional<Integer> gy1 = list.stream().map(emp -> emp.getAge()).collect(Collectors.reducing((x, y) -> x + y));
Integer gy2 = list.stream().map(emp -> emp.getAge()).collect(Collectors.reducing(0, (x, y) -> x + y));
System.out.println(gy1);  // Optional[80]
System.out.println(gy2);  //80
// ============================================================================================
// 分操作 groupingBy 根据地址，把原list进行分组
Map<String, List<Emp>> mapGroup = list.stream().collect(Collectors.groupingBy(Emp::getName));
// partitioningBy 分区操作 需要根据类型指定判断分区
Map<Boolean, List<Integer>> partitioningMap = list.stream().map(emp -> emp.getAge())
        .collect(Collectors.partitioningBy(emp -> emp > 20));
System.out.println(mapGroup);
System.out.println(partitioningMap);
// 结果:
// {xiaoHong1=[LambdaTest$Emp@49097b5d], xiaoHong2=[LambdaTest$Emp@6e2c634b], xiaoHong4=[LambdaTest$Emp@37a71e93]}
// {false=[20], true=[35, 25]}
```

---

> 好的，结束了。流操作大概就这些了的。 ε=(´ο｀*)))唉   内容已经很多了。
>
> 以上操作，理解后，记个大概即可，反正以后也是百度或者来查笔记的。╮(╯_╰)╭ 

---

---

### **Optional** 静态类 【重点】

> 这个类的使用，主要还是在流处理上的。

说明：在java8中，很多的stream的终端操作，都返回了一个Optional<T>对象，这个对象，是用来解决空指针的问题，而产生的一个类；

> 类似 OptionalDouble、OptionalInt、OptionalLong 等，是服务于基本类型的可空对象。
>
> 使用 Optional，不仅可以避免使用 Stream 进行级联调用的空指针问题；更重要的是，它提供了一些实用的方法帮我们避免判空逻辑。

#### 静态方法

定义：

* **empty**方法，直接返回一个类加载后就创建的一个空的optional对象，
* **of**(T value)方法，可以看到，直接new了一个optional对象；
* **ofNullable**(T value)方法，可以看到，这个方法，先对传入的泛型对象，做了null的判断，为null的话，返回第一个静态方法的空对象；

```java
	private Optional() {
		this.value = null;
	}
 
	private Optional(T value) {
		this.value = Objects.requireNonNull(value);
	}
 
	public static <T> Optional<T> empty() {
		@SuppressWarnings("unchecked")
		Optional<T> t = (Optional<T>) EMPTY;
		return t;
	}
 
	public static <T> Optional<T> of(T value) {
		return new Optional<>(value);
	}
 
	public static <T> Optional<T> ofNullable(T value) {
		return value == null ? empty() : of(value);
```

注意：

```java
System.out.println(Optional.empty().get());  // 会报错！！！！
System.out.println(Optional.empty().orElseGet(()->"ddd"));
```



#### 泛型对象方法

定义：

* .**get**()直接取，<span style="color:yellow;" >**如果为null，就返回异常**  </span>
* **orElse**(T other)在取这个对象的时候，设置一个默认对象（默认值）；如果当前对象为null的时候，就返回默认对象
* **orElseGet**(Supplier<? extends T> other)<span style="color:yellow;" > **【推荐使用】**</span>跟第二个是一样的，区别只是参数，传入了一个函数式参数；
* **orElseThrow**(Supplier<? extends X> exceptionSupplier)第四个，跟上面表达的是一样的，为null的时候，返回一个特定的异常；

```java
	public T get() {
		if (value == null) {
			throw new NoSuchElementException("No value present");
		}
		return value;
	}
 
	public T orElse(T other) {
		return value != null ? value : other;
	}
 
	public T orElseGet(Supplier<? extends T> other) {
		return value != null ? value : other.get();
	}
 
	public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
		if (value != null) {
			return value;
		} else {
			throw exceptionSupplier.get();
		}
	}
```

#### 其他方法

* isPresent()是对当前的value进行null判断
* ifPresent(Consumer<? super T> consumer)具体的意思：非空消费
* filter(Predicate<? super T> predicate)，与Stream上操作一致的
* map(Function<? super T, ? extends U> mapper) ，与Stream上操作一致的
* flatMap(Function<? super T, Optional<U>> mapper) ，与Stream上操作一致的

---

---

### 时间API -（JSR310）【当做API字典使用】

#### 介绍，参考等

> Java 8引入了新的日期时间API（JSR 310）改进了日期时间的管理。日期和时间管理一直是Java开发人员最痛苦的问题。java.util.Date和后来的java.util.Calendar一点也没有改变这个情况（甚至让人们更加迷茫）。
>
> 因为上面这些原因，产生了[Joda-Time](http://www.joda.org/joda-time/) ，可以替换Java的日期时间API。[Joda-Time](http://www.joda.org/joda-time/)深刻影响了 Java 8新的日期时间API，Java 8吸收了[Joda-Time](http://www.joda.org/joda-time/) 的精华。

新的java.time包包含了所有关于日期、时间、日期时间、时区、Instant（跟日期类似但精确到纳秒）、duration（持续时间）和时钟操作的类。设计这些API的时候很认真地考虑了这些类的不变性（从java.util.Calendar吸取的痛苦教训）

**值得注意的是**：这些新增的日期时间类都是不可变类，**每次通过其方法更变或者修改都是返回一个全新的对象**，因此它们都是**线程安全**的。

参考详细文章：

* **https://cloud.tencent.com/developer/article/1649969**
* **https://cloud.tencent.com/developer/article/1649882**

**看了下，内容还是非常多的，既然整理了还是都记一下，最多部分省略咯 罒ω罒**  完整详情还是看上面这两个的原文吧！

> 如果只是使用查看的话，可以直接跳到栗子上 查看代码就可以了。

---

#### Clock ：时钟

> 就是那个记录毫秒数的那个东东   (ノ｀Д)ノ  

用于创建的静态方法：

* systemUTC ()：获取可以返回当前时刻的系统时钟，使用UTC(零)时区进行进行时间转换[SystemClock]
* systemDefaultZone()：获取可以返回当前时刻的系统时钟，使用默认时区进行时间转换[SystemClock]
* system(ZoneId zone)：获取可以返回当前时刻的系统时钟，使用指定时区ID进行时间转换[SystemClock]
* tickMillis(ZoneId zone)：获取以整数毫秒返回当前时刻的时钟，使用指定时区ID进行时间转换[TickClock]
* tickSeconds(ZoneId zone)：获取以整数秒返回当前时刻的时钟，使用指定时区ID进行时间转换[TickClock]
* tickMinutes(ZoneId zone)：获取以整数分钟返回当前时刻的时钟，使用指定时区ID进行时间转换[TickClock]
* tick(Clock baseClock, Duration tickDuration)：返回一个以基础时钟和时钟记录基础单位为构造的时钟[TickClock]
* fixed(Instant fixedInstant, ZoneId zone): 获得一个始终返回同一时刻的时钟，使用指定时区ID进行时间转换[FixedClock]
* offset(Clock baseClock, Duration offsetDuration):返回一个以基础时钟和固定时间偏移量为构造的时钟[OffsetClock]

>  方法有这么多，但是用起来的话，感觉不怎么会用到的啊  ╮(╯_╰)╭   就记一下好了

##### 栗子：

```java
 Clock clockUTC = Clock.systemUTC();
 System.out.println( clockUTC.instant() );
 System.out.println( clockUTC.millis());
 // 整数分钟返回   整数秒返回   的精度问题
 Clock tickMillis = Clock.tickMinutes(ZoneId.systemDefault());
 Clock tickSeconds = Clock.tickSeconds(ZoneId.systemDefault());
 System.out.println(tickMillis.millis());
 System.out.println(tickSeconds.millis());
 // 最常用的是默认ZoneId下的系统时钟SystemClock和UTC系统时钟：
 Clock clock = Clock.systemDefaultZone();
 System.out.println(clock.millis());
 Clock utc = Clock.systemUTC();
 System.out.println(utc.millis());
 System.out.println(System.currentTimeMillis());
// 结果
// 2020-10-29T07:45:48.622Z
// 1603957548687
// 1603957500000
// 1603957548000
// 1603957548704
// 1603957548704
// 1603957548704
```

---

#### Instant：瞬时时间

说明：是`java.sql.Timestamp`的对应类，代表时间线(Time-Line)上的一个瞬时时间点。

> 内部持有一个`long`类型的纪元秒属性(seconds)和一个`int`类型的纳秒属性(nanos，nanos的取值范围是[0,999_999_999])，纪元秒如果为正数，表示该瞬时时间点位于格林威治新纪元`1970-01-01T00:00:00Z`之后。

##### 常用方法：

【由于Instant 没有公共构造器，必须通过工厂方法构造的】

```java
// ==============  静态方法，工厂构建使用  ======================
// 当前时刻的瞬时时间点
public static Instant now()
// 基于时钟实例获取瞬时时间点
public static Instant now(Clock clock)
// 基于距离新纪元的秒数创建瞬时时间点
public static Instant ofEpochSecond(long epochSecond)
// 基于距离新纪元的秒数和纳秒创建瞬时时间点
public static Instant ofEpochSecond(long epochSecond, long nanoAdjustment)
// 基于毫秒数创建瞬时时间点
public static Instant ofEpochMilli(long epochMilli)
// 基于其他日期时间API创建瞬时时间点
public static Instant from(TemporalAccessor temporal)
// 基于特定格式字符串创建瞬时时间点，如2007-12-03T10:15:30.00Z
public static Instant parse(final CharSequence text)
// ==============  实例方法  ======================
// 获取当前Instant实例对于不同计时单位的值，见ChronoField
public long getLong(TemporalField field)
// 获取当前Instant实例的纪元秒属性
public long getEpochSecond()
// 获取当前Instant实例的纳秒属性
public int getNano()
// 获取当前Instant实例的毫秒值
public long toEpochMilli() 
// 基于TemporalField实例(TemporalField)和新的值调整并且创建一个新的Instant
public Instant with(TemporalField field, long newValue)
// 当前Instant实例基于TemporalUnit(ChronoUnit)截断并且返回一个新的Instant
public Instant truncatedTo(TemporalUnit unit)
// 顾名思义，基于一个时间基准单位进行时间量增加，返回一个新的Instant
public Instant plus(long amountToAdd, TemporalUnit unit)
// 顾名思义，基于一个时间基准单位进行时间量减少，返回一个新的Instant
public Instant minus(long amountToSubtract, TemporalUnit unit)
// 计算当前Instant实例和入参endExclusive基于时间基准单位unit之间的时间量
public long until(Temporal endExclusive, TemporalUnit unit)
```

##### 栗子

```java
// 使用各种创建的Instant 的静态工厂方法进行创建  并调用实例方法输出
Instant instant = Instant.now();
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
instant = Instant.now(Clock.systemDefaultZone());
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
instant = Instant.ofEpochSecond(new Date().toInstant().getEpochSecond());
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
instant = Instant.ofEpochMilli(System.currentTimeMillis());
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
instant = Instant.from(Instant.now());
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
instant = Instant.parse("2018-12-31T10:15:30.00Z");
System.out.println(String.format("Second:%d,Nano:%d", instant.getEpochSecond(), instant.getNano()));
// Second:1603958527,Nano:232000000
// Second:1603958527,Nano:313000000
// Second:1603958527,Nano:0
// Second:1603958527,Nano:313000000
// Second:1603958527,Nano:313000000
// Second:1546251330,Nano:0
```

---

#### LocalDate：无时区的日期【重点-常用】

##### 说明：

代表ISO-8601日历系统中**不包含时区的日期(当然也不包含具体的时间)表示**，例如2007-12-03。

> 就是，只能表示日期的东东。 

提供，日期的一些操作：例如一年中的第几日(day-of-year)、星期几(day-of-week)和一年中的第几周(week-of-year)等

> 注意：`LocalDate`之间的比较只能通过`LocalDate#equals()`方法，其他比较操作如`==`或者`hash()`方法会产生无法预知的结果

##### 常用方法等

```java
// ================= 工厂方法，部分【由于太多了】====================
// 基于当前日期获取LocalDate实例
public static LocalDate now()
// 基于当前日期和时区获取LocalDate实例
public static LocalDate now(ZoneId zone)
// 基于当前日期和时钟获取LocalDate实例
public static LocalDate now(Clock clock)
// 基于年月(枚举)日获取LocalDate实例
public static LocalDate of(int year, Month month, int dayOfMonth)
// 基于年月日获取LocalDate实例
public static LocalDate of(int year, int month, int dayOfMonth)
// 基于年和具体该年中的某一日获取LocalDate实例
public static LocalDate ofYearDay(int year, int dayOfYear)
// 基于新纪元1970-01-01的偏移天数获取LocalDate实例
public static LocalDate ofEpochDay(long epochDay)
// ============= 实例方法，部分【由于太多了】【感觉灰常好用的样子 罒ω罒】====
// 获取年份值
public int getYear()
// 获取月份值，范围是1-12
public int getMonthValue()
// 获取月份枚举
public Month getMonth()
// 返回当前LocalDate实例的该年中具体的第几天
public int getDayOfYear()
// 返回当前LocalDate实例的具体是星期几
public DayOfWeek getDayOfWeek()
// 是否闰年
public boolean isLeapYear()
// 返回当前LocalDate实例月份长度
public int lengthOfMonth()
// 返回当前LocalDate实例年份长度
public int lengthOfYear()
// ===================== 操作 与 推算 等 =======================
// 基于一个日期属性修改对应的值返回一个新的LocalDate实例
public LocalDate with(TemporalField field, long newValue)
// 基于一个日期时间基准单位增加对应的值返回一个新的LocalDate实例
public LocalDate plus(long amountToAdd, TemporalUnit unit)
// 基于一个日期时间基准单位减去对应的值返回一个新的LocalDate实例
public LocalDate minus(long amountToSubtract, TemporalUnit unit)
// 基于一个日期时间基准单位计算以入参为endExclusive计算日期或者时间的间隔
public long until(Temporal endExclusive, TemporalUnit unit)
// 返回基于新纪元年1970-1-1的偏移天数
public long toEpochDay()
// ====================↓↓↓↓↓↓↓↓↓ 对比必须使用的方法 ↓↓↓↓↓↓↓================
// 如果入参为LocalDate类型功能和equals一致，否则通过基于纪元年的偏移天数比较
public boolean isEqual(ChronoLocalDate other)
// 只有年月日三个成员同时相等此方法才返回true
public boolean equals(Object obj)
```

##### 栗子：

```java
LocalDate localDate = LocalDate.now();
System.out.println(localDate); // 2020-10-29
localDate = LocalDate.of(2018, 12, 31);
System.out.println(localDate); // 2018-12-31
localDate = localDate.plus(1, ChronoUnit.DAYS);
System.out.println(localDate); // 2019-01-01
localDate = localDate.with(ChronoField.YEAR,2020);
System.out.println(localDate); // 2020-01-01
System.out.println(localDate.equals(LocalDate.of(2019,1,1))); // false
System.out.println(localDate.toEpochDay()); //18262
```

---

#### LocalTime：无时区的时间

##### 说明

代表ISO-8601日历系统中**不包含时区的时间(当然也不包含具体的日期)表示**，例如10:15:30；

> 跟上面的那个日期很类似的，只是这个值表示时间。**精度可以到纳秒的**。

##### 常用方法

```java
// ====================== 静态实例 =========================
// 一天的起始时间 - 00:00
public static final LocalTime MIN
// 一天的结束时间 - 23:59:59.999999999
public static final LocalTime MAX
// 午夜 - 00:00 其实和MIN是一样的
public static final LocalTime MIDNIGHT
// 中午 - 12:00
public static final LocalTime NOON
// ================= 工厂方法，部分【由于太多了】====================
// 基于当前时间构造LocalTime实例
public static LocalTime now()
// 基于当前时间和时区ID构造LocalTime实例
public static LocalTime now(ZoneId zone)
// 基于当前时间和时钟实例构造LocalTime实例
public static LocalTime now(Clock clock)
// 基于小时、分钟(、秒和纳秒)构造LocalTime实例
public static LocalTime of(int hour, int minute)
public static LocalTime of(int hour, int minute, int second)
public static LocalTime of(int hour, int minute, int second, int nanoOfSecond)
// 基于瞬时时间实例和时区ID构造LocalTime实例
public static LocalTime ofInstant(Instant instant, ZoneId zone)
// 基于一天当中的具体秒数构造LocalTime实例,secondOfDay范围是[0,24 * 60 * 60 - 1]
public static LocalTime ofSecondOfDay(long secondOfDay)
// 基于一天当中的具体纳秒数构造LocalTime实例,nanoOfDay[0,24 * 60 * 60 * 1,000,000,000 - 1]
public static LocalTime ofNanoOfDay(long nanoOfDay)  
```

##### 栗子

```java
        LocalTime localTime = LocalTime.now();
        System.out.println(localTime);
        localTime = LocalTime.of(23, 59);
        System.out.println(localTime);
        localTime = LocalTime.MAX;
        System.out.println(String.format("Hour:%d,minute:%d,second:%d,nano:%d", localTime.getHour(),
                localTime.getMinute(), localTime.getSecond(), localTime.getNano()));
// 结果：
// 16:06:54.073     // 感觉这个.*** 纳秒的进度不想要，碍事的感觉啊
// 23:59
// Hour:23,minute:59,second:59,nano:999999999
```

---

#### LocalDateTime：上面↑两个合并 【重点】

> 感觉上这个类，的使用场景会比较多，在要求不高的情况下会比较常使用的。

上面两个`LocalDate`和`LocalTime`的结合版本；内容差不过的了。

##### 常用方法

```java
// 基于当前日期时间、时区ID、时钟创建LocalDateTime实例
public static LocalDateTime now()
public static LocalDateTime now(ZoneId zone)
public static LocalDateTime now(Clock clock)
// 基于年月(枚举)日时分秒纳秒创建LocalDateTime实例
public static LocalDateTime of(int year, Month month, int dayOfMonth, int hour, int minute)
    .....这个of（。。）有各种类型的  
// 基于LocalDate和LocalTime实例创建LocalDateTime实例
public static LocalDateTime of(LocalDate date, LocalTime time)
// 基于Instant实例和时区ID实例创建LocalDateTime实例
public static LocalDateTime ofInstant(Instant instant, ZoneId zone)
// 基于新纪元偏移秒数、纳秒数和时间偏移量创建LocalDateTime实例
public static LocalDateTime ofEpochSecond(long epochSecond, int nanoOfSecond, ZoneOffset offset)
```

##### 栗子

```java
LocalDateTime localDateTime = LocalDateTime.now(ZoneId.systemDefault());
System.out.println(localDateTime);
localDateTime = LocalDateTime.ofInstant(Instant.now(), ZoneId.systemDefault());
System.out.println(localDateTime);
localDateTime = localDateTime.plus(1, ChronoUnit.YEARS);
System.out.println(localDateTime);
// 2020-10-30T16:13:23.513
// 2020-10-30T16:13:23.514
// 2021-10-30T16:13:23.514
```

---

#### OffsetTime：UTC偏移量时间

##### 说明：

ISO-8601日历系统中带有基于UTC/Greenwich时间偏移量的时间，例如10:15:30+01:00；

相比`LocalTime`，它多存储了一个时区时间偏移量(zone offset)属性。

##### 常用方法

```java
// 代表00:00:00+18:00
public static final OffsetTime MIN = LocalTime.MIN.atOffset(ZoneOffset.MAX)
// 代表23:59:59.999999999-18:00
public static final OffsetTime MAX = LocalTime.MAX.atOffset(ZoneOffset.MIN)
// ================= 工厂方法====================
// 基于当前时间、时区ID、时钟创建OffsetTime实例
public static OffsetTime now()
public static OffsetTime now(ZoneId zone)
public static OffsetTime now(Clock clock)
// 基于LocalTime实例和时间偏移量创建OffsetTime实例
public static OffsetTime of(LocalTime time, ZoneOffset offset)
// 基于时分秒纳秒和时间偏移量创建OffsetTime实例
public static OffsetTime of(int hour, int minute, int second, int nanoOfSecond, ZoneOffset offset)
// 基于Instant实例和时区ID实例创建OffsetTime实例
public static OffsetTime ofInstant(Instant instant, ZoneId zone)
```

##### 栗子

```java
System.out.println(OffsetTime.now());
System.out.println(OffsetTime.ofInstant(Instant.now(), ZoneId.systemDefault()));
System.out.println(OffsetTime.of(LocalTime.now(), ZoneOffset.UTC));
// 17:02:21.295+08:00
// 17:02:21.296+08:00
// 17:02:21.296Z
```

---

#### OffsetDateTime：UTC偏移量日期

##### 说明

表示ISO-8601日历系统中带有基于UTC/Greenwich时间偏移量的日期时间，例如2007-12-03T10:15:30+01:00。

> 整理到这里，感觉都差不多了，这些方法

##### 常用方法与栗子

```java
// ================= 常用的方法 =======================
// 基于当前的日期时间、时区ID、时钟创建OffsetDateTime实例
public static OffsetDateTime now()
public static OffsetDateTime now(ZoneId zone)
public static OffsetDateTime now(Clock clock)
// 基于LocalDate实例、LocalTime实例和时间偏移量创建OffsetDateTime实例
public static OffsetDateTime of(LocalDate date, LocalTime time, ZoneOffset offset)
// 基于LocalDateTime实例和时间偏移量创建OffsetDateTime实例
public static OffsetDateTime of(LocalDateTime dateTime, ZoneOffset offset)

// 基于年月日时分秒纳秒和时间偏移量创建OffsetDateTime实例
public static OffsetDateTime of(
            int year, int month, int dayOfMonth,
            int hour, int minute, int second, int nanoOfSecond, ZoneOffset offset)
// 基于Instant实例和时区ID实例创建OffsetDateTime实例
public static OffsetDateTime ofInstant(Instant instant, ZoneId zone)
// ================= 简单栗子 =======================
System.out.println(OffsetDateTime.now());
System.out.println(OffsetDateTime.ofInstant(Instant.now(), ZoneId.systemDefault()));
System.out.println(OffsetDateTime.of(LocalDateTime.now(), ZoneOffset.ofHours(8)));
// 2020-10-30T17:18:25.517+08:00
// 2020-10-30T17:18:25.518+08:00
// 2020-10-30T17:18:25.518+08:00
```

---

#### ZonedDateTime：日期+偏移时间+时区

> 说是重点，其实也还好了。只能说最具体的时间吧。所有时间的东东它都有了。

##### 说明：

可以简单理解为`LocalDateTime`，时区ID和一个可处理的`ZoneOffset`三者的共同实现，或者更简单理解为日期时间、时间偏移量、区域时区等时区规则的多重实现。

常用的格式为：年-月-日 时:分:秒-时区偏移量-区域，例如2007-12-03T10:15:30+01:00 Europe/Paris。

##### 常用方法

```java
// ================= 工厂方法====================
// 根据当前的日期时间、时区ID和时钟创建ZonedDateTime实例
public static ZonedDateTime now()
public static ZonedDateTime now(ZoneId zone)
public static ZonedDateTime now(Clock clock)

// 基于LocalDate实例、LocalTime实例和时区ID创建ZonedDateTime实例
public static ZonedDateTime of(LocalDate date, LocalTime time, ZoneId zone)
// 基于LocalDateTime实例和时区ID创建ZonedDateTime实例
public static ZonedDateTime of(LocalDateTime localDateTime, ZoneId zone)
// 基于年月日时分秒纳秒和时区ID创建ZonedDateTime实例
public static ZonedDateTime of(
            int year, int month, int dayOfMonth,
            int hour, int minute, int second, int nanoOfSecond, ZoneId zone)
// 基于LocalDateTime实例、时区ID和候选偏好的时间偏移量创建ZonedDateTime实例
public static ZonedDateTime ofLocal(LocalDateTime localDateTime, ZoneId zone, ZoneOffset preferredOffset)	
```

##### 栗子

```java
System.out.println(ZonedDateTime.now());
System.out.println(ZonedDateTime.of(LocalDateTime.now(), ZoneId.systemDefault()));
System.out.println(ZonedDateTime.ofInstant(Instant.now(), ZoneId.systemDefault()));
// 2020-11-03T10:33:42.239+08:00[Asia/Shanghai]
// 2020-11-03T10:33:42.240+08:00[Asia/Shanghai]
// 2020-11-03T10:33:42.240+08:00[Asia/Shanghai]
```

---

#### 其他-年月日等

* Year：有判断闰年等的方法
* Month：内部有12个月份枚举成员，方便使用。
* DayOfWeek：7个枚举类型表示星期几
* MonthDay：代表月份和对应月份一共存在的天数，内部维护着整型的属性month和整型的属性day
  * 栗子：`System.out.println(MonthDay.now());  // --11-03`;
* YearMonth：代表年份和月份，内部维护着整型的属性month和整型的属性month
  * 栗子：`System.out.println(YearMonth.now());  // 2020-11`;

---

#### 时间类转换

> 各个时间类的工厂方法中的`ofInstant()`，就可以用来构建转换实例的。

##### 日期时间类之间互相转换

**日期时间-> 日期或时间**：本身有保存时间类作为成员属性的，只用就可以取到。

```java
LocalDateTime localDateTime = LocalDateTime.now();
LocalDate localDate = localDateTime.toLocalDate();
LocalTime localTime = localDateTime.toLocalTime();
```

**日期->日期时间**：由于部分不包含的属性，时间部分会取最小值。

```java
LocalDate localDate = LocalDate.now();
System.out.println(localDate);
System.out.println(localDate.atStartOfDay());
System.out.println(localDate.atStartOfDay(ZoneId.systemDefault()));
// 2020-11-03
// 2020-11-03T00:00
// 2020-11-03T00:00+08:00[Asia/Shanghai]
```

时区与非时区：可以轻易转换，相反就添加对应的时区ID属性；

##### 新旧日期时间相关类之间的转换

`java.sql.Timestamp`和`java.time.LocalDateTime`之间的转换：

```java
LocalDateTime localDateTime = LocalDateTime.now();
Timestamp timestamp = Timestamp.valueOf(localDateTime);
LocalDateTime ldt = timestamp.toLocalDateTime();
```

`java.sql.Date`和`java.time.LocalDate`之间的转换：

```java
Date date = new Date(2018, 1, 1);
LocalDate localDate = date.toLocalDate();
date = new Date(localDate.getYear(), localDate.getMonthValue(), localDate.getDayOfMonth());
```

只要是能使用毫秒表示的旧的日期时间类，都可以和`java.time.Instant`相互转换，例如：

```java
Timestamp timestamp = new Timestamp(System.currentTimeMillis());
Instant instant = timestamp.toInstant();
java.util.Date date = new Date(System.currentTimeMillis());
instant = date.toInstant();
timestamp = new Timestamp(instant.toEpochMilli());
date = new Date(instant.toEpochMilli());
```



#### 日期时间API之间的关系

![img](https://ask.qcloudimg.com/http-save/7454122/st88hww50r.jpg?imageView2/2/w/1620)

![日期时间api关系](.\日期时间api关系.png)

> 从这个的图片上，可以看出来各个新的时间类的大小属性关系吧。
>
> 至于所谓的实际使用，还是看参考文章里面的第二篇吧，很详细的。

---

---

### 并行处理-流/数组？

* 流处理的并行：使用的是parallelStream，
  
* 这个方法，在上文已经使用了的。与Stream类型，问题就是要考虑到并行的情况。
  
* 并行数组：使用的是**parallelSort**这个方法。

  * > 只能说java8 的对这个排序方法优化了，代码编写层面没差啦。

  * 栗子：

    ```java
    long[] arrayOfLong = new long [ 20000 ];
    // parallelSetAll 提供了计算每个元素的生成器函数。
    Arrays.parallelSetAll( arrayOfLong,
            index -> ThreadLocalRandom.current().nextInt( 1000000 ) );
    Arrays.stream( arrayOfLong ).limit( 10 ).forEach(
            i -> System.out.print( i + " " ) );
    System.out.println();
    //`parallelSort 将指定数组按数字升序排序。
    Arrays.parallelSort( arrayOfLong );
    Arrays.stream( arrayOfLong ).limit( 10 ).forEach(
            i -> System.out.print( i + " " ) );
    System.out.println();
    ```

好像就这些了。╮(╯_╰)╭

---

### 并发相关-5种

#### 1-CountDownLatch - 阻塞主线程

```java
int taskCount = 100;   // 任务数 任务为+1
int threadCount = 5;   // 线程数
//总操作次数计数器
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.set(1);
//使用CountDownLatch来等待所有线程执行完成
CountDownLatch countDownLatch = new CountDownLatch(threadCount);
//使用IntStream把数字直接转为Thread
IntStream.rangeClosed(1, threadCount).mapToObj(i -> new Thread(() -> {
    //手动把taskCount分成taskCount份，每一份有一个线程执行
    IntStream.rangeClosed(1, taskCount / threadCount).forEach(j -> {atomicInteger.addAndGet(1);});
    // atomicInteger.addAndGet(1); 如果这个代替上面这一句更好理解，就是分5个线程数操作。
    //每一个线程处理完成自己那部分数据之后，countDown一次
    countDownLatch.countDown();
})).forEach(Thread::start);
//等到所有线程执行完成
countDownLatch.await();
//查询计数器当前值
System.out.println(atomicInteger.get());   // 101
```

#### 2-Executors.newFixedThreadPool  - 线程池

```java
int taskCount = 100;   // 任务数 任务为+1
int threadCount = 5;   // 线程数
//总操作次数计数器
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.set(1);
//初始化一个线程数量=threadCount的线程池
ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
//所有任务直接提交到线程池处理
IntStream.rangeClosed(1, taskCount).forEach(i -> executorService.execute(() -> {atomicInteger.addAndGet(1);}));
//提交关闭线程池申请，等待之前所有任务执行完成
executorService.shutdown();
executorService.awaitTermination(1, TimeUnit.HOURS);
//查询计数器当前值
System.out.println(atomicInteger.get());   // 101
```

#### 3-ForkJoinPool - 线程池

> ForkJoinPool 和传统的 ThreadPoolExecutor 区别在于，前者对于 n 并行度有 n 个独立队列，后者是共享队列。如果有大量执行耗时比较短的任务，ThreadPoolExecutor 的单队列就可能会成为瓶颈。这时，使用 ForkJoinPool 性能会更好。
>
> 因此，ForkJoinPool 更适合大任务分割成许多小任务并行执行的场景，而 ThreadPoolExecutor 适合许多独立任务并发执行的场景。

```java
int taskCount = 100;   // 任务数 任务为+1
int threadCount = 5;   // 线程数
//总操作次数计数器
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.set(1);
//自定义一个并行度=threadCount的ForkJoinPool
ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount);
//所有任务直接提交到线程池处理  parallel:并行处理
forkJoinPool.execute(() -> IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> {atomicInteger.addAndGet(1);}));
// 上面这种写法还是比较的优雅的，下面这种写法感觉也是可以的，但是没有体现这种东东的优势
// IntStream.rangeClosed(1, taskCount).forEach(i -> forkJoinPool.execute(() -> {atomicInteger.addAndGet(1);}));
//提交关闭线程池申请，等待之前所有任务执行完成
forkJoinPool.shutdown();
forkJoinPool.awaitTermination(1, TimeUnit.HOURS);
//查询计数器当前值
System.out.println(atomicInteger.get());   // 101
```

#### 4-ForkJoinPool.commonPool()-并行流

> 直接使用并行流，并行流使用公共的 ForkJoinPool，也就是 ForkJoinPool.commonPool()。
> 公共的 ForkJoinPool 默认的并行度是 CPU 核心数 -1，原因是对于 CPU 绑定的任务分配超过 CPU 个数的线程没有意义。由于并行流还会使用主线程执行任务，也会占用一个 CPU 核心，所以公共 ForkJoinPool 的并行度即使 -1 也能用满所有 CPU 核心。

这个的用法感觉就比较麻烦了。毕竟这个并行度是公共的啊。

```java
int taskCount = 100;   // 任务数 任务为+1
int threadCount = 5;   // 线程数
//总操作次数计数器
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.set(1);
//设置公共ForkJoinPool的并行度
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", String.valueOf(threadCount));
//由于我们设置了公共ForkJoinPool的并行度，直接使用parallel提交任务即可
IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> {atomicInteger.addAndGet(1);});
//查询计数器当前值
System.out.println(atomicInteger.get());   // 101
```

> 另外需要注意的是，在上面的例子中我们一定是先运行 stream 方法再运行 forkjoin 方法，对公共 ForkJoinPool 默认并行度的修改才能生效。
>
> ForkJoinPool 类初始化公共线程池是在静态代码块里，加载类时就会进行的，如果 forkjoin 方法中先使用了 ForkJoinPool，即便 stream 方法中设置了系统属性也不会起作用。因此我的建议是，设置 ForkJoinPool 公共线程池默认并行度的操作，应该放在应用启动时设置。
>
> 这个的说法我并没有看懂，╮(╯_╰)╭   ！！！

#### 5-CompletableFuture 【组合式异步编程】【灰常厉害的样子】

CompletableFuture.runAsync 方法可以指定一个线程池，一般会在使用 CompletableFuture 的时候用到.

```java
int taskCount = 100;   // 任务数 任务为+1
int threadCount = 5;   // 线程数
//总操作次数计数器
AtomicInteger atomicInteger = new AtomicInteger();
atomicInteger.set(1);
//自定义一个并行度=threadCount的ForkJoinPool
ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount);
//使用CompletableFuture.runAsync通过指定线程池异步执行任务
CompletableFuture.runAsync(() -> IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> {atomicInteger.addAndGet(1);}), forkJoinPool).get();
//查询计数器当前值
System.out.println(atomicInteger.get());   // 101
// 可以使用System.out.println(Thread.currentThread().getName()); 打印出来看看线程情况
```

> 其他的东西，关于其使用方法。**有点像流的编程方式。**
>
> ```java
> // 任务 A 无返回值，所以对应的，第 2 行和第 3 行代码中，resultA 其实是 null。
> CompletableFuture.runAsync(() -> {}).thenRun(() -> {});
> CompletableFuture.runAsync(() -> {}).thenAccept(resultA -> {});
> CompletableFuture.runAsync(() -> {}).thenApply(resultA -> "resultB");
> 
> //  thenRun(Runnable runnable)，任务 A 执行完执行 B，并且 B 不需要 A 的结果
> CompletableFuture.supplyAsync(() -> "resultA").thenRun(() -> {});
> // thenAccept(Consumer action)，任务 A 执行完执行 B，B 需要 A 的结果，但是任务 B 不返回值。
> CompletableFuture.supplyAsync(() -> "resultA").thenAccept(resultA -> {});
> //  thenApply(Function fn)，任务 A 执行完执行 B，B 需要 A 的结果，同时任务 B 有返回值。
> CompletableFuture.supplyAsync(() -> "resultA").thenApply(resultA -> resultA + " resultB");
> ```

具体这个CompletableFuture 的内容，参考一下详细文章：

* https://blog.csdn.net/w605283073/article/details/92418504
* https://www.cnblogs.com/dreamroute/p/13706784.html
* https://www.jianshu.com/p/b3c4dd85901e

---

### 小结

关于java8的新特性，当然不只是以上这些内容了。这些还只是我据地重要的。

原本只打算整理个流相关的，结果越了解，才发现要学的东西越来越多了。

以上，欢迎━(*｀∀´*)ノ亻!  点赞关注收藏     

小杭 - 2020-11 ∠(°ゝ°) 

---



