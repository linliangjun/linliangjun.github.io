---
title: Spring Security
abbrlink: 308886303
date: 2024-12-26 21:51:55
tags:
  - Spring Security
categories:
  - Java
---

Spring Security 是一个 Java 框架，用于保护应用程序的安全性。它提供了一套全面的安全解决方案，包括身份验证、授权、防止攻击等功能。Spring Security 基于过滤器链的概念，可以轻松地集成到任何基于 Spring 的应用程序中。它支持多种身份验证选项和授权策略，开发人员可以根据需要选择适合的方式。此外，Spring Security 还提供了一些附加功能，如集成第三方身份验证提供商和单点登录，以及会话管理和密码加密等。总之，Spring Security 是一个强大且易于使用的框架，可以帮助开发人员提高应用程序的安全性和可靠性。

<!--more-->

## Spring Security 的过滤器链

![](2024-12-26-1001.png)

## 建造者模式（Builder Pattern）与 SecurityBuilder

建造者模式旨在将一个复杂对象的构建过程与它的表示分离，使得同样的构建过程可以创建不同的表示。简单来说，它通过一步步地构建一个复杂对象，并将构建过程和表示分开，以便于在构建过程中控制产品的各个部分。

在 Spring Security 中，与安全性相关的组件（例如：SecurityFilterChain、AuthenticationManager）都是通过建造者模式实例化的。Spring Security 为此抽象出了一个 SecurityBuilder 接口，源码如下所示：

```java
package org.springframework.security.config.annotation;

public interface SecurityBuilder<O> {

    O build() throws Exception;
}
```

AbstractSecurityBuilder 是 SecurityBuilder 的一个抽象实现，它能确保只会构建一次安全组件，源码如下所示：

```java
package org.springframework.security.config.annotation;

import java.util.concurrent.atomic.AtomicBoolean;

public abstract class AbstractSecurityBuilder<O> implements SecurityBuilder<O> {

    private AtomicBoolean building = new AtomicBoolean();

    private O object;

    @Override
    public final O build() throws Exception {
        // CAS 乐观锁
        if (this.building.compareAndSet(false, true)) {
            this.object = doBuild();
            return this.object;
        }
        throw new AlreadyBuiltException("This object has already been built");
    }

    public final O getObject() {
        if (!this.building.get()) {
            throw new IllegalStateException("This object has not been built");
        }
        // 这里是有可能获取到 null 的
        return this.object;
    }

    protected abstract O doBuild() throws Exception;
}
```

HttpSecurityBuilder 是 SecurityBuilder 的一个抽象实现，旨在构建 DefaultSecurityFilterChain 对象（Spring Security 的过滤器链）。源码如下所示：

```java
package org.springframework.security.config.annotation.web;

import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.DefaultSecurityFilterChain;

import jakarta.servlet.Filter;

public interface HttpSecurityBuilder<H extends HttpSecurityBuilder<H>>
    extends SecurityBuilder<DefaultSecurityFilterChain> {

    // 省略了部分当前不重要的方法

    H authenticationProvider(AuthenticationProvider authenticationProvider);

    H userDetailsService(UserDetailsService userDetailsService) throws Exception;

    H addFilterAfter(Filter filter, Class<? extends Filter> afterFilter);

    H addFilterBefore(Filter filter, Class<? extends Filter> beforeFilter);

    H addFilter(Filter filter);
}
```

所以，可以总结出通过 SecurityBuilder 构建安全组件的流程为：

1. 以链式调用的方式传入子组件（这个要看具体的实现类支持传入什么子组件）
2. 最后调用 build 方法构建安全组件（如果是 AbstractSecurityBuilder，多次调用会报错）

## 配置式编程与 SecurityConfigurer

SecurityBuilder 为构建安全组件提供了统一的流程，只要传入需要的子组件即可。但是面对不同的安全场景，有时候需要传入某个子组件，有时候又不需要；有时候子组件需要设置一个属性，有时候又不需要设置。如果以更加本质的视角看待这些行为，其实就是根据不同的场景，设置不同的配置。

这就是命令式编程与配置式编程的区别了。前者通过明确的步骤和命令告诉计算机如何完成任务；后者通过设置高层次的配置定义行为和规则，而不需要指定具体的操作步骤，框架或工具根据配置自动推断并执行相关操作。

对于安全场景，很难做到面面俱到，因此 Spring Security 也抽象出了 SecurityConfigurer 接口。源码如下所示：

```java
package org.springframework.security.config.annotation;

public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {

    void init(B builder) throws Exception;

    void configure(B builder) throws Exception;
}
```

## 支持配置式编程的 SecurityBuilder

AbstractConfiguredSecurityBuilder 是支持配置式编程的 SecurityBuilder，是一个 SecurityConfigurer 容器。维护 SecurityConfigurer 的方法如下所示：

```java
package org.springframework.security.config.annotation;

// 省略了 import

public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
    extends AbstractSecurityBuilder<O> {

    // 实际存储 SecurityConfigurer 的容器
    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();

    // 是否允许添加相同类型的 SecurityConfigurer
    private final boolean allowConfigurersOfSameType;

    private <C extends SecurityConfigurer<O, B>> void add(C configurer) {
        // 省略了实现
    }

    public <C extends SecurityConfigurer<O, B>> List<C> getConfigurers(Class<C> clazz) {
        // 省略了实现
    }

    public <C extends SecurityConfigurer<O, B>> List<C> removeConfigurers(Class<C> clazz) {
        // 省略了实现
    }

    public <C extends SecurityConfigurer<O, B>> C getConfigurer(Class<C> clazz) {
        // 省略了实现
    }

    public <C extends SecurityConfigurer<O, B>> C removeConfigurer(Class<C> clazz) {
        // 省略了实现
    }
}
```

那么，这些 SecurityConfigurer 又是在什么时候被使用的呢？原来，AbstractConfiguredSecurityBuilder 将构建分为了 5 个阶段，如下图所示：

![](2024-12-27-1001.png)

在 `init()` 方法中会调用每一个 SecurityConfigurer 的 `init(builder)`；在 `configure()` 方法中会调用每一个 SecurityConfigurer 的 `configure(builder)`。源码如下所示：

```java
package org.springframework.security.config.annotation;

// 省略了 import

public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
    extends AbstractSecurityBuilder<O> {

    private BuildState buildState = BuildState.UNBUILT;

    @Override
    protected final O doBuild() throws Exception {
        synchronized (this.configurers) {
            this.buildState = BuildState.INITIALIZING;
            beforeInit();
            init();
            this.buildState = BuildState.CONFIGURING;
            beforeConfigure();
            configure();
            this.buildState = BuildState.BUILDING;
            O result = performBuild();
            this.buildState = BuildState.BUILT;
            return result;
        }
    }

    protected void beforeInit() throws Exception {
    }

    protected void beforeConfigure() throws Exception {
    }

    // * 由子类实现的，具体的构建方法 *
    protected abstract O performBuild() throws Exception;

    @SuppressWarnings("unchecked")
    private void init() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.init((B) this);
        }
        // 这里暂且先忽略 configurersAddedInInitializing，留个悬念
        for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing) {
            configurer.init((B) this);
        }
    }

    @SuppressWarnings("unchecked")
    private void configure() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.configure((B) this);
        }
    }

    private Collection<SecurityConfigurer<O, B>> getConfigurers() {
        List<SecurityConfigurer<O, B>> result = new ArrayList<>();
        for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {
            result.addAll(configs);
        }
        return result;
    }

    private enum BuildState {

        UNBUILT(0),
        INITIALIZING(1),
        CONFIGURING(2),
        BUILDING(3),
        BUILT(4);

        private final int order;

        BuildState(int order) {
            this.order = order;
        }
    }
}
```

对于一个 AbstractConfiguredSecurityBuilder，它只能构建一次。如果在构建完成后，再添加 SecurityConfigurer，那是没有意义的。因此规定只能在 `CONFIGURING` 阶段之前添加 SecurityConfigurer。源码如下所示：

{% note default %}
为什么只能在 `CONFIGURING` 阶段之前添加 SecurityConfigurer，而不是 `INITIALIZING` 阶段之前呢？
{% endnote %}

```java
package org.springframework.security.config.annotation;

public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
    extends AbstractSecurityBuilder<O> {

    private final List<SecurityConfigurer<O, B>> configurersAddedInInitializing = new ArrayList<>();

    private <C extends SecurityConfigurer<O, B>> void add(C configurer) {
        Assert.notNull(configurer, "configurer cannot be null");
        Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer
            .getClass();
        // 这里和 doBuild() 竞争的是同一个对象，不允许 doBuild() 和本方法并发执行
        synchronized (this.configurers) {
            if (this.buildState.isConfigured()) {
                throw new IllegalStateException("Cannot apply " + configurer + " to already built object");
            }

            // 此时，可能是 UNBUILT 阶段，doBuild() 方法还未执行
            // 也可能是 INITIALIZING 阶段，执行 init() 方法的线程重入
            // 也就是说，在 init() 方法中仍然可以添加 SecurityConfigurer
            // 例如：一个 SecurityConfigurer 在初始化时，又需要添加另一个 SecurityConfigurer
            List<SecurityConfigurer<O, B>> configs = null;
            if (this.allowConfigurersOfSameType) {
                configs = this.configurers.get(clazz);
            }
            configs = (configs != null) ? configs : new ArrayList<>(1);
            configs.add(configurer);
            this.configurers.put(clazz, configs);

            // 由于在 SecurityConfigurer 初始化时新添加的 SecurityConfigurer 不会被遍历到
            // 因此还需要保存到 configurersAddedInInitializing
            // 注意：新添加的 SecurityConfigurer 不能在初始化时再添加另一个新的 SecurityConfigurer
            if (this.buildState.isInitializing()) {
                this.configurersAddedInInitializing.add(configurer);
            }
        }
    }

    private enum BuildState {

        public boolean isInitializing() {
            return INITIALIZING.order == this.order;
        }

        public boolean isConfigured() {
            return this.order >= CONFIGURING.order;
        }
    }
}
```

## 策略模式 与 Spring Security 的认证机制

认证（Authentication）是验证用户提供的或者存储在系统的凭证（Token），证明用户就是他们所说的人的过程。如果凭证正确，就授予对应的权限，如果不正确，就拒绝访问。凭证的形式多种多样，最常用的是用户名和密码，此外还包括生物信息凭证（存储在可信区中的指纹、面部、虹膜信息等），第三方凭证（QQ、微信扫码登录）等。

在 Spring Security 中，使用 Authentication 接口表示凭证，使用 AuthenticationProvider 接口表示认证方式。Authentication 和 AuthenticationProvider 的种类是一一对应的。由 AuthenticationManager 统一管理 AuthenticationProvider。源码如下所示：

```java
package org.springframework.security.core;

import java.io.Serializable;
import java.security.Principal;
import java.util.Collection;

import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.core.context.SecurityContextHolder;

public interface Authentication extends Principal, Serializable {

    Collection<? extends GrantedAuthority> getAuthorities();

    Object getCredentials();

    Object getDetails();

    Object getPrincipal();

    boolean isAuthenticated();

    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

```java
package org.springframework.security.authentication;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;

@FunctionalInterface
public interface AuthenticationManager {

    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

```java
package org.springframework.security.authentication;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;

public interface AuthenticationProvider {

    Authentication authenticate(Authentication authentication) throws AuthenticationException;

    boolean supports(Class<?> authentication);
}
```

## AuthenticationManager 的构建过程

AuthenticationManager 是一个安全组件，需要使用 SecurityBuilder 的方式构建，ProviderManagerBuilder 就是这样的接口。源码如下所示：

```java
package org.springframework.security.config.annotation.authentication;

import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.config.annotation.SecurityBuilder;

public interface ProviderManagerBuilder<B extends ProviderManagerBuilder<B>>
        extends SecurityBuilder<AuthenticationManager> {

    B authenticationProvider(AuthenticationProvider authenticationProvider);
}
```

支持配置式编程的接口为 AuthenticationManagerBuilder，源码如下：

```java
package org.springframework.security.config.annotation.authentication.builders;

// 省略 import

public class AuthenticationManagerBuilder
        extends AbstractConfiguredSecurityBuilder<AuthenticationManager, AuthenticationManagerBuilder>
        implements ProviderManagerBuilder<AuthenticationManagerBuilder> {

    // 基于 LDAP 的认证配置
    public LdapAuthenticationProviderConfigurer<AuthenticationManagerBuilder> ldapAuthentication() throws Exception {
        // 省略了实现
    }

    // 基于数据表的认证配置
    public JdbcUserDetailsManagerConfigurer<AuthenticationManagerBuilder> jdbcAuthentication() throws Exception {
        // 省略了实现
    }

    // 基于内存的认证配置
    public InMemoryUserDetailsManagerConfigurer<AuthenticationManagerBuilder> inMemoryAuthentication() {
        // 省略了实现
    }
}
```
