

select 可以防止 goroutine deedlock, 如果单独写在方法里，阻塞没有读取，会报deedlock.

