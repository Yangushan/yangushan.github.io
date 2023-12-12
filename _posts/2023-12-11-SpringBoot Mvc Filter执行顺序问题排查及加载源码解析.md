---
title: SpringBoot MVC Filter执行顺序问题排查及加载源码解析
date: 2023-12-11 16:03
categories: [源码学习]
tags: [Java, Spring, Spring源码学习]
pin: false
image: https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/20231211161239.png
---

> 这篇文章是记录一次关于SpringBoot中MVC Filter没有按照我想要的顺序执行的问题排查，以及关于后续Filter加载顺序的源码解析

## 前言

最近有个项目，让我帮忙解决跨域的问题于是我就写了一个filter，这本来是一个很简单的事情，但是前端调用依旧有跨域问题，经过代码调试之后发现有另外一个同事写的用户拦截filter，顺序比我的高，原来在他们的filter上增加了@order注解

然后我也在我的filter上新增了@order注解，启动之后发现顺序正常了，**但是后来我想起来了order的顺序是倒序的**，也就是他的-1应该比我的1大，但是我的filter却先执行了，让我有点觉得自己记忆错乱，上网就搜索了一番确实是倒序，也就是-1比1大才是，并且order注解的最大数是一个integer最大负数

所以为什么他的filter没有生效呢，也让我产生了疑惑，到底Springmvc的filter是如何加载的，又是如何判断这个顺序的，让我们从源码来查看这个问题

## 源码思路

由于不知道filter的启动方式，所以我们选择打个断点，在我们的filter上，进行request的filter断点拦截，判断filter是从什么地方开始调用过来的

![图0](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211153943388.png) *图0*

从断点可以看出来，后面大部分其实是在递归调用不同的filter，所以可以忽略，我们直接从filter上面的invoke方法点进去查看org.apache.catalina.core.StandardWrapperValve#invoke

![图1](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154111268.png) *图1*

从这里的filterChain的备注可以看出来，每次请求的时候，会创建一个filter chain也就是filter链，来进行过滤，那么我们去看看这个FilterChain.doFilter方法

![图2](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154439581.png) *图2*

可以看到上面没做什么，去看看核心的`internalDoFilter`

![图3](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154545921.png) *图3*

从图3可以看到，每次都是从Filters数组里面拿一个Filter去慢慢执行，也就是filters index=0会先执行，因为Pos++，那么我们看看这个filters，从下图可以看出来就是一个`ApplicationFilterConfig`数组，但是从上面源码可以看出来，这个对象里面包含了我们需要的filter，所以我们就是要看filter是如何插入数据的

![Untitled](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154606862.png)

我们查看图4这个filters调用的相关方法，只有哪个system.arraycopy是往里面插入数据的，那我们继续进入这个方法查看

![图4](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154620792.png) *图4*

![图5](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154632567.png) *图5*

从图5可以看出来这个方法的逻辑也是比较简单的，那我们继续查看这个方法被调用的地方

![图6](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154702494.png) *图6*

从图6看出来只有2个地方调用了这个方法，我们进入这两个调用的地方，发现外面其实是同一个方法调用org.apache.catalina.core.ApplicationFilterFactory#createFilterChain，这个方法就比较熟悉了，也就是我们在*图1*中，通过这个方法创建了filter chain，才有后续，所以这个方法就是我们要找的地方，从下图中可以看到其实是通过循环filterMaps，来添加filterConfig，那么我们往上查看这个filterMaps是什么

![图7](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211154945304.png) *图7*

从图7代码上面一部分的代码可以看到这个filterMaps是从StandardContext上下文中拿到的，我们进入这个findFilterMaps的方法看下

![图8（图8和图7是一个方法，只是代码太多，分开截图，图8在图7上面一点）](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155006372.png)

*图8（图8和图7是一个方法，只是代码太多，分开截图，图8实际代码在图7代码上面一点）*

方法比较简单，那么我们继续查看这个filterMaps

![图9](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155136087.png) *图9*

从spring mvc的备注来看，这个filterMaps就是存放所有filter的地方，可以看到这个这个备注，是一个有序的map，那么我们要看filter的执行顺序和如何增加我们的filter，其实就是看这个filterMaps是如何插入的就可以了，我们继续查看这个filterMaps被插入的地方

![图10](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155155439.png) *图10*

可以看到一个方法是正常插入，一个是插入到前面，所以，我们查看这两个方法

![图11](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155208435.png) *图11*

可以从上面两个方法上看出来，一个是插入到末尾，一个是插入到头部

![图12](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155225885.png) *图12*

![图13](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155243528.png) *图13（这个是图12里面filterMaps分别调用add和addBefore的源码)*

由于图12的两个方法调用位置比较多，难以判断源码，所以我们采取打断点的方式来看，这段代码是什么地方调用了，重启项目

## SpringMVC启动注册filter流程

![图14](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155303174.png) *图14*

**由于上面我们在两个方法打上了断点，之后发现启动项目之后在注入bean的时候就调用这些方法了**，第一个进来的是addFilterMapBefore方法，我们根据调用栈一步步来查找我们需要的位置

org.apache.catalina.core.ApplicationFilterRegistration#addMappingForUrlPatterns，这里可以看出来，是创建了filterMap的地方，所以继续往上看

![图15](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155321913.png) *图15*

org.springframework.boot.web.servlet.AbstractFilterRegistrationBean#configure，从这段源码可以看出来`FilterRegistration.Dynamic registration` 主要还是外面一层的代码

![图16](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155421694.png) *图16*

继续往上org.springframework.boot.web.servlet.DynamicRegistrationBean#register，可以看到，这里拿到了注册对象也就是上面的`registration`，所以还要继续

![图17](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155450009.png) *图17*

继续往上org.springframework.boot.web.servlet.RegistrationBean#onStartup，这个onStartup方法还是不够，还是在注册单一对象

![图18](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155512151.png) *图18*

org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#selfInitialize，看到这个代码，终于找到位置了，也就是这里通过调用了`getServletContextInitializerBeans`方法，拿到所有的bean对象进行循环注册的，那么我们就去看看这个方法的源码

![图19](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155529429.png) *图19*

可以看到这个备注，就是我们想要的东西，可以找到所有的servlet和filter，eventListener，那么我们进入这个构造方法继续查看

![图20](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155549430.png) *图20*

可以看到其实上面一个方法是构造了一个`ServletContextInitializerBeans`对象，这个对象继承了`AbstractCollection`，所以是一个collection对象，其实从这个对象的备注也可以看出来，这个对象就是一个列表结构对象，里面包含了所有的servlet, filter等等，所以我们继续查看它的那个构造方法

![图21](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155603156.png) *图21*

从构造方法来看里面就是在实例化数据的，从*图23*可以知道这里面最重要的是sortedList，是一个排好序的`ServletContextInitializer`对象列表，所以我们只要观察这个对象就可以了，从构造方法看出来这个对象是用的`initializerTypes`去排序的，并且排序方式用的是`AnnotationAwareOrderComparator *INSTANCE*`

![图22](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155613665.png) *图22*

![图23](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155624530.png) *图23*

我们查看`AnnotationAwareOrderComparator *INSTANCE`这个类，可以看出来了这个东西的排序方式，用的@Order注解，或者实现了order interface，或者就是使用了Priority，这个不关心，我们只是知道order注解在这里使用上了，所以按道理说我同事写的那个order应该会生效，但是却没有生效，问题依然存在*

![图24](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155652898.png) *图24*

我们继续回到*图22*的流程上，查看`initializerTypes`这个对象上，从源码看起来就是在这两段代码中，进行的设置，看不到其他的流程了，所以我们一个个对两个源码进行分析

![图25](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155701556.png) *图25*

先进入org.springframework.boot.web.servlet.ServletContextInitializerBeans#addServletContextInitializerBeans，可以看到下面的逻辑，确实是有添加的动作，但是是个for循环，所以我们先从`getOrderedBeansOfType`方法入手，

![图26](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155709616.png) *图26*

这段代码比较简单可以看出来，是从spring bean工厂拿到对应的type，然后进行Order的排序之后，返回这个列表，由于直接看代码不好理解，我们改为使用调试代码的方式，更加清晰

![图27](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155724942.png) *图27*

再一次重启项目，在这个`addServletContextInitializerBeans`方法上打断点，可以看到此时的堆栈信息，这个方法给的type是interface org.springframework.boot.web.servlet.ServletContextInitializer，我们看看拿到了什么

![图28](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155736539.png) *图28*

jsonTypeInfoFilter和UserAuthFilter是我同事写的，而我自己写的CorsFilter却没有拿到，因为我同事写的jsonTypeInfoFilter是用的通过注入`FilterRegistrationBean`这个bean的方式（这个bean最上层的类实现了ServletContextInitializer接口），UserAuthFilter用的springmvc提供的@WebFilter的模式注入的filter（至于这个@WebFilter为什么属于这个后面再写一篇文章解释）；这两种方式是属于interface org.springframework.boot.web.servlet.ServletContextInitializer的方式，而我自己写的CorsFilter，是通过的@Component继承普通filter的方式，所以在这里不会读出来

![图29](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155802475.png) *图29*

我们在方法的最后出return beans打上断点，可以看到我同事写的@order竟然没有注入进去，也就是通过@WebFilter实例化出来的ServletContextInitializer bean,竟然不承认@order注解，但是用另一个方法图31，create `FilterRegistrationBean` 设置的Order是承认的，原因在我写的另一篇关于@WebFilter注解里面有解释，这里先写结论：`@WebFilter注入一个Filter的时候是无法搭配@Order使用的。`

我们可以看一下@Order注解的源码解释它是只服务于普通的bean，也就是@Component和@Bean产生的这种Bean，@WebFilter不属于这类，所以@Order不生效（图32），后面会出一篇关于@WebFilter解析的Blog.

![图30](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155814030.png) *图30*

![图31](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155826077.png) *图31*

![图32](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155841013.png) *图32*

回到*图26*的流程，看完`getOrderedBeansOfType`，返回一个bean list进行循环调用`addServletContextInitializerBean`方法，我们继续查看这个方法，可以看到下图，通过对不同的bean实例判断，我们进入filter的分支，然后通过调用`addServletContextInitializerBean`方法插入到`initializers`里面，这里这个方法就可以告一段落了。

![图33](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155853815.png) *图33*

![图34](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155909788.png) *图34*

**由于上面的@WebFilter的order没有生效，所以我增加了@Component注解之后，再一次重启进行调试了，这里要注意**

但是我们的CorsFilter还没设置进去，所以这个时候继续往刚刚*图25*的另外一个方法`addAdaptableBeans进去查看流程`，从这里的代码看出来，我们核心需要的应该是查看第三行，进入第三行断点，给filter.class的情况

![图35](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155924787.png) *图35*

可以看到，这里又一次调用了`getOrderedBeansOfType`方法，我们再次进入这个方法看看

![图36](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155934792.png) *图36*

从下图查看堆栈信息，我们这里可以看到，这次给的type是interface jakarta.servlet.Filter，所以继承了filter接口的都被加载出来了，而且可以看到随着sort之后，也开始正常排序了，但是这里有一个问题了，userAuthFilter在上面那个interface被加载了一次，在这个方法filter interface也被加载了一次，也就是一个类其实创建了两个bean**（这也是为什么如果同时使用@Component和@WebFilter，这个时候不要设置WebFilter的filterName，因为这个filterName会被作为beanName，要么就起的不一样，否则启动会报错，会提示有两个重命名的bean），但是filter被创建了两个bean难道一个filter会被过滤两次吗？留着这个疑问继续往下走**

![图37](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211155959417.png) *图37*

我们继续回到图36这个方法，对放回的bean进行循环加入seen set列表，我们进行断点排查，此时此刻确实出现了两个UserAuthFilter对象

![图38](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211160012804.png) *图38*

![图39](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211160021909.png) *图39*

继续往下走，看到这个sortedList确实有两个UserAuthFilter

![图40](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231211/image-20231211160032028.png) *图40*

最后走到`addFilterMapBefore`，确实添加了两次。。。并且在这个filter类上打断点，一个filter确实被执行了两次，这显然是不合理的，所以显然在使用@WebFilter的时候不应该使用@Order，因为如果要使用@Order就要增加@Component注解使其生效，但是会导致执行两次的问题，不过这里又产生了一个疑惑，到底@WebFilter是如何被加载到spring bean容器里的，还有为什么@WebFilter的@Order不生效?

## 总结

1. @WebFilter注解如果添加@Order注解并不会生效，这也是为什么我1比-1先执行的原因，因为根本没生效
2. 使用@WebFilter注解的时候如果想要@Order注解生效，就要添加一下@Component注解，但是这会导致一个filter被两个interface注入，导致生成两个bean，最后被加入到filterMaps里面这个filter会被设置两次，次方案不赞成使用，因为一个filter在一次请求中会被调用两次
3. @Order注解并不是对所有的类都生效，必须是spring都普通bean，比如@Component等等
4. `FilterRegistrationBean`的顶层是interface org.springframework.boot.web.servlet.ServletContextInitializer，而不是interface jakarta.servlet.Filter
5. @WebFilter最后被初始化到bean里面也会被识别为类interface org.springframework.boot.web.servlet.ServletContextInitializer，和`FilterRegistrationBean`是同一个

这些源码跨越了tomcat, springboot, spring mvc, spring core等等，并且有些是在request请求的时候注入的，而有些又是在项目启动的时候注入的，如果排查一个底层源码问题，不熟悉确实就要像我一样可能需要花费1，2个小时才能查到源头，不过此次虽然是一个小BUG引起的，但是收获满满，至少对springmvc filter启动有了一个很不错的了解，等待下一篇继续了解@WebFilter的注册机制

--------

**封面图由bing image creator创建*