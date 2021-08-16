---
title: rpc 相关
date: 2021-06-11 10:02:14
tags: [go, rpc]
categories: go
---

最近在学习rpc 相关内容，简单记录下新路历程。

<!--more-->

## 快速开始##

https://zhuanlan.zhihu.com/p/161473581

tips: go get 和 go install 的区别  https://segmentfault.com/a/1190000021429226

## 问题##

### proto 文件名冲突###

公司内部服务之间的调用逐渐开始都切到rpc，我们的rpc 文件都放在各自组内的仓库进行统一管理

![](https://cytuchuang-1256930988.cos.ap-shanghai.myqcloud.com/20210611100945.png)

为了防止函数名冲突，我们一般建立对应的文件夹（这个在go项目中经常这样用），然后编写proto 文件，利用protoc-gen-go 二进制文件生成对应的go文件。但是最近遇到这样一个报错

```
panic: proto: file "common.proto" is already registered
    previously from: "github.com/soft/test-platform.go/common"
    currently from:  "github.com/soft/proto-asterix/asterix"
```

我们组的proto 文件名和 别的组的proto 文件名一样，按理说go 包名前缀 比如 code.xxx.cn/apps/sh-apps-rpc/  就是为了解决这种情况的存在而产生的，为什么还会产生这样的报错， https://stackoverflow.com/questions/67693170/proto-file-is-already-registered-with-different-packages ， 真的证明上述代码是bug， so 按照提示做了如下升级

```
go install google.golang.org/protobuf/cmd/protoc-gen-go@febffdd
google.golang.org/protobuf v1.26.1-0.20210525005349-febffdd88e85
```

上述冲突没了，结果又引入了下面问题



### protoc 生成go 文件失败###

```
--go_out: protoc-gen-go: plugins are not supported; use 'protoc --go-grpc_out=...' to generate gRPC
```

升级完了之后我再生成go文件就出现了上述错误，提示的这么清晰，那我就改呗，结果 还是生成失败，查了下， protoc-gen-go 这个工具原本在github.com 仓库下，后来google 公司准备迁移到自己公司下，但是代码还没发完，在review 中，所以还用不了，所以对于protoc-gen-go 还是得用github.com 仓库下的，（虽然protobuf 我们已经升级到google 仓库下）。



## 文章参考##

https://juejin.cn/post/6844904134332645383

