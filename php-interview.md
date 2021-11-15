include 和  require 的区别

```
 include 引入文件的时候，如果碰到错误，会给出提示，并继续运行下边的代码。
 require 引入文件的时候，如果碰到错误，会给出提示，并停止运行下边的代码
 
 required_once
```



composer insall和update的区别

```
install读取lock文件，没有的话，则读取json文件，并生成lock (install 不更新已有的lock 文件)
update会读取json，拉取最新依赖，把新的版本写入lock
也就是说，当本地没有lock文件的时候，install和update是一样的
```



* php7相对于 php5.6

  ```
  1.try catch  \Throwable
  2. ??
  3. 返回值类型可以定义
  ```

  
