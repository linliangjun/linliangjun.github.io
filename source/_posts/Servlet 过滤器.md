---
title: Servlet 过滤器
tags:
  - Servlet
categories:
  - Java
abbrlink: 295680105
date: 2024-12-26 11:01:21
---

在 Servlet 规范中，由 Servlet 负责处理请求（ServletRequest）并写入数据到响应（ServletResponse）。对于一些公共的逻辑，例如：转换字符编码、身份认证和记录访问日志等，如果都在 Servlet 中处理，就显得太过臃肿，且也不符合单一职责的原则。因此需要一种能在请求到达 Servlet 之前或响应离开 Servlet 之后进行处理的机制。

<!--more-->

## 过滤器模式（Filter Pattern）

过滤器模式旨在通过一组处理步骤（过滤器（Filter））来处理数据流。其核心思想是将复杂的处理任务拆分成一个个独立的小的处理组件，然后按需组合这些组件，形成一个过滤器链（FilterChain）来逐步处理数据。每个过滤器只关心自己的一部分任务，从而使得系统变得更加灵活和可扩展。

## Servlet 中的过滤器

在 Servlet 2.3 规范中新增了 Filter 和 FilterChain 接口。其代码如下所示：

```java
package jakarta.servlet;

import java.io.IOException;

public interface FilterChain {

    void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException;
}
```

```java
package jakarta.servlet;

import java.io.IOException;

public interface Filter {

    default void init(FilterConfig filterConfig) throws ServletException {
    }

    // 调用 chain.doFilter(request, response) 将控制权交给下一个 Filter 或目标 Servlet
    // 否则，请求会被阻止，不会继续执行后续的 Filter 或 Servlet
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException;

    default void destroy() {
    }
}
```

## Servlet 中的过滤器的工作流程

1. 客户端请求：客户端发起请求，进入过滤器链。
2. 逐一过滤：过滤器链依次处理请求。每个过滤器都会对请求做一些预处理，并可以选择是否继续将请求传递给下一个过滤器。
3. 最终处理：在过滤器链的最后，会有一个实际的处理逻辑（Servlet），对请求做最终处理，生成响应。
4. 响应回传：响应结果通过过滤器链逆向返回，每个过滤器可以对响应进行后处理（例如修改响应内容、添加响应头等）。

![](2024-12-26_1001.png)

## Servlet 中的过滤器的配置方式

在 web.xml 配置文件中指明 Filter 类以及应用到的 URL，例如：

```xml
<filter>
    <filter-name>MyFilter</filter-name>
    <filter-class>com.example.MyFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>MyFilter</filter-name>
    <url-pattern>/myServlet</url-pattern>
</filter-mapping>
```

也可以使用注解方式配置，例如：

```java
@WebFilter("/myServlet")
public class MyFilter implements Filter {

  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
    // 具体的过滤逻辑
  }
}
```

## 总结

Filter 能够在请求和响应的生命周期中进行灵活的处理。合理地使用可以提高 Web 应用的模块化、可维护性和可扩展性。它和 Servlet 是互补的，通过分离关注点来简化复杂的 Web 应用。
