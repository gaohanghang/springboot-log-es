# springboot-log-es


> 原文地址：[https://mp.weixin.qq.com/s/sarYCLCN3wDjsWxABl8gcQ](https://mp.weixin.qq.com/s/sarYCLCN3wDjsWxABl8gcQ)
> 

> 代码地址：[https://github.com/gaohanghang/springboot-log-es](https://github.com/gaohanghang/springboot-log-es)
> ES下载地址：[https://www.elastic.co/cn/downloads/elasticsearch](https://www.elastic.co/cn/downloads/elasticsearch)
> Kibana下载地址：[https://www.elastic.co/cn/downloads/kibana](https://www.elastic.co/cn/downloads/kibana)



先上效果图：


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734945422-43ba2f75-e015-4fff-9259-7d98c7861c56.png#align=left&display=inline&height=824&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1648&originWidth=3360&size=482284&status=done&style=none&width=1680)






不知道你有没有经历过被日志支配的恐惧？


我就经历过，以前在服务器上要找到一个请求经过所有链路的日志，并串联起来发现真的好难，而且有了日志还没用，最好还有有参数，有响应可以串联起来整个业务逻辑，最大程度进行场景复原。


那段找日志的时光真是不堪回首，令人难忘，好在后来我离开了再没去服务器上看过日志了。


针对这种场景，怎么解呢？针对每次请求如果我们生成一个id,每次打印日志的时候都把这个id打印出来，那么当我们搜索每次请求的时候，根据这个id进行搜索就行了,本文也是基于这个思路来实现这个功能的。


### 请求链路


client --->  tomcat ---> other API


为了简化我们的请求逻辑,这里假设client访问我们的tomcat,然后tomcat访问外部服务


### 记录请求参数以及响应


为了记录请求参数以及响应结果，我们需要借助Filter来解决这个问题


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551082-36d5704c-c131-4b1a-8ac5-dbeb9f48f7d8.webp#align=left&display=inline&height=954&margin=%5Bobject%20Object%5D&originHeight=954&originWidth=1080&size=0&status=done&style=none&width=1080)logging-filter.png


### 记录Restful请求以及响应


一般而言，我们的服务会调用其他服务，无论你是使用的dubbo还是http，我们都能通过拦截器来记录请求和响应参数，我们以rest来进行举例


rest-interceptor.png


由于我们的response是stream,我们想要消费多次(一次返回给调用者，一次用于记录),所以我们必须对这个response进行封装


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551512-41317fab-c6da-4c71-8893-d15b3742bbde.webp#align=left&display=inline&height=666&margin=%5Bobject%20Object%5D&originHeight=666&originWidth=1080&size=0&status=done&style=none&width=1080)


### Log4j2配置


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551090-303c53cc-f41f-43b0-a77c-bc077b373269.webp#align=left&display=inline&height=481&margin=%5Bobject%20Object%5D&originHeight=481&originWidth=1080&size=0&status=done&style=none&width=1080)


至此,我们的日志记录已经初具雏形了，这个时候当我们访问后端请求时,就可以通过返回的 requestId 去服务器上查询这次请求的链路日志


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551723-46aa4b37-e817-4131-9ab4-b680bb2c7e2f.webp#align=left&display=inline&height=368&margin=%5Bobject%20Object%5D&originHeight=368&originWidth=604&size=0&status=done&style=none&width=604)


后台日志如下


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551476-6b9c14d8-a97c-443c-928f-7eb316aed28f.webp#align=left&display=inline&height=240&margin=%5Bobject%20Object%5D&originHeight=240&originWidth=1080&size=0&status=done&style=none&width=1080)


**创建用户请求**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734663008-605e661b-074f-4322-a91f-5421b1591984.png#align=left&display=inline&height=750&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1500&originWidth=2416&size=150121&status=done&style=none&width=1208)


```java
http://localhost:8080/api/user


{
    "age": 11,
    "name": "111",
    "school": "111"
}
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734868477-aa72729f-80b6-4cda-bd2a-f04b59e6d4d7.png#align=left&display=inline&height=940&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1880&originWidth=3360&size=742187&status=done&style=none&width=1680)




查询时我们可以使用这样的命令


```
cat api.log | grep ${requestId}
```


这样就足够了吗？不，还不够好。我们可以将我们的日志发送到ElasticSearch,然后通过Kibana进行搜索，这样就不用每次查看日志都还要登录服务器了


### 自定义appender


由于需要将日志发送到ES,所以我们需要自定义appender来发送日志，一个简单的appender的如下


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551151-f96d0280-840f-48d6-84b9-2d36f96da468.webp#align=left&display=inline&height=1469&margin=%5Bobject%20Object%5D&originHeight=1469&originWidth=1080&size=0&status=done&style=none&width=1080)


然后在Log4j中进行如下配置即可


![](https://cdn.nlark.com/yuque/0/2021/webp/576791/1618364551614-dd1b3a78-f725-4d40-b569-31bd25dc3f70.webp#align=left&display=inline&height=560&margin=%5Bobject%20Object%5D&originHeight=560&originWidth=1080&size=0&status=done&style=none&width=1080)


这里我的es使用的是本地安装的es,所以在运行的时候需要先启动es


### 在Kibana中查询


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734424691-14cbd39d-8cf5-404c-b0e3-6f020b54dafc.png#align=left&display=inline&height=829&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1658&originWidth=656&size=114342&status=done&style=none&width=328)


Create index pattern

![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734238541-103ae942-7686-4199-8b89-7d56acf512b6.png#align=left&display=inline&height=833&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1666&originWidth=3358&size=485027&status=done&style=none&width=1679)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734272221-cb7070c1-035b-49dc-b34c-2dda8e41dd0f.png#align=left&display=inline&height=799&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1598&originWidth=3360&size=395655&status=done&style=none&width=1680)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734299201-d9dc31b7-31dd-4c48-83ba-c502ee56c583.png#align=left&display=inline&height=821&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1642&originWidth=3360&size=338715&status=done&style=none&width=1680)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734317575-bf915552-615d-44fa-afe0-824628ccfa17.png#align=left&display=inline&height=822&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1644&originWidth=3358&size=354325&status=done&style=none&width=1679)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734335347-137f1041-b4fc-42b6-803d-8fe67b13f0d5.png#align=left&display=inline&height=826&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1652&originWidth=3360&size=363853&status=done&style=none&width=1680)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734437635-c1f0be3a-754e-4e4d-b3df-f045f62f42fb.png#align=left&display=inline&height=829&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1658&originWidth=656&size=114342&status=done&style=none&width=328)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734465416-f4c726ea-46df-4b47-9b5d-160b2768e465.png#align=left&display=inline&height=823&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1646&originWidth=3350&size=586671&status=done&style=none&width=1675)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/576791/1618734544425-63699e88-7155-4d7d-ad92-5626b0f78ce0.png#align=left&display=inline&height=836&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1672&originWidth=3360&size=499621&status=done&style=none&width=1680)


至此，我们的日志平台就搭建起来了，以后别人就别想那么容易的推锅给你了。


所以的代码我都上传到了我的github: ([https://github.com/generalthink/code](https://github.com/generalthink/code))


以供大家参考


### 总结


核心思想就是让 requestId出现在需要记录日志的各个地方，然后通过requestId串联起来。







