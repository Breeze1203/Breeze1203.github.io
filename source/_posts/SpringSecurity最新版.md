---
title: "最新版"
date: 2025-06-11 19:23:38
categories: SpringSecurity
tags: SpringSecurity
---

# SpringSecurity

项目源码:<a href="https://github.com/eazybytes/springsecurity6" rel="noopener noreferrer nofollow" target="_blank">https://github.com/eazybytes/springsecurity6</a>

### 入门：基本配置

springboot导入依赖springsecurity依赖

``` xml
<dependency>           <groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-security</artifactId></dependency>

<dependency>.   <groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

``` java
package com.eazybytes.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class ProjectSecurityConfig {

    @Bean
    SecurityFilterChain  defaultSecurityFilterChain(HttpSecurity http) throws Exception {


        http.authorizeHttpRequests((requests) -> requests
                        .requestMatchers("/myAccount","/myBalance","/myLoans","/myCards").authenticated()
                        .requestMatchers("/notices","/contact").permitAll())
                .formLogin(Customizer.withDefaults())
                .httpBasic(Customizer.withDefaults());
        return http.build();
    }

}
```

可以看到/myAccount","/myBalance","/myLoans","/myCards"这些路径需要认证之后我们才能访问

这两个路径无需认证"/notices","/contact"

当我们访问需要认证的路径时候，他会自动跳转到登录页，登录之后才可以访问

<img src="/upload/截屏2024-04-02%2014.04.50.png" style="display: inline-block;width:100.0%;height:100.0%" />

关于这里的用户名，可以在配置文件设置，如下：

``` xml
spring.security.user.name = eazybytes
spring.security.user.password = 12345
```

不配置的话，会在控制台打印输出

但我们日常生活中，都是通过查询数据库来获取用户，下面进行讲解利用数据源进行登录授权

### 认证：基于数据源

1.实现UserDetailsService接口，重写loadUserByUsername方法

<img src="/upload/截屏2024-04-02%2016.13.57.png" style="display: inline-block;width:100.0%;height:100.0%" />可以看到有一个username参数，这个参数正是前面login登录也传过来的用户名，我们根据这个参数调用service层，查看是否有这个用户，如果没有这个用户，直接抛出UserNameNotFoundException，springsecurity会根据这个异常进行后续处理，如果用户存在，我们将查出来的用户名，密码及用户角色传给User对象，后续进行这里的password和登录界面输入的password比对,进行后续处理

2.springsecurity还可以对密码进行加密处理，只需要在springboot容器中注入对应的对象

<img src="/upload/截屏2024-04-02%2016.21.13.png" style="display: inline-block;width:100.0%;height:100.0%" />

可以看到我这里注入的对象是没有进行加密处理的，返回的是一个PasswordEncoder接口的对象

<img src="/upload/截屏2024-04-02%2016.23.54.png" style="display: inline-block;width:100.0%;height:100.0%" /><img src="/upload/截屏2024-04-02%2016.23.39.png" style="display: inline-block;width:100.0%;height:100.0%" />在这里可以看到很多进行密码加密实现类，我们可以注入对应算法的密码加密方法,当然，我们也可以定义自己的密码加密方法，只需要实现这个接口，并将其注入到spring容器中；

首先定义一个累实现这个接口，重写里面的方法，三个方法的作用，你可以询问gpt，下面是我询问的结果

<img src="/upload/截屏2024-04-02%2016.31.27.png" style="display: inline-block;width:100.0%;height:100.0%" />可以看到，我们只需要关注第二个方法,这个方法传递过来两个参数，第一个参数是前端页面用户输入的，第二个是loadUserByName查询出来的用户密码，我们可以自己进行比对，返回true，则用户登录成功，false则登录失败。

### 认证：基于内存

上述是进行数据源进行用户认证的，我们也可以通过基于内存认证,spring容器中注入InMemoryUserDetailsManager对象

<img src="/upload/截屏2024-04-02%2016.42.31.png" style="display: inline-block;width:100.0%;height:100.0%" />
