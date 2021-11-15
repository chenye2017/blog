---
title: go 数据库原生操作
date: 2020-06-09 20:28:16
tags: [go,mysql]
categories: go
---

go web 开发，操作mysql 是基础啦

<!--more-->

```
import (
   "database/sql"
   _ "github.com/go-sql-driver/mysql"
)
首先是包的引入，很常见吧，下面那个驱动引入，却未使用可能是只希望使用这个包的 init()方法
```

> 通常来说, 不应该直接使用驱动所提供的方法, 而是应该使用 sql.DB, 因此在导入 mysql 驱动时, 这里使用了匿名导入的方式(在包路径前添加 _), 当导入了一个数据库驱动后, 此驱动会自行初始化并注册自己到Golang的database/sql上下文中, 因此我们就可以通过 database/sql 包提供的方法访问数据库了.
>
> 可能这就是interface 的作用吧



```
type DbWorker struct {
    //mysql data source name
    Dsn string 
}

func main() {
    dbw := DbWorker{
        Dsn: "user:password@tcp(127.0.0.1:3306)/test",
    }	
    db, err := sql.Open("mysql",
        dbw.Dsn) // 返回一个sql.DB 的指针
    if err != nil {
        panic(err)
        return
    }
    defer db.Close()
}
```

> 1. sql.Open并不会立即建立一个数据库的网络连接, 也不会对数据库链接参数的合法性做检验, 它仅仅是初始化一个sql.DB对象. 当真正进行第一次数据库查询操作时, 此时才会真正建立网络连接;
> 2. sql.DB表示操作数据库的抽象接口的对象，但不是所谓的数据库连接对象，sql.DB对象只有当需要使用时才会创建连接，如果想立即验证连接，需要用Ping()方法;
> 3. sql.Open返回的sql.DB对象是协程并发安全的.
> 4. sql.DB的设计就是用来作为长连接使用的。不要频繁Open, Close。比较好的做法是，为每个不同的datastore建一个DB对象，保持这些对象Open。如果需要短连接，那么把DB作为参数传入function，而不要在function中Open, Close。
>
> 所以我们平时用的都是长连接，在池子里面可以复用的啦。平时我们在 prepare ，query(查询)， exec(插入，修改) 之后都要及时 close



> MySQL 5.5 之前， UTF8 编码只支持1-3个字节,从MYSQL5.5开始，可支持4个字节UTF编码utf8mb4，一个字符最多能有4字节，utf8mb4兼容utf8，所以能支持更多的字符集;关于emoji表情的话mysql的utf8是不支持，需要修改设置为utf8mb4，才能支持。
>
> (4 字节的utf8 我们应该一直在用啦)



插入代码

```
stmt, err := db.Prepare("insert tasks (content, user, create_at, update_at, deleted) values (?, ?, ?, ?, ?)")

	if err != nil {
		log.Println(err)
	}

	defer stmt.Close()

	now, _ := time.Parse("2006-01-02 15:04:05", "2016-01-02 15:04:05")

	update := time.Now().Format("2006-01-02 15:04:05")

	result, err := stmt.Exec("测试", "cy", now, update, 0)
	if err != nil {
		log.Println(err)
	}

	fmt.Println(result.LastInsertId())
```

修改代码

```
stmt, err := db.Prepare("update tasks set content = ? where id = ?")

	if err != nil {
		log.Println(err)
	}

	defer stmt.Close()

	stmt.Exec("ceshi11111", 2)
```

查询代码

```
db, err := sql.Open("mysql", "root:wyqnkxk2012_CY@tcp(118.184.219.156)/cy")
	if err != nil {
		log.Println(err)
	}



	err = db.Ping()
	if err != nil {
		log.Println(err)
	}

	//update(db)
	//return

	stmt, err := db.Prepare("select * from tasks where id = ?")
	if err != nil {
		log.Println(stmt)
	}
	defer stmt.Close()

	rows, err := stmt.Query(3)

	defer rows.Close()

	if err != nil {
		log.Println(rows)
	}

	arr := []*task{}

	for rows.Next() {
		tmp := &task{}
		rows.Scan(&tmp.id, &tmp.content, &tmp.user, &tmp.create_at, &tmp.update_at, &tmp.deleted)
		arr = append(arr, tmp)
	}
```

