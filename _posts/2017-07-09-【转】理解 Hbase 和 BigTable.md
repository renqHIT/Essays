---
title: 【译】理解 Hbase 和 BigTable：Understanding HBase and BigTable
description: 【译】理解 Hbase 和 BigTable：Understanding HBase and BigTable
header: 【译】理解 Hbase 和 BigTable：Understanding HBase and BigTable
---
> 原文地址：https://dzone.com/articles/understanding-hbase-and-bigtab    

学习HBase（Google BigTable 的具体实现）最困难的地方在于，HBase的概念很难让人理解。    

不幸的是，在HBase和BigTable的介绍中，都包含了table这个单词，很容易让学过关系型数据库的人们（包括我自己）产生困惑。    

这篇文章致力于从概念的角度介绍这个分布式数据存储系统。阅读之后，当你遇到想使用HBase或者是想使用传统数据库的时候，你就可以有一个清晰的认识。

### 一切都在术语中

Google的[BigTable论文](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf) 清晰的指出了什么是BigTable。这里是其中Data Model一节的第一句话：    
> Bigtable is a sparse, distributed, persistent multidimensional sorted map.    

接下来，论文继续解释道：
> The map is indexed by a row key, column key, and a timestamp; each value in the map is an uninterpreted array of bytes.

在HBase的Wiki介绍中，有如下描述：
> HBase uses a data model very similar to that of Bigtable. Users store data rows in labelled tables. A data row has a sortable key and an arbitrary number of columns. The table is stored sparsely, so that rows in the same table can have crazily-varying columns, if the user likes.

尽管看起来让人难以理解，但是一次理解一个单词，也是有意义的。下面我将按照以下顺序解释这些单词：map, persistent, distributed, sorted, multidimensional, 和sparse。

### map
HBase和BigTable的核心是一个map。依靠你的编程语言背景，你可以很容易理解，这就像PHP中的关联数组，Python中的字典，Ruby中的哈希，或者是JavaScript中的对象。    

从Wiki中的解释来看，map就是"由key和value组成的数据结构，其中每一个key和一个value相关联"。

使用JavaScript的对象语法，这是一个简单的map实例：

    {
      "zzzzz" : "woot",
      "xyz" : "hello",
      "aaaab" : "world",
      "1" : "x",
      "aaaaa" : "y"
    }    

### persistent
Persistence（持久性）只是表示当你的程序结束或数据入口关闭后，你保存在map中的数据会被持久化，这个和其他的持久化存储方式没区别。

### distributed
Hbase和BigTable构建在分布式文件系统上，以便底层文件存储可以在独立机器阵列之间传播。

HBase可以运行在HDFS上，也可以运行在Amazon的Simple Storage Service上，同时BigTable可以运行在Google File System上。

数据以类似的方式在多个参与节点中复制，以便数据在基于独立冗余磁盘阵列的系统中的光盘之间进行条带化。

在本文中，我们并不关心使用哪个分布式文件系统实现。要了解的事情是它是分布式的，它提供了一个保护机制，例如，集群内的某个节点发生故障，可以做到容灾即可。

### sorted
与大多数Map实现不同，在Hbase/BigTable中，键值对以严格的字母顺序保存。 也就是说，键“aaaaa”的存储行应该在键“aaaab”的旁边，并且距离具有键“zzzzz”的存储行非常远。

回到我们的map示例，排序后的版本如下：

    {
      "1" : "x",
      "aaaaa" : "y",
      "aaaab" : "world",
      "xyz" : "hello",
      "zzzzz" : "woot"
    }

因为这些系统往往是特别庞大的，并且数据是分布式存储，所以这种排序功能其实非常重要。具有类似键的行的空间优势确保了当一个操作必须扫描表格时，查询最感兴趣的内容将彼此靠近。

这在选择行key时很重要。 例如，假如一个表的关键字是域名，最有意义的是以反向符号（即“com.jimbojw.www”而不是“www.jimbojw.com”）列出它们，以便关于子域名的行将在父域名行附近。

继续解释域名的例子，如果是以反向符号存储的话，假如有一行数据是“mail.jimbojw.com”，那么它应该被存储在“www.jimbojw.com”行的旁边，而不是在“mail.xyz.com”旁边。

需要注意的是，在HBase和BigTable中的value是不被排序的，只有key被排序。除了这个，其他的和普通的map实现是一样的。

### multidimensional
到目前为止，我们没有提到任何“列（columns）”的概念，而是将“表”视为概念中的常规Hash/Map。 这是完全有意为之的。单词“列”是另一个人为引入的单词，如“table”和“base”，它是多年关系型数据库从业人员的命名习惯。

相反，我更喜欢把它想象为一个map的嵌套结构。比如如下结构：

    {
      "1" : {
        "A" : "x",
        "B" : "z"
      },
      "aaaaa" : {
        "A" : "y",
        "B" : "w"
      },
      "aaaab" : {
        "A" : "world",
        "B" : "ocean"
      },
      "xyz" : {
        "A" : "hello",
        "B" : "there"
      },
      "zzzzz" : {
        "A" : "woot",
        "B" : "1337"
      }
    }

在上述例子中，你会发现每一个内层的map都有两个key，A和B。从现在开始，每一个外层的map视为一行，每一个内层的map中的A、B视为A、B列。

创建表的时候，需要首先指定好表的列，后续难以修改，或不可修改。同时，添加新的列的代价也是非常昂贵的。

不过幸运的是，在每一个列中，同样可以添加无数的列，就像JSON中一样，举例如下：

    {
      // ...
      "aaaaa" : {
        "A" : {
          "foo" : "y",
          "bar" : "d"
        },
        "B" : {
          "" : "w"
        }
      },
      "aaaab" : {
        "A" : {
          "foo" : "world",
          "bar" : "domination"
        },
        "B" : {
          "" : "ocean"
        }
      },
      // ...
    }

上面的示例中，A列有两个子列：foo和bar，B列有一个子列，用空字符串标识。

在查询HBase和BigTable的时候，必须提供数据的全限定名，比如"A:foo", "A:bar"和"B:"。

请注意，尽管列是静态的，但列的子列不是。比如以下结构：

    {
      // ...
      "zzzzz" : {
        "A" : {
          "catch_phrase" : "woot",
        }
      }
    }

在这种情况下，“zzzzz”行只有一列“A：catch_phrase”。 因为每一行可能有不同的列数，所以没有内置的方法来查询所有列中所有子列的列表。要获取该信息，必须进行全表扫描。然而，可以查询所有列的内容，因为这些列是不可变的。

Hbase/BigTable表示的最终维度是时间。所有数据使用整数时间戳或用户选择的其他整数进行统一时间控制。客户端可以在插入数据时指定时间戳。

假设有如下采用某一时间序列存储的数据：

    {
      // ...
      "aaaaa" : {
        "A" : {
          "foo" : {
            15 : "y",
            4 : "m"
          },
          "bar" : {
            15 : "d",
          }
        },
        "B" : {
          "" : {
            6 : "w"
            3 : "o"
            1 : "w"
          }
        }
      },
      // ...
    }

每个列的子列可能有自己的规则来设置要保留的版本数量（单元格由其行键/列对标识）。在大多数情况下，应用程序将简单地询问给定单元格的数据，而不指定时间戳。 在这种常见情况下，Hbase/BigTable将返回最新的版本（具有最高时间戳的版本），因为它们以倒序的时间顺序存储。

如果应用程序在给定的时间戳中请求给定行，则Hbase将返回单元格数据，其中时间戳小于或等于提供的时间戳。

使用上述示例的Hbase表，查询“aaaaa”/“A：foo”的行/列将返回“y”，同时查询“aaaaa”/“A：foo”/ 10的行/列/时间戳将返回“M”。 查询“aaaaa”/“A：foo”/ 2的行/列/时间戳将返回空结果。

### sparse
最后一个关键字是sparse（稀疏的）。 如上文已经提到的，给定行可以在每个列族中具有任意数量的列，或者根本没有列。 另一种类型的稀疏性是基于行的间隙，这仅仅意味着键之间可能存在间隙。

当然，如果您在本文的基于map的术语中考虑过Hbase/BigTable，而不是在RDBMS中看到类似的概念，那么这是非常有道理的。（言外之意就是说，在HBase/BigTable中的数据是稀疏的，而在RDBMS中不经常是。）

### 综述

以上就是HBase/BigTable的概念介绍。

> 原文地址：https://dzone.com/articles/understanding-hbase-and-bigtab    
