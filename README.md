# springboot-log-es


> 原文地址：https://mp.weixin.qq.com/s/sarYCLCN3wDjsWxABl8gcQ



不知道你有没有经历过被日志支配的恐惧？

我就经历过，以前在服务器上要找到一个请求经过所有链路的日志，并串联起来发现真的好难，而且有了日志还没用，最好还有有参数，有响应可以串联起来整个业务逻辑，最大程度进行场景复原。

那段找日志的时光真是不堪回首，令人难忘，好在后来我离开了再没去服务器上看过日志了。

针对这种场景，怎么解呢？针对每次请求如果我们生成一个id,每次打印日志的时候都把这个id打印出来，那么当我们搜索每次请求的时候，根据这个id进行搜索就行了,本文也是基于这个思路来实现这个功能的。

### 请求链路

client --->  tomcat ---> other API

为了简化我们的请求逻辑,这里假设client访问我们的tomcat,然后tomcat访问外部服务

### 记录请求参数以及响应

为了记录请求参数以及响应结果，我们需要借助Filter来解决这个问题

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcReibgxCWenNRzc1kQbKibmzdPoO53eRbAluzRZibX9Hhb4W828LrB1g1vIA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)logging-filter.png

### 记录Restful请求以及响应

一般而言，我们的服务会调用其他服务，无论你是使用的dubbo还是http，我们都能通过拦截器来记录请求和响应参数，我们以rest来进行举例

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcRejfSTbAmT0iaauyHEiateXric5QWO0QyjviaVHol8bXqTaQ95mllj7oj5QA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)rest-interceptor.png

由于我们的response是stream,我们想要消费多次(一次返回给调用者，一次用于记录),所以我们必须对这个response进行封装

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcRe9J7ytNmpwq5d4aYGNicibJYv55oFjpX7vMmZ7N6ElNDevWJY3cc45d1w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)rest-config.png

### Log4j2配置

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcReicoENfM648cxUzXCrWpDrxOrOIqTrOEbROrbib3cru8GbrMdGlRbicHOg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)log4j2-config1.png

至此,我们的日志记录已经初具雏形了，这个时候当我们访问后端请求时,就可以通过返回的requestId去服务器上查询这次请求的链路日志

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcRe6vXz7icR3mjT8MAINdYY9GvQA8GTicyoFYF1n8FlYFlCPBc5ShgPRw4w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)test-logging.png

后台日志如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcReJdqbibjwop3yaDW8riaEg3DssBQIIcdTOkOGicLU1KBV1dc2cFnqgic8MA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)log.png

查询时我们可以使用这样的命令

```
cat api.log | grep ${requestId}
```

这样就足够了吗？不，还不够好。我们可以将我们的日志发送到ElasticSearch,然后通过Kibana进行搜索，这样就不用每次查看日志都还要登录服务器了

### 自定义appender

由于需要将日志发送到ES,所以我们需要自定义appender来发送日志，一个简单的appender的如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcRecmE2hfnY5rVyGW2TQ40yMbBeSsw0G2fTdTX1d65WJu8GDX7D4XRxaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)esAppender.png

然后在Log4j中进行如下配置即可

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcReqhdSQyZkVMwNWTyicfhyuibyKlS210ruz069F3pB9PMzsh26ko0f5O9Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)log4j2-config2.png

这里我的es使用的是本地安装的es,所以在运行的时候需要先启动es

### 在Kibana中查询

![图片](https://mmbiz.qpic.cn/mmbiz_png/iaj3Tu78ibIgfWR5275HeMojgNZp3nBcReEmcibAMzATn90uagkgxHpfLthJPWgTjpFWGib6vd7e8F1va5rKmU9K1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)kibana-search-log.png

至此，我们的日志平台就搭建起来了，以后别人就别想那么容易的推锅给你了。

所以的代码我都上传到了我的github: (https://github.com/generalthink/code)

以供大家参考

### 总结

核心思想就是让 requestId出现在需要记录日志的各个地方，然后通过requestId串联起来。
