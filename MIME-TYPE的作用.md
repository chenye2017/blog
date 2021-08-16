---
title: MIME TYPE的作用
date: 2018-04-23 21:44:00
tags: [PHP,HTTP]
categories: HTTP
---

其实本来不用单独写这篇文章的，可以放在http协议中，因为他本来就是关联这个http请求和返回信息中的content type，但其实用的还挺多，主要就是上传文件这块和php不借助插件而是通过输出html表格元素让页面按照excel来解析的方式生成excel。

<!--more-->

>一、
>
>首先，我们要了解浏览器是如何处理内容的。在浏览器中显示的内容有 HTML、有 XML、有 GIF、还有 Flash ……那么，浏览器是如何区分它们，决定什么内容用什么形式来显示呢？答案是 MIME Type，也就是该资源的媒体类型。
>媒体类型通常是通过 HTTP 协议，由 Web 服务器告知浏览器的，更准确地说，是通过 Content-Type 来表示的，例如:
>Content-Type: text/HTML
>表示内容是 text/HTML 类型，也就是超文本文件。为什么是“text/HTML”而不是“HTML/text”或者别的什么？MIME Type 不是个人指定的，是经过 ietf 组织协商，以 RFC 的形式作为建议的标准发布在网上的，大多数的 Web 服务器和用户代理都会支持这个规范 (顺便说一句，Email 附件的类型也是通过 MIME Type 指定的)。
>通常只有一些在互联网上获得广泛应用的格式才会获得一个 MIME Type，如果是某个客户端自己定义的格式，一般只能以 application/x- 开头。
>XHTML 正是一个获得广泛应用的格式，因此，在 RFC 3236 中，说明了 XHTML 格式文件的 MIME Type 应该是 application/xHTML+XML。(现在不用xml，而是用json，一般在返回头中都有)
>
>```
>$response->header('Content-Type', 'application/json');
>```
>
>注意这个content type，不仅在response中有，在request中也有，
>
>```
>application/x-www-form-urlencoded（使用HTTP的POST方法提交的表单）
>multipart/form-data（同上，但主要用于表单提交时伴随文件上传的场合）
>```
>
>上面的两种应该是我们平时最常见到的，一种是普通的表单提交，一种是当表单中有上传文件的时候进行的提交方式中content type的设置(好像默认的表单提交就是第一种，不用设置，只有上传的时候需要特殊设置，像我们itbasic上面，大部分时候用的是ajax进行提交，而不是表单，content type在ajax中应该封装成了text/json)
>
>```
><form action="http://192.168.33.10:56732/bbs/login/test" method="post" enctype="multipart/form-data">
><label for="file">文件名：</label>
><input type="file" name="file" id="file"><br>
><input type="text" name="name" id="name"><br>
><input type="submit" name="submit" value="提交">
></form>
>```
>
>当然，处理本地的文件，在没有人告诉浏览器某个文件的 MIME Type 的情况下，浏览器也会做一些默认的处理，这可能和你在操作系统中给文件配置的 MIME Type 有关。比如在 Windows 下，打开注册表的“HKEY_LOCAL_MACHINESOFTWAREClassesMIMEDatabaseContent Type”主键，你可以看到所有 MIME Type 的配置信息。
>
> 

更详细的介绍[https://www.cnblogs.com/jsean/articles/1610265.html](https://www.cnblogs.com/jsean/articles/1610265.html)

因为这篇文章复制有很多重复的部分（▄█▀█●），复制下来改很麻烦。

认识的几点就是：服务器端要存储文件后缀名和mime type的对应关系，我是这样理解的，虽然在linux下文件后缀名并没有什么卵用，感觉这都是content type的作用，但在windows下起了很大的作用啊，在服务器端保存文件的使用，需要通过content type对应的mime type 转换成文件后缀名，然后保存（虽然感觉直接读上传的文件名就可以了，感觉也许是兼容那些没有文件后缀名的情况吧，linux？）

response 里面content type的作用很明显是通知客户端怎么解析这个文件。


