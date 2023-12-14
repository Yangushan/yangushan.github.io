---
title: 关于Spring BeanFactoryPostProcessor注入源码解析
date: 2023-12-14 11:06
categories: [Spring源码学习]
tags: [Java, Spring, 源码学习]
pin: false
image: https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/OIG.jpeg
---

> 这是一篇关于BeanFactoryPostProcessor在Spring中的注入流程源码解析

## 前言

在上一篇[《关于SpringMVC @WebFilter注解的注入流程源码解析》](https://yangushan.github.io/2023/12/12/%E5%85%B3%E4%BA%8ESpringMVC-@WebFilter%E6%B3%A8%E8%A7%A3%E7%9A%84%E6%B3%A8%E5%85%A5%E6%B5%81%E7%A8%8B%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html) 中我们分析了`@WebFilter`是如何注入到Spring容器中的这么一个过程，在分析过程中发现使用了一个`BeanFactoryPostProcessor`这样的一个Spring的类，就开始对`BeanFactoryPostProcessor`在Spring中是如何生效以及他的具体作用产生了疑问🤔。



## 关于BeanFactoryPostProcessor

```java
/**
 * Factory hook that allows for custom modification of an application context's
 * bean definitions, adapting the bean property values of the context's underlying
 * bean factory.
 *
 * <p>Useful for custom config files targeted at system administrators that
 * override bean properties configured in the application context. See
 * {@link PropertyResourceConfigurer} and its concrete implementations for
 * out-of-the-box solutions that address such configuration needs.
 *
 * <p>A {@code BeanFactoryPostProcessor} may interact with and modify bean
 * definitions, but never bean instances. Doing so may cause premature bean
 * instantiation, violating the container and causing unintended side effects.
 * If bean instance interaction is required, consider implementing
 * {@link BeanPostProcessor} instead.
 *
 * <h3>Registration</h3>
 * <p>An {@code ApplicationContext} auto-detects {@code BeanFactoryPostProcessor}
 * beans in its bean definitions and applies them before any other beans get created.
 * A {@code BeanFactoryPostProcessor} may also be registered programmatically
 * with a {@code ConfigurableApplicationContext}.
 *
 * <h3>Ordering</h3>
 * <p>{@code BeanFactoryPostProcessor} beans that are autodetected in an
 * {@code ApplicationContext} will be ordered according to
 * {@link org.springframework.core.PriorityOrdered} and
 * {@link org.springframework.core.Ordered} semantics. In contrast,
 * {@code BeanFactoryPostProcessor} beans that are registered programmatically
 * with a {@code ConfigurableApplicationContext} will be applied in the order of
 * registration; any ordering semantics expressed through implementing the
 * {@code PriorityOrdered} or {@code Ordered} interface will be ignored for
 * programmatically registered post-processors. Furthermore, the
 * {@link org.springframework.core.annotation.Order @Order} annotation is not
 * taken into account for {@code BeanFactoryPostProcessor} beans.
 *
 * @author Juergen Hoeller
 * @author Sam Brannen
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

上面是`BeanFactoryPostProcessor`的官方源码和文档，从上面Spring给这个类的文档来分析这个类的作用

> Factory hook that allows for custom modification of an application context's bean definitions, adapting the bean property values of the context's underlying bean factory.
>
> 工厂钩子是一种机制，它允许我们自定义修改应用程序上下文中的bean定义，并且可以调整上下文底层的bean工厂中的属性值。
>
> A BeanFactoryPostProcessor may interact with and modify bean definitions, but never bean instances. Doing so may cause premature bean instantiation, violating the container and causing unintended side effects. If bean instance interaction is required, consider implementing BeanPostProcessor instead.
>
> BeanFactoryPostProcessor可以对bean定义进行操作和修改，但不能对bean实例进行操作。这样做可能会导致在合适的时机之前就创建了bean实例，从而违反了容器的规则，并产生了意想不到的影响。如果需要对bean实例进行操作，请考虑使用BeanPostProcessor来完成。

也就是在我们定义好了`BeanDefinition`之后，我们可以通过BeanFactory钩子来进行修改。这个类就是专门服务于BeanDefinition的，如果想要修改我们的bean对象，推荐使用`BeanPostProcessor`。

> Registration
> An ApplicationContext auto-detects BeanFactoryPostProcessor beans in its bean definitions and applies them before any other beans get created. A BeanFactoryPostProcessor may also be registered programmatically with a ConfigurableApplicationContext.
>
> 注册流程
>
> ApplicationContext在其bean定义中自动检测BeanFactoryPostProcessor bean，并在创建任何其他bean之前应用它们。可以通过ConfigurableApplicationContext以编程方式注册BeanFactoryPostProcessor。
>

这里可以看到Spring对这个类的注册流程有一个简单的描述，也就是`BeanFactoryPostProcessor`的bean是优先于普通bean之前被应用的。

> Ordering
> BeanFactoryPostProcessor beans that are autodetected in an ApplicationContext will be ordered according to org.springframework.core.PriorityOrdered and org.springframework.core.Ordered semantics. In contrast, BeanFactoryPostProcessor beans that are registered programmatically with a ConfigurableApplicationContext will be applied in the order of registration; any ordering semantics expressed through implementing the PriorityOrdered or Ordered interface will be ignored for programmatically registered post-processors. Furthermore, the @Order annotation is not taken into account for BeanFactoryPostProcessor beans.
>
> 排序规则
>
> 在ApplicationContext中自动检测到的BeanFactoryPostProcessor beans会根据org.springframework.core.PriorityOrdered和org.springframework.core.Ordered规则进行排序。然而，在通过ConfigurableApplicationContext以编程方式注册的情况下，这些bean将按照注册顺序应用；对于通过编程方式注册的后处理器来说，忽略了实现PriorityOrdered或Ordered接口所定义的排序规则。另外，请注意@Order注解不适用于BeanFactoryPostProcessor beans。
>

从这里可以看到如果我们实现了好几个`BeanFactoryPostProcessor`，想要他们的执行顺序按照我们的方式来排序执行，并不能通过@Order注解，而应该使用其他两种方式，并且在`ConfigurableApplicationContext`编程模式下排序将会失效。

关于唯一的方法`postProcessBeanFactory`的文档描述：

> Modify the application context's internal bean factory after its standard initialization. All bean definitions will have been loaded, but no beans will have been instantiated yet. This allows for overriding or adding properties even to eager-initializing beans.
>
> 在标准初始化之后，修改应用程序上下文的内部Bean工厂。此时已经加载了所有Bean定义，但还没有实例化任何Bean。这样可以即使对于急切初始化的Bean也能进行属性重写或添加操作。
>
> 所以我们可以看到对于我们的BeanFactoryPostProcessor是在所有Bean实例化之前调用的

可以从Spring的这些文档和源码总结一下关于`BeanFactoryPostProcessor`：

1. `BeanFactoryPostProcessor`是给`BeanDefintion`定义的一个后置处理钩子，也就是允许我们修改我们一些无法修改的`BeanDefinition`属性，这对于扩展一些第三方这种bean应该是比较有帮助的
2. `BeanFactoryPostProcessor`不应该用来直接操作Bean对象，如果想要操作Bean对象应该使用`BeanPostProcessor`这个类(这个类的注入后期再写一篇文章～)
3. `BeanFactoryPostProcessor`会在创建其他Bean之前提前应用他们
4. 关于`BeanFactoryPostProcessor` bean的排序，他们不支持@Order，并且如果使用了`ConfigurableApplicationContext`编程方式注入的`BeanFactoryPostProcessor` bean任何排序都会失效。
5. `BeanFactoryPostProcessor`只有一个方法`postProcessBeanFactory`，这个方法Spring加载了所有的BeanDefinition之后，但是这个时候还没有实例话任何Bean

## 源码思路

我们从`BeanFactoryPostProcessor`唯一的方法入手，看看调用方，发现只有一个地方调用了，其他地方都只是doc备注，所以进入唯一的那个方法![图1](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2013.55.23%402x.png) *图1*

`org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(java.util.Collection<? extends org.springframework.beans.factory.config.BeanFactoryPostProcessor>, org.springframework.beans.factory.config.ConfigurableListableBeanFactory)`，这个方法很清晰，看起来就是拿到所有的`BeanFactoryPostProcessor`循环调用每一个的的`postProcessBeanFactory`方法，继续往上看，谁调用了这个方法![图2](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2013.56.31%402x.png) *图2*

发现最终只有一个方法中调用了这个方法`org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)`，其实还是在这个类里，是一个同名的重载方法，这个方法内容比较多，先找调用方![图3](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2013.57.46%402x.png) *图3*

继续往上找，走到了`org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)`

![图4](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2013.59.52%402x.png) *图4*

看到这个方法比较简单，就是直接调用我们 *图3* 的静态方法，然后给beanFactory设置一点属性，继续找这个方法被调用的地方![图5](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.00.55%402x.png) *图5*

第一个类特别熟悉，我们从第一个类里面点进去查看，看到了熟悉的地方，在`org.springframework.context.support.AbstractApplicationContext#refresh`方法里面，也就是在启动Spring容器的方法里面，调用了这个方法。![图6](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.03.16%402x.png) *图6*

而从第二个点进去，这个方法并不熟悉，所以显然不是我们要的，那就直接看上面的refresh方法

![图7](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.05.02%402x.png) *图7*

## 从refresh开始调试

为了更好的测试`BeanFactoryPostProcessor`的效果，并且对上面的那块关于Ordering注解也比较感兴趣，所以在项目中自己创建了两个`BeanFactoryPostProcessor`，代码如下

```java
@Component
public class Test1BeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("Test1111111111BeanFactoryPostProcessor do something");
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
@Component
public class Test2BeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("Test222222222BeanFactoryPostProcessor do something");
    }

    @Override
    public int getOrder() {
        return 1;
    }
}
```

我创建了两个类，并且给他们继承了Ordered接口，Test1的顺序高于Test2加上了答应日志，如果按照Spring的流程来说，那么我们的Test1的日志应该会先打印。这里先放下这段代码，进行项目启动。

由于refresh方法内容比较多，我们这次只关注在`BeanFactoryPostProcessor`这块，当refresh方法走到`postProcessBeanFactory`这个方法的时候，在singletonObjects（这个对象是存放Spring容器中的所有单例对象）里面已经有一些对象了，不过并没有我们的`BeanFactoryPostProcessor`对象，所以继续往下![图8](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.18.13%402x.png) *图8*

当我们执行完`invokeBeanFactoryPostProcessors`方法的时候，我们发现了在beanDefinitionMap(存放所有BeanDefinition的一个map)里面已经把所有的BeanDefinition都加载完了。

![图9](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.26.41%402x.png) *图9*

并且在singletonObjects里面又多了一些对象，其中包括了我们上期中看到的`ServletComponentRegisteringPostProcessor`也就是我们的`BeanFactoryPostProcessor`的beans也是在这个类初始化的，所以我们核心就是要看`invokeBeanFactoryPostProcessors`方法了

![图10](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.27.49%402x.png) *图10*

也就是说在这个方法中，创建了我们所有的BeanDefinition，并且初始化了我们的`BeanFactoryPostProcessor`的bean对象，还执行了他们所有的`postProcessBeanFactory`方法。

## invokeBeanFactoryPostProcessors方法

我们断点进入这个方法，方法中的第一段代码，这个方法调用了另外一个方法给了两个参数，另外一个参数是通过调用其他方法获取的，我们要先看这个

![图11](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.34.54%402x.png) *图11*

进入`org.springframework.context.support.AbstractApplicationContext#getBeanFactoryPostProcessors`方法可以看到这个方法比较简单，就直接返回了，从 *图13*这个属性的备注来看，应该是用来存放所有BeanFactoryPostProcessor的，但是这个时候只返回了3个看起来像是Spring默认的PostProcessor还并没有我们自己的，所以我们继续往下走

![图12](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.38.06%402x.png) *图12*

![图13](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.38.53%402x.png) *图13*

回到 *图11*我们进入`org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(org.springframework.beans.factory.config.ConfigurableListableBeanFactory, java.util.List<org.springframework.beans.factory.config.BeanFactoryPostProcessor>)`方法，通过一步步调试，来观察我们的beanFactory里面的数据发现，当我们走完`invokeBeanDefinitionRegistryPostProcessors`方法之后，已经加载了我们的BeanDefinition了，不过观察singletonObjects我们的`BeanFactoryPostProcessor`还没被实例话，观察这方法的2个参数，registry从上面的代码看起来就是我们的`DefaultListableBeanFactory`，而另外一个currentRegistryProcessors list里面只有一个对象，就是`ConfigurationClassPostProcessor`是一个继承了接口`BeanDefinitionRegistryPostProcessor`的对象

![图14](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2014.49.43%402x.png) *图14*

关于`BeanDefinitionRegistryPostProcessor`的描述，我们可以看到这个类的作用就是用来加载BeanDefinition的，所以我们的`ConfigurationClassPostProcessor`就是用来加载我们自己定义的那些常规的`BeanDefintion`

```java
/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```

回到代码上当我们往下看那些源码备注之后，我们就能看懂流程了，为什么Spring后续还需要继续拿到`BeanDefinitionRegistryPostProcessor`继续操作，因为`BeanDefinitionRegistryPostProcessor`是可以注册`BeanDefinition`的，所以可能在扫描完`BeanDefinitionRegistryPostProcessor`的那些bean之后，可能又会有新的`BeanDefinitionRegistryPostProcessor`需要操作，所以最后有一个while循环，直到所有的`BeanDefinitionRegistryPostProcessor`加载完之后才会结束

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  // WARNING: Although it may appear that the body of this method can be easily
  // refactored to avoid the use of multiple loops and multiple lists, the use
  // of multiple lists and multiple passes over the names of processors is
  // intentional. We must ensure that we honor the contracts for PriorityOrdered
  // and Ordered processors. Specifically, we must NOT cause processors to be
  // instantiated (via getBean() invocations) or registered in the ApplicationContext
  // in the wrong order.
  //
  // Before submitting a pull request (PR) to change this method, please review the
  // list of all declined PRs involving changes to PostProcessorRegistrationDelegate
  // to ensure that your proposal does not result in a breaking change:
  // https://github.com/spring-projects/spring-framework/issues?q=PostProcessorRegistrationDelegate+is%3Aclosed+label%3A%22status%3A+declined%22

  // Invoke BeanDefinitionRegistryPostProcessors first, if any.
  Set<String> processedBeans = new HashSet<>();

  // 我们的beanFactory=DefaultListableBeanFactory，所以进入这个if条件
  if (beanFactory instanceof BeanDefinitionRegistry registry) {
    List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
    List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

    // 这个beanFactoryPostProcessors是从外面传进来的3个
    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
      if (postProcessor instanceof BeanDefinitionRegistryPostProcessor registryProcessor) {
        registryProcessor.postProcessBeanDefinitionRegistry(registry);
        registryProcessors.add(registryProcessor);
      }
      else {
        regularPostProcessors.add(postProcessor);
      }
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // Separate between BeanDefinitionRegistryPostProcessors that implement
    // PriorityOrdered, Ordered, and the rest.
    List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

    // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
    // 由于Spring在之前已经注册好了一些bean，所以这里可以拿到postProcessorNames=org.springframework.context.annotation.internalConfigurationAnnotationProcessor
    String[] postProcessorNames =
        beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        // 在这里getBean会拿到ConfigurationClassPostProcessor对象，实现了BeanDefinitionRegistryPostProcessor接口
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    // 这个方法里面ConfigurationClassPostProcessor会加载BeanDinfition
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
    currentRegistryProcessors.clear();

    // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
    // 再次操作BeanDefinitionRegistryPostProcessors
    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
    for (String ppName : postProcessorNames) {
      if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
        processedBeans.add(ppName);
      }
    }
    sortPostProcessors(currentRegistryProcessors, beanFactory);
    registryProcessors.addAll(currentRegistryProcessors);
    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
    currentRegistryProcessors.clear();

    // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
    // 再次操作BeanDefinitionRegistryPostProcessors，直到没有东西可以注册为止
    boolean reiterate = true;
    while (reiterate) {
      reiterate = false;
      postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
      for (String ppName : postProcessorNames) {
        if (!processedBeans.contains(ppName)) {
          currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
          processedBeans.add(ppName);
          reiterate = true;
        }
      }
      sortPostProcessors(currentRegistryProcessors, beanFactory);
      registryProcessors.addAll(currentRegistryProcessors);
      invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());
      currentRegistryProcessors.clear();
    }

    // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
    invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
  }

  else {
    // Invoke factory processors registered with the context instance.
    invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
  }

  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let the bean factory post-processors apply to them!
  // 拿到所有的BeanFactoryPostProcessor
  String[] postProcessorNames =
      beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

  // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
      // skip - already processed in first phase above
    }
    else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      // 从beanFactory拿bean，没有则创建bean，也就是在这里创建了我们的BeanFactoryPostProcessor
      priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
  // 优先执行priorityOrderedPostProcessors的数据
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

  // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
  // 执行实现了orderedPostProcessors的数据
  List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
  for (String postProcessorName : orderedPostProcessorNames) {
    // 从beanFactory拿bean，没有则创建bean，也就是在这里创建了我们的BeanFactoryPostProcessor
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  // 按照ordered接口
  sortPostProcessors(orderedPostProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

  // Finally, invoke all other BeanFactoryPostProcessors.
  // 执行没有排序的BeanFactoryPostProcessor
  List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
  for (String postProcessorName : nonOrderedPostProcessorNames) {
    // 从beanFactory拿bean，没有则创建bean，也就是在这里创建了我们的BeanFactoryPostProcessor
    nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
  }
  invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

  // Clear cached merged bean definitions since the post-processors might have
  // modified the original metadata, e.g. replacing placeholders in values...
  beanFactory.clearMetadataCache();
}
```

当我们走到最后的时候，看到和我们`BeanFactoryPostProcessor`相关的代码我们就了解了，为什么@Order注解无法生效，因为Spring这里压根没有处理@Order注解，把他们归为没有排序的那一类了，在分类的时候分为了3类list，最先执行`priorityOrderedPostProcessors`，看到这里竟然发现spring源码也有简单的一面啊。顺带一提beanFactory.getBean方法又是一个比较长的流程，后面重新开坑写一篇吧，这里就不多说了

当我们走完Ordered接口的那个流程之后，我们前面写下的两个类的打印也顺利在控制台输出了![图15](https://yangushan-image.oss-cn-shanghai.aliyuncs.com/blog/20231214/CleanShot%202023-12-14%20at%2015.21.11%402x.png) *图15*

## 总结

1. `BeanFactoryPostProcessor`是一个可以用来后置修改我们BeanDefinition的钩子，在加载完所有BeanDefinition之后，会执行他的方法`postProcessBeanFactory`，并且是在bean初始化之前
2. `BeanFactoryPostProcessor`的bean支持排序，优先使用priority排序，然后再使用ordered接口排序，并不支持@Order排序
3. 从上面源码我们还可以学习到，我们想要再Spring中注入我们自己的BeanDefinition可以通过`BeanDefinitionRegistryPostProcessor`这个钩子进行扩展
4. 还有在调试过程中发现，我们普通的Bean的BeanDefinition都是通过`ConfigurationClassPostProcessor`这个`BeanDefinitionRegistryPostProcessor`加载出来的，下次可以进入这个类的`PostProcessBeanDefinitionRegistry`再次进行扩展学习



这次的源码学习又有了额外的收获，不过又对其他代码产生了兴趣`ConfigurationClassPostProcessor`是怎么加载到我们普通的Bean的BeanDefintion的😂。真的是代码看到哪里学习到哪里。

--------

**封面图由bing image creator创建*
