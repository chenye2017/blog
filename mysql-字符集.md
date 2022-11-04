mysql 中保存emoji表情是一个很常见的需求，之前在做航海福利的时候用户保存自己的收件名和收件地址，航海庆典活动寄送给主播的明信片，但凡用户可以输入的地方，都可能存在emoji 表情。 

首先我们说下为什么mysql 通常情况下不支持emoji。因为通常我们设置字符集是utf8, mysql 的utf8 为了存储效率等，是一种阉割版的utf8, 最大智齿3字节，而emoji 占用4字节，所以保存不成功。mysql 中4字节的字符是utf8mb4， 通过一下确保mysql 支持emoji表情

1.保证列字段字符集是utf8mb4。

```
show full columns from table_name;
可以查看每一列的字符集。（一般只展示字符集的排列规则，但我们通过字符集的排列规则就能知道这列是什么字符集，比如 utf8_general_ci, 我们就知道字符集是utf8。 又比如 utf8mb4_generabl_ci, 我们就知道字符集是 utf8mb4）。如果单列不是utf8mb4字符集，修改：

ALTER TABLE table_name MODIFY column_name varchar(120) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL default '';

拓展：
show variables like '%character%';  // 查询字符集
show variables like 'collation%'; // 查询字符排列规则
show charset; //查看MYSQL所支持的字符集
```

2.上面的列可以保证mysql 存储没问题，接下来就是dsn 的设置

```
live:password@tcp(db-host:3312)/live_guard?timeout=2s&readTimeout=2s&writeTimeout=2s&parseTime=true&loc=Local&charset=utf8mb4
之前这个charset 设置成2个，charset=utf8,utf8mb4 这样好像也会出问题，删掉utf8
```



3.检查数据库配置

```
show variables like 'character_set_%';
```

![image-20220912214146718](/Users/yechen/Library/Application Support/typora-user-images/image-20220912214146718.png)



那个character_set_system 似乎不重要。









原文: 掘金小册。从根上理解mysql。字符集那章

比较核心的几个点：

1.`MySql有4个级别的字符集和比较规则，分别是：

- 服务器级别
- 数据库级别
- 表级别
- 列级别

没有验证仅仅列级别设置了utf8mb4 能不能保存emoji表情 ？一般这些都会设置成一样的utfmb4.

2.乱码的原因。当前服务器的编码和character_set_client 的编码不一致，导致解析失败。

```
说到底，字符串在计算机上的体现就是一个字节串，如果你使用不同字符集去解码这个字节串，最后得到的结果可能让你挠头。

我们知道字符'我'在utf8字符集编码下的字节串长这样：0xE68891，如果一个程序把这个字节串发送到另一个程序里，另一个程序用不同的字符集去解码这个字节串，假设使用的是gbk字符集来解释这串字节，解码过程就是这样的：

首先看第一个字节0xE6，它的值大于0x7F（十进制：127），说明是两字节编码，继续读一字节后是0xE688，然后从gbk编码表中查找字节为0xE688对应的字符，发现是字符'鎴'

继续读一个字节0x91，它的值也大于0x7F，再往后读一个字节发现木有了，所以这是半个字符。

所以0xE68891被gbk字符集解释成一个字符'鎴'和半个字符。

假设用iso-8859-1，也就是latin1字符集去解释这串字节，解码过程如下：

先读第一个字节0xE6，它对应的latin1字符为æ。

再读第二个字节0x88，它对应的latin1字符为ˆ。

再读第三个字节0x91，它对应的latin1字符为‘。

所以整串字节0xE68891被latin1字符集解释后的字符串就是'æˆ‘'

可见，如果对于同一个字符串编码和解码使用的字符集不一样，会产生意想不到的结果，作为人类的我们看上去就像是产生了乱码一样。
```



3.mysql 接受数据转码过程如下。



![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20220912212321.png)

- 服务器认为客户端发送过来的请求是用`character_set_client`编码的。

  假设你的客户端采用的字符集和 ***character_set_client*** 不一样的话，这就会出现意想不到的情况。比如我的客户端使用的是`utf8`字符集，如果把系统变量`character_set_client`的值设置为`ascii`的话，服务器可能无法理解我们发送的请求，更别谈处理这个请求了。

- 服务器将把得到的结果集使用`character_set_results`编码后发送给客户端。

  假设你的客户端采用的字符集和 ***character_set_results*** 不一样的话，这就可能会出现客户端无法解码结果集的情况，结果就是在你的屏幕上出现乱码。比如我的客户端使用的是`utf8`字符集，如果把系统变量`character_set_results`的值设置为`ascii`的话，可能会产生乱码。

- `character_set_connection`只是服务器在将请求的字节串从`character_set_client`转换为`character_set_connection`时使用，它是什么其实没多重要，但是一定要注意，该字符集包含的字符范围一定涵盖请求中的字符，要不然会导致有的字符无法使用`character_set_connection`代表的字符集进行编码。比如你把`character_set_client`设置为`utf8`，把`character_set_connection`设置成`ascii`，那么此时你如果从客户端发送一个汉字到服务器，那么服务器无法使用`ascii`字符集来编码这个汉字，就会向用户发出一个警告。

知道了在`MySQL`中从发送请求到返回结果过程里发生的各种字符集转换，但是为啥要转来转去的呢？不晕么？

答：是的，很头晕，所以我们通常都把 ***character_set_client*** 、***character_set_connection***、***character_set_results*** 这三个系统变量设置成和客户端使用的字符集一致的情况，这样减少了很多无谓的字符集转换。为了方便我们设置，`MySQL`提供了一条非常简便的语句：

```
SET NAMES 字符集名;
```

这一条语句产生的效果和我们执行这3条的效果是一样的：

```
SET character_set_client = 字符集名;
SET character_set_connection = 字符集名;
SET character_set_results = 字符集名;
```

