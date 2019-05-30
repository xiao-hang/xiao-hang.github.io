# MySQL-JSON数据说明和操作

[TOC]

---

## 说明

公司数据库使用版本为：MySQL 5.7.18

官方说明，在MySQL5.7.8 的时候开始支持JSON数据类型，支持的最大值为1G（max_allowed_packet ->1073741824）。

说明就这些了，然后开始介绍JSON的数据操作，只选择常用或者我觉得方便使用的，其他贼难用的就简单略过了。

### 参考文档

* <https://dev.mysql.com/doc/refman/5.7/en/json-function-reference.html>
* <https://dev.mysql.com/doc/refman/5.7/en/json.html>

### 栗子中的数据

```sql
默认数据都是这个： data_json字段数据类型为json （测试数据条件：from census_summary  where id = 1111）
data_json字段的值： {"data": {"1天MPOS": [0, 0, 0], "2天MPOS": [0, 0, 0], "8天MPOS": [0, 0, 0], "10天MPOS": [0, 0, 0], "1天传统POS": [1000, 10, 1], "3天传统POS": [0, 0, 0], "11天传统POS": [0, 0, 0], "30天传统POS": [3000, 30, 1], "360天传统POS": [6000, 60, 5]}, "column": ["属性", "总申领金额", "总申领台数", "总申领人数"], "总数据": [10000, 100, "7"]}
```



## JSON数据创建

查看了文档，创建的方法有辣么几个：

* JSON_ARRAY(val,val,val.........) ：创建数组JSON对象
* JSON_OBJECT(key, val [key, val].......) : 创建键值对JSON对象
* JSON_QUOTE(string) ：转换字符串为纯粹的字符串，转义掉各种符号为`【\ + 符号】`的形式

这里使用这些方法来构建JSON数据的话，可以保证JSON数据是正确的，错误的话执行不了(ノ｀Д)ノ

但是，方法的使用很麻烦，**如果是复杂的JSON需要嵌套使用**。

给个**例子**，说明一下使用情况：

```sql
SET @j = JSON_ARRAY("a1","a2")
set @k = JSON_OBJECT("k1","V1")
SELECT JSON_ARRAY("a0",JSON_ARRAY("A2","A3"),@j,JSON_OBJECT("k1","V1"),@k)
-- 结果：["a0", ["A2", "A3"], "[\"a1\", \"a2\"]", {"k1": "V1"}, "{\"k1\": \"V1\"}"]
select @j  -- 结果：["a1", "a2"]
select @k  -- 结果：{"k1": "V1"}
-- ↓↓↓↓↓↓↓↓↓↓建议之间使用字符串拼接，简单方便多了：↓↓↓↓↓↓↓↓↓↓
select '["a0", ["A2", "A3"], "[\"a1\", \"a2\"]", {"k1": "V1"}, "{\"k1\": \"V1\"}"]'
```

这里可以看到，构建的方法必须按照标准的格式来，嵌套使用的情况会比较麻烦。所以才**建议复杂的之间字符串拼接保存到对应的JSON字段里**面去就可以了。╮(╯_╰)╭

## JSON的常用搜索

搜索的方法很多，就忽略一些搜索判断的了，直接了解常用的取数据的方法。

### 匹配查询

**JSON_EXTRACT(json_doc, path[, path] ...)**

> **返回JSON文档中的path匹配的数据**，json_doc参数和path路径不是有效的会直接报错

**用法**就直接看**例子**好了：

```sql
select JSON_EXTRACT(data_json,'$.data."360天传统POS"[0]') from census_summary  where id = 1111
-- 6000
JSON_EXTRACT(data_json,'$.data."360天传统POS"') from census_summary  where id = 1111
-- [6000, 60, 5]
-- path中，‘$’表示第一个参数数据本身，之后：‘.’用来表示键值对查询，‘【】’表示数组查询
```

### 匹配查询简写（常用）

**->:column->path**

这个符号就是，**上面JSON_EXTRACT方法的简写**，在MySQL 5.7.9及更高版本的时候支持。

就像这样：

```sql
select data_json -> '$.data."360天传统POS"' from census_summary  where id = 1111
-- [6000, 60, 5]   是一样的效果的
```

> 这里的效果是一样的。

### 匹配查询简写

**->> : column->>path**

在MySQL 5.7.13及更高版本中使用，这个也是搜索取值，表示的意思是对正常取值后进行转义（数据里面的反斜杠数据会被转义）；

与以下两种的写法是一样的效果:

* JSON_UNQUOTE( JSON_EXTRACT(column, path) )

* JSON_UNQUOTE(column -> path)

>JSON_UNQUOTE 这个方法是：取消引用JSON值，就是转义：【取消引用JSON值并将结果作为`utf8mb4`字符串返回】

这个就暂时没有例子，直接看官方的：

```sql
mysql> CREATE TABLE tj10 (a JSON, b INT);
Query OK, 0 rows affected (0.26 sec)

mysql> INSERT INTO tj10 VALUES
    ->     ('[3,10,5,"x",44]', 33),
    ->     ('[3,10,5,17,[22,"y",66]]', 0);
Query OK, 2 rows affected (0.04 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> SELECT a->"$[3]", a->"$[4][1]" FROM tj10;
+-----------+--------------+
| a->"$[3]" | a->"$[4][1]" |
+-----------+--------------+
| "x"       | NULL         |
| 17        | "y"          |
+-----------+--------------+
2 rows in set (0.00 sec)

mysql> SELECT a->>"$[3]", a->>"$[4][1]" FROM tj10;
+------------+---------------+
| a->>"$[3]" | a->>"$[4][1]" |
+------------+---------------+
| x          | NULL          |
| 17         | y             |
+------------+---------------+
2 rows in set (0.00 sec)
```

可以看到这个效果，就是对`["]`这个双引号的处理，已有如果有碰到转义的问题可以试试这个东东。╮(╯_╰)╭

### 查询所有KEY

**JSON_KEYS(json_doc[, path])**

**返回JSON对象的顶级值返回键作为JSON数组，或者，如果*path* 给出参数，则返回所选路径中的顶级键。**

上例子：

```sql
select  JSON_KEYS(data_json) from census_summary  where id = 1111
-- ["data", "column", "总数据"]
select  JSON_KEYS(data_json,'$.data') from census_summary  where id = 1111
-- ["1天MPOS", "2天MPOS", "8天MPOS", "10天MPOS", "1天传统POS", "3天传统POS", "11天传统POS", "30天传统POS", "360天传统POS"]
select  JSON_KEYS(data_json,'$.data."360天传统POS"') from census_summary  where id = 1111
-- null
```

说明一下，就是返回对应键值对JSON数据的所有KEY的列表。

> 如果是数组的话 ，则只是返回(Null) 了。

### 查询指定数据路径

**JSON_SEARCH(json_doc, one_or_all, search_str)**

返回JSON文档中给定**字符串**的路径。就是查询特定值的具体JSON路径。

> **这里尤其要注意一下，这个查询只能查询字符串的值，数字是不行的，并且KEY的值也是不行的。**

其中，参数one_or_all的定义：

* `'one'`：搜索在第一个匹配后终止，并返回一个路径字符串。未定义哪个匹配首先考虑。
* `'all'`：搜索返回所有匹配的路径字符串，以便不包含重复的路径。如果有多个字符串，则将它们自动包装为数组。数组元素的顺序未定义。

**参数 search_str 的特殊定义：%和_ 字符用于LIKE 运算符：%匹配任意数量的字符（包括零个字符），并且只 _匹配一个字符。**

高级的多层过滤使用我不会，请自行查询，看下简单的**栗子**：

```sql
select JSON_SEARCH(data_json,'all','总申领金额') from census_summary  where id = 1111
-- "$.column[1]"
select JSON_SEARCH(data_json,'all','data') from census_summary  where id = 1111
-- null
select data_json,JSON_SEARCH(data_json,'one',100) from census_summary  where id = 1111
-- null
select data_json,JSON_SEARCH(data_json,'one',7) from census_summary  where id = 1111
-- "$.\"总数据\"[2]"
```

这里可以看出来的几点：

* 只能查字符串的值
* 不能查询KEY的值
* 返回的字符串路径中是带反斜杠的

> 高级使用，是不止3个参数的：JSON_SEARCH(json_doc, one_or_all, search_str[, escape_char[, path] ...])
>
> 是可以很多的，但是我不会，只知道大概是还可以继续过滤条件的样子

### 重点搜索需要注意的东东

一些使用上的功能说明和注意：

* JSON值是可以进行比较的；（是直接Json比较，比较规则需要的话查询官方文档）

* JSON与非JSON值之间的转换，会存在字符转义的问题：

  > | 其他类型                                   | CAST（其他类型AS JSON）                                      | CAST（JSON AS其他类型）                                      |
  > | :----------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
  > | JSON                                       | 没变                                                         | 没变                                                         |
  > | UTF8字符类型（`utf8mb4`，`utf8`，`ascii`） | 该字符串被解析为JSON值。                                     | JSON值被序列化为`utf8mb4`字符串。                            |
  > | 其他字符类型                               | 其他字符编码被隐式转换 `utf8mb4`为utf8字符类型所描述和处理。 | JSON值被序列化为`utf8mb4`字符串，然后转换为其他字符编码。结果可能没有意义。 |
  > | `NULL`                                     | 结果`NULL`为JSON类型的值。                                   | 不适用。                                                     |
  > | 几何类型                                   | 几何值通过调用转换为JSON文档 [`ST_AsGeoJSON()`](https://dev.mysql.com/doc/refman/5.7/en/spatial-geojson-functions.html#function_st-asgeojson)。 | 非法操作。解决方法：将结果传递 给 。[`CAST(*json_val* AS CHAR)`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast)[`ST_GeomFromGeoJSON()`](https://dev.mysql.com/doc/refman/5.7/en/spatial-geojson-functions.html#function_st-geomfromgeojson) |
  > | 所有其他类型                               | 结果是由单个标量值组成的JSON文档。                           | 如果JSON文档由目标类型的单个标量值组成，并且标量值可以强制转换为目标类型，则成功。否则，返回`NULL` 并发出警告。 |

* **SQL 语句中可以对JSON直接进行的操作：**

  * 排序，`ORDER BY CAST(JSON_EXTRACT(jdoc, '$.id') AS UNSIGNED)`
  * 可以再where 子句中直接判断：`WHERE JSON_EXTRACT(c, "$.id") > 1`
  * 聚合函数（支持的）
    * 非`NULL`值被转换为数字类型和聚合，除 [`MIN()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_min)， [`MAX()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_max)和 [`GROUP_CONCAT()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_group-concat)。

---

## JSON 修改的操作 - 其实不常用

这个操作其实不一定是常用的操作，对于JSON的数据，通常只是把数据保存和读取，较少会进行修改的。

**这里列举一下一些的操作方法：**

### 追加指定数组末尾

**JSON_ARRAY_APPEND(json_doc, path, val[, path, val] ...)**

> 只能是数组的，没什么用的感觉，可以用到的吗╮(╯_╰)╭

```mysql
mysql> SET @j = '["a", ["b", "c"], "d"]';
mysql> SELECT JSON_ARRAY_APPEND(@j, '$[1]', 1);
+----------------------------------+
| JSON_ARRAY_APPEND(@j, '$[1]', 1) |
+----------------------------------+
| ["a", ["b", "c", 1], "d"]        |
```

### 数组指定插入值

**JSON_ARRAY_INSERT(json_doc, path, val[, path, val] ...)**

介绍：略过了。。。。。

### 数据插入

**JSON_INSERT(json_doc, path, val[, path, val] ...)**

需要了解一下的规则是：

* 如果位置已有值存在，则忽略。
* 如果位置没有值的存在，则会在末尾进行扩展。

栗子：

```mysql
mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]');
+----------------------------------------------------+
| JSON_INSERT(@j, '$.a', 10, '$.c', '[true, false]') |
+----------------------------------------------------+
| {"a": 1, "b": [2, 3], "c": "[true, false]"}        |
+----------------------------------------------------+
```

### 数据合并

略过。。。。。

### 数据删除

**JSON_REMOVE(json_doc, path[, path] ...)**

> 这个看一下就可以了，就只是删除数据，没有什么特别的，就是删掉指定路径的数据。

### 数据替换

**JSON_REPLACE(json_doc, path, val[, path, val] ...)**

替换掉当前位置的值，这个与前面的数据插入有一点不同，规则上的。（正好相反）

**规则：指定路径上的值存在则替换数据，不存在则忽略**

栗子：

```mysql
mysql> SET @j = '{ "a": 1, "b": [2, 3]}';
mysql> SELECT JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]');
+-----------------------------------------------------+
| JSON_REPLACE(@j, '$.a', 10, '$.c', '[true, false]') |
+-----------------------------------------------------+
| {"a": 10, "b": [2, 3]}                              |
+-----------------------------------------------------+
```

### 数据插入或者更新

**JSON_SET(json_doc, path, val[, path, val] ...)**

感觉这个就是前边，数据插入和替换的合并版本 (ノ｀Д)ノ  ，分这么多真麻烦

规则对比：

* JSON_SET() 替换现有值并添加不存在的值。
* JSON_INSERT() 插入值而不替换现有值。
* JSON_REPLACE()仅替换 现有值。

### 数据转义的取消

**JSON_UNQUOTE(json_val)**

这个东西是在出现各种转义问题的时候使用的，具体的看下栗子，我也说不清楚：

```mysql
mysql> SELECT @@sql_mode;
+------------+
| @@sql_mode |
+------------+
|            |
+------------+

mysql> SELECT JSON_UNQUOTE('"\\t\\u0032"');
+------------------------------+
| JSON_UNQUOTE('"\\t\\u0032"') |
+------------------------------+
|       2                           |
+------------------------------+

mysql> SET @@sql_mode = 'NO_BACKSLASH_ESCAPES';
mysql> SELECT JSON_UNQUOTE('"\\t\\u0032"');
+------------------------------+
| JSON_UNQUOTE('"\\t\\u0032"') |
+------------------------------+
| \t\u0032                     |
+------------------------------+

mysql> SELECT JSON_UNQUOTE('"\t\u0032"');
+----------------------------+
| JSON_UNQUOTE('"\t\u0032"') |
+----------------------------+
|       2                         |
+----------------------------+
```



---

## 其他一些属性操作

这里的一些操作，在做分析数据和动态处理数据的时候可能会用到的。

都是一些很直接的具体方法，就直接给出方法定义就好了。

### 查看数据深度

**JSON_DEPTH(json_doc)**

返回JSON最大深度，空的深度为1

### 查看数据长度

**JSON_LENGTH(json_doc[, path])**

返回指定位置的数组或map长度，不会计算前套内的数据。

### 查看数据类型

**JSON_TYPE(json_val)**

用于查看JSON数据的类型，包括OBJECT，ARRAY等等 略过。。。。。

查看数据是否为有效JSON

**JSON_VALID(val)**

这个方法，看看就好了 ╮(╯_╰)╭

## 附加内容

### MySQL类型强制转换

| **名称**                                                     | **描述**                   |
| ------------------------------------------------------------ | -------------------------- |
| [`BINARY`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#operator_binary) | 将字符串转换为二进制字符串 |
| [`CAST()`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast) | 将值转换为特定类型         |
| [`CONVERT()`](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_convert) | 将值转换为特定类型         |

这里就说一下，CASE方法：

栗子：可以用来转字符串到JSON的：

```mysql
CAST('[true, false]' AS JSON)
```

> 具体或者另外另个方法，还是具体参考官方的文档：
>
> <https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html>

---

## 最后

没有了，就这些了。

虽然用JSON来保存数据，之后的操作会挺麻烦的，但感觉比以后为了业务去增加表字段来的好一些。

反正，这个坑已经挖了，就整理下这个各种操作的方法，**留着以后用来填坑**。

ε=(´ο｀*)))唉

2019-05-30

小杭





