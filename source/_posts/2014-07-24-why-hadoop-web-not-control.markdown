---
layout: post
title: "why hadoop web not control"
date: 2014-07-24 23:08:27 +0800
comments: true
categories: hadoop
tags: hadoop
share: true
description: 分析hadoop的web界面user.name
toc: true
---

hadoop 的web界面 user.name=xxx 为什么不能控制访问呢？我们一探究竟

<!--more-->
起因是因为有人问到，hadoop的web界面为什么不能控制访问人员呢？

例如 http://localhost:50070/dfshealth.jsp?user.name=xxx  ，其中这个xxx可以随便写均可以跳过权限控制？

PS：关于hadoop的web安全添加 这里有一篇文章 [<font color="#6868b4">hadoop添加web安全</font>](http://blog.csdn.net/caoshichaocaoshichao/article/details/13005635)

---

## 分析过程

1.先 [<font color="#6868b4">开启远程调试hadoop的debug模式</font>](http://blog.yuanxiaolong.cn/blog/2014/07/21/how-to-debug-hadoop-on-local/)

2.在core-site.xml中找到给hadoop添加权限的类

``` xml core-site.xml
<property>
  <name>hadoop.http.filter.initializers</name>
  <value>org.apache.hadoop.security.AuthenticationFilterInitializer</value>
</property>
```

看样子是个类似servlet的过滤器，先看一下这个类

``` java AuthenticationFilterInitializer.java

static final String PREFIX = "hadoop.http.authentication.";

static final String SIGNATURE_SECRET_FILE = AuthenticationFilter.SIGNATURE_SECRET + ".file";

@Override
public void initFilter(FilterContainer container, Configuration conf) {
  Map<String, String> filterConfig = new HashMap<String, String>();
  ... //省略无关代码
  String signatureSecretFile = filterConfig.get(SIGNATURE_SECRET_FILE);
  ... //省略无关代码
  try {
    StringBuilder secret = new StringBuilder();
    Reader reader = new FileReader(signatureSecretFile);
    int c = reader.read();
    while (c > -1) {
      secret.append((char)c);
      c = reader.read();
    }
    reader.close();
    filterConfig.put(AuthenticationFilter.SIGNATURE_SECRET, secret.toString());
  } catch (IOException ex) {
    throw new RuntimeException("Could not read HTTP signature secret file: " + signatureSecretFile);
  }
  ... //省略无关代码
  container.addFilter("authentication",
                      AuthenticationFilter.class.getName(),
                      filterConfig);
```

做了3件事情:

*  获取指定的权限文件
*  将权限文件放到filterConfig这个map里
*  将这个map传递给了AuthenticationFilter这个类，作为初始参数

3.再看一下AuthenticationFilter，继承了servlet的fitler接口，那么直奔doFilter方法吧！

``` java AuthenticationFilter.java
public class AuthenticationFilter implements Filter
```

我们在doFilter里打个断点，刷新界面，果然进入断点处。
![hadoop-web-filter](/images/hadoop/hadoop-web-filter.png)


里面就1个行比较可疑

``` java AuthenticationFilter.java
... //省略无关代码
token = authHandler.authenticate(httpRequest, httpResponse);
... //省略无关代码
```

这个authHander是个接口，实现类有2个 <font color="green"> KerberosAuthenticationHandler.java </font>和 <font color="green"> PseudoAuthenticationHandler.java </font> 到底哪个呢？

4.一般这种对接口的实例化，spring的IOC是一种，但是hadoop不可能依赖spring，那么它的实例化，十有八九在初始化的时候做的。看一下init方法

``` java AuthenticationFilter.java
@Override
public void init(FilterConfig filterConfig) throws ServletException {
  String configPrefix = filterConfig.getInitParameter(CONFIG_PREFIX);
  configPrefix = (configPrefix != null) ? configPrefix + "." : "";
  Properties config = getConfiguration(configPrefix, filterConfig);
  String authHandlerName = config.getProperty(AUTH_TYPE, null);
  String authHandlerClassName;
  if (authHandlerName == null) {
    throw new ServletException("Authentication type must be specified: simple|kerberos|<class>");
  }
  if (authHandlerName.equals("simple")) {
    authHandlerClassName = PseudoAuthenticationHandler.class.getName();
  } else if (authHandlerName.equals("kerberos")) {
    authHandlerClassName = KerberosAuthenticationHandler.class.getName();
  } else {
    authHandlerClassName = authHandlerName;
  }
  ... //省略无关代码
```

这里我们看到，用哪个类是根据 authHandlerName 是 <font color="#0f918d">simple</font>还是<font color="#0f918d">kerberos</font>，而且有configPrefix，说明是统一前缀的，就是 <font color="#b1892c">AuthenticationFilterInitializer.PREFIX </font>

因此这个 authHandlerName 就是 core-site.xml 的 hadoop.http.authentication.type <font color="#7c837f">（验证方式，默认为简单simple，也可自己定义class,需配置所有节点）</font>

![auth-simple](/images/hadoop/hadoop-web-auth-simple.png)

---

## 结论

由此可见，这里只判断了userName是否为空的情况，并没有严格校验内容。
