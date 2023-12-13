---
title: 关于SpringMVC @WebFilter注解的注入流程源码解析
date: 2023-12-12 10:48
categories: [源码学习]
tags: [Java, Spring, Spring源码学习]
pin: false
image: https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/1702370400268.jpg
---

> 这是一篇关于@WebFilter注解注入流程的解析，以及为什么@Order注解和@WebFilter无法搭配使用的解惑

## 前言

由于在上一篇中[《SpringBoot MVC Filter执行顺序问题排查及加载源码解析》](https://yangushan.github.io/2023/12/11/SpringBoot-Mvc-Filter%E6%89%A7%E8%A1%8C%E9%A1%BA%E5%BA%8F%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5%E5%8F%8A%E5%8A%A0%E8%BD%BD%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html)我们了解到了关于Filter的启动流程，但是发现了再使用@WebFilter到过程中，和@Order搭配使用的时候无法达到我们想要的效果，所以这篇就对@WebFilter注解进行一个分析，关于他的Spring注入流程还有为什么和@Order无法搭配的问题进行一个解惑

## 源码思路

从*图1*看到在@WebFilter的类上Tomcat的解释`jakarta.servlet.annotation.WebFilter`，没有看到什么关于@WebFilter注入的相关信息

![CleanShot 2023-12-12 at 11.27.19@2x](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2011.27.19%402x.png) *图1*

那么我们只能从哪些类调用了这个注解来看是否有什么对应的思路，从这个类调用的地方可以看到有一个类上面对应的注解，Enables scanning for Servlet components，这个`ServletComponentScan`很像是我们需要的类

![图2](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.06.23%402x.png) *图2*

我们进入这个注解进行查看，可以看到Spring关于这个注解的解释，这个类就是用来专门给@WebFilter, @WebServlet, @WebListener服务的，所以我们直接看@Import这个类就可以了`ServletComponentScanRegistrar`，*关于@Import注解的使用原理我们后面再写一篇文章来看看*

![图3](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.17.43%402x.png) *图3*

我们进入`ServletComponentScanRegistrar`类，可以看到最上面Spring的备注，这个Registrar就是用于@ServletComponentScan注解的，这个类是继承与`ImportBeanDefinitionRegistrar`的，我们可以进入这个接口源码看下![图4](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.42.44%402x.png) *图4*

可以看到`ImportBeanDefinitionRegistrar`是用来注册一些你自己定义的Bean，*PS.关于`ImportBeanDefinitionRegistrar`后面也写一个解析原理看下*

![图5](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.44.35%402x.png) *图5*

回到*图4*里面有1个继承的方法，那应该就是这个继承的方法最重要了，我们看下继承方法上的Spring注解，可以看到这个方法就是在注入我们想要的对应的Bean，默认是一个空方法。所以我们在我们`ServletComponentScanRegistrar`上的这个方法上打上断点，来查看注入流程![图6](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.47.09%402x.png) *图6*

## @WebFilter注入源码流程

启动项目打上断点之后，可以看到，首先拿到了你项目中所有休要扫描的包，我们项目只有一个；然后进行一个判断之后，进入了`addPostProcessor`方法，`org.springframework.boot.web.servlet.ServletComponentScanRegistrar#registerBeanDefinitions`

![图7](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.51.45%402x.png) *图7*

进入到`addPostProcessor`方法之后，可以看到创建了一个`ServletComponentRegisteringPostProcessorBeanDefinition`对象，而这个对象是当前类里面的一个内部对象，继承了一个`GenericBeanDefinition`，也就是一个`BeanDefinition`而已，然后就通过registry对象调用`registerBeanDefinition`方法，进入这个方法

![图8](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2014.55.09%402x.png) *图8*

进入`org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition`这个方法，beanName是在上面*图8*这个地方传进来的，是这个类写死的`servletComponentRegisteringPostProcessor`字符串，

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

  // 进行一些校验
  Assert.hasText(beanName, "Bean name must not be empty");
  Assert.notNull(beanDefinition, "BeanDefinition must not be null");

  if (beanDefinition instanceof AbstractBeanDefinition abd) {
    try {
      abd.validate();
    }
    catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
          "Validation of bean definition failed", ex);
    }
  }

  // 从beanDefinitionMap里面拿beanName，由于我们是第一次注入，所以肯定是空的，走到了空的逻辑里面
  BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
  if (existingDefinition != null) {
    if (!isAllowBeanDefinitionOverriding()) {
      throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
    }
    else if (existingDefinition.getRole() < beanDefinition.getRole()) {
      // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
      if (logger.isInfoEnabled()) {
        logger.info("Overriding user-defined bean definition for bean '" + beanName +
            "' with a framework-generated bean definition: replacing [" +
            existingDefinition + "] with [" + beanDefinition + "]");
      }
    }
    else if (!beanDefinition.equals(existingDefinition)) {
      if (logger.isDebugEnabled()) {
        logger.debug("Overriding bean definition for bean '" + beanName +
            "' with a different definition: replacing [" + existingDefinition +
            "] with [" + beanDefinition + "]");
      }
    }
    else {
      if (logger.isTraceEnabled()) {
        logger.trace("Overriding bean definition for bean '" + beanName +
            "' with an equivalent definition: replacing [" + existingDefinition +
            "] with [" + beanDefinition + "]");
      }
    }
    this.beanDefinitionMap.put(beanName, beanDefinition);
  }
  else {
    // 判断这个内是否是别名，我们这个类并不是，所以不进入
    if (isAlias(beanName)) {
      if (!isAllowBeanDefinitionOverriding()) {
        String aliasedName = canonicalName(beanName);
        if (containsBeanDefinition(aliasedName)) {  // alias for existing bean definition
          throw new BeanDefinitionOverrideException(
              beanName, beanDefinition, getBeanDefinition(aliasedName));
        }
        else {  // alias pointing to non-existing bean definition
          throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
              "Cannot register bean definition for bean '" + beanName +
              "' since there is already an alias for bean '" + aliasedName + "' bound.");
        }
      }
      else {
        removeAlias(beanName);
      }
    }
    // 可以看到下面hasBeanCreationStarted方法的代码，由于我们项目启动肯定是已经有创建其他的bean了，所以这里会返回true
    if (hasBeanCreationStarted()) {
      // Cannot modify startup-time collection elements anymore (for stable iteration)
      synchronized (this.beanDefinitionMap) {
        // 把bean对象放入我们的beanDefinitionMap集合里面
        this.beanDefinitionMap.put(beanName, beanDefinition);
        List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
        updatedDefinitions.addAll(this.beanDefinitionNames);
        updatedDefinitions.add(beanName);
        this.beanDefinitionNames = updatedDefinitions;
        removeManualSingletonName(beanName);
      }
    }
    else {
      // Still in startup registration phase
      this.beanDefinitionMap.put(beanName, beanDefinition);
      this.beanDefinitionNames.add(beanName);
      removeManualSingletonName(beanName);
    }
    this.frozenBeanDefinitionNames = null;
  }

  // existingDefinition我们是null，所以进入containsSingleton判断，由于我们还没注入这个bean对象，所以在singletonObjects对象里面也不会包含这个beanName，进入else if 判断
  if (existingDefinition != null || containsSingleton(beanName)) {
    resetBeanDefinition(beanName);
  }
  // 我们的系统并没有设置configurationFrozen这种参数，所以也不会进入
  else if (isConfigurationFrozen()) {
    clearByTypeCache();
  }
  // 结束方法
}

/**
 * Check whether this factory's bean creation phase already started,
 * i.e. whether any bean has been marked as created in the meantime.
 * @since 4.2.2
 * @see #markBeanAsCreated
 */
protected boolean hasBeanCreationStarted() {
  return !this.alreadyCreated.isEmpty();
}

@Override
public boolean containsSingleton(String beanName) {
  return this.singletonObjects.containsKey(beanName);
}

@Override
public boolean isConfigurationFrozen() {
  return this.configurationFrozen;
}
```

从上面代码结束之后，可以看到我们*图7*的`addPostProcessor`方法就这样结束了，也就是其实这块代码唯一做的就是把beanName=servletComponentRegisteringPostProcessor的放入了 *beanDefinitionMap* 就这样结束了，这显然不可能是全部。所以我们把追踪的目标类放在了`servletComponentRegisteringPostProcessor`上面

![图9](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.14.00%402x.png) *图9*

找到`org.springframework.boot.web.servlet.ServletComponentRegisteringPostProcessor`这个类，我们看到spring的源码文档，可以看到这个PostProcessor的作用就是为了扫描到servlet下的所有components的，它是实现了`BeanFactoryPostProcessor`接口的，我们进去看看

```java
/*
 * Copyright 2002-2019 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

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

这个类的文档很长，可以看到这个类的上面一大段的注视，主要讲的是`BeanFactoryPostProcessor`可以对我们的BeanDefinition进行属性修改的这么一个后置处理器，但是最有意思的是后面一段，他说

> BeanFactoryPostProcessor beans that are autodetected in an ApplicationContext will be ordered according to org.springframework.core.PriorityOrdered and org.springframework.core.Ordered semantics. In contrast, BeanFactoryPostProcessor beans that are registered programmatically with a ConfigurableApplicationContext will be applied in the order of registration; any ordering semantics expressed through implementing the PriorityOrdered or Ordered interface will be ignored for programmatically registered post-processors. Furthermore, the @Order annotation is not taken into account for BeanFactoryPostProcessor beans.
>
> 在ApplicationContext中自动检测到的BeanFactoryPostProcessor beans会根据优先级和顺序进行排序。而通过ConfigurableApplicationContext以编程方式注册的BeanFactoryPostProcessor beans则会按照它们被注册时的顺序应用。对于以编程方式注册的后处理器来说，无论是否实现了PriorityOrdered或Ordered接口定义的排序规则都会被忽略。另外，@Order注解也不适用于BeanFactoryPostProcessor beans。

**似乎这就是我们@WebFilter为什么不兼容@Order的原因了，原来是藏在了这里。**

但是为什么会这样，我们还是通过代码分析的角度来看看原因吧。回到上面的 *图9* ，由于我们`BeanFactoryPostProcessor`接口只有一个方法，所以我们直接进入`ServletComponentRegisteringPostProcessor`的`postProcessBeanFactory`方法进行断点调试

![图10](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.33.38%402x.png) *图10*

进入类中的`createComponentProvider`方法，从下图可以看出来，就是创建一个`ClassPathScanningCandidateComponentProvider`对象，进行一些属性设置

![图11](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.35.01%402x.png) *图11*

我们继续往下走，走到了scanPackage方法，可以简单的看下这段逻辑，首先是通过findCandidateComponents拿到所有扫描到的BeanDefinition，然后对每个BeanDefinition进行一个HANDLE的处理，通过*图12*我们可以看出来HANDLERS是这个类写死的![图11](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.36.09%402x.png) *图11*

![图12](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.38.50%402x.png) *图12*

我们进入`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#findCandidateComponents`这个方法查看他们是怎么扫描到我们需要的class的，进入else方法

![图13](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.55.03%402x.png) *图13*

查看`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents`方法

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
  Set<BeanDefinition> candidates = new LinkedHashSet<>();
  try {
    // 进行一个路径拼接，拿到是这样的对象classpath*:com/ejuetc/**/**/*.class
    String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
        resolveBasePackage(basePackage) + '/' + this.resourcePattern;
    // 拿到所有这个资源下的class文件，封装为Resource对象
    Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
    boolean traceEnabled = logger.isTraceEnabled();
    boolean debugEnabled = logger.isDebugEnabled();
    // 循环遍历每个Resource
    for (Resource resource : resources) {
      String filename = resource.getFilename();
      // 有些文件需要过滤
      if (filename != null && filename.contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
        // Ignore CGLIB-generated classes in the classpath
        continue;
      }
      if (traceEnabled) {
        logger.trace("Scanning " + resource);
      }
      try {
        // 解析Resource的元信息
        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
        // 这里也就是最重要的一步，判断是否符合条件的Component，如果符合了才会加入到candidates里面，进行返回
        if (isCandidateComponent(metadataReader)) {
          ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
          sbd.setSource(resource);
          if (isCandidateComponent(sbd)) {
            if (debugEnabled) {
              logger.debug("Identified candidate component class: " + resource);
            }
            candidates.add(sbd);
          }
          else {
            if (debugEnabled) {
              logger.debug("Ignored because not a concrete top-level class: " + resource);
            }
          }
        }
        else {
          if (traceEnabled) {
            logger.trace("Ignored because not matching any filter: " + resource);
          }
        }
      }
      catch (FileNotFoundException ex) {
        if (traceEnabled) {
          logger.trace("Ignored non-readable " + resource + ": " + ex.getMessage());
        }
      }
      catch (Throwable ex) {
        throw new BeanDefinitionStoreException(
            "Failed to read candidate component class: " + resource, ex);
      }
    }
  }
  catch (IOException ex) {
    throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
  }
  return candidates;
}

/**
 * Determine whether the given class does not match any exclude filter
 * and does match at least one include filter.
 * @param metadataReader the ASM ClassReader for the class
 * @return whether the class qualifies as a candidate component
 */
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
  for (TypeFilter tf : this.excludeFilters) {
    if (tf.match(metadataReader, getMetadataReaderFactory())) {
      return false;
    }
  }
  for (TypeFilter tf : this.includeFilters) {
    if (tf.match(metadataReader, getMetadataReaderFactory())) {
      return isConditionMatch(metadataReader);
    }
  }
  return false;
}
```

我们进入`isCandidateComponent`方法断点查看，可以看到在this.includeFilters里面有三个数据，也就是说，这些元信息会被我们三个Filter进行过滤，从名字可以看出来我们最关心的是`jakarta.servlet.annotation.WebFilter`这个，所以我们进入这个断点![图14](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.02.06%402x.png) *图14*

往下进入了`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#isConditionMatch`方法，然后我们进入这个shouldSip看看源码

![图15](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.09.17%402x.png) *图15*

查看`org.springframework.context.annotation.ConditionEvaluator#shouldSkip(org.springframework.core.type.AnnotatedTypeMetadata, org.springframework.context.annotation.ConfigurationCondition.ConfigurationPhase)`源码，竟然在代码第一步就返回了false，也就是只要我们的类上面没有@Conditional注解，我们就会返回false

![图16](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.11.06%402x.png) *图16*

由于 *图15* 会在判断前面增加!，所以其实返回的是True，也就是只要可以进入我们的handler判断，就会返回true。

我们回到 *图14*的那个地方，我们进入match进去看看，handler是怎么判断是否match的，进入`org.springframework.core.type.filter.AbstractTypeHierarchyTraversingFilter#match(org.springframework.core.type.classreading.MetadataReader, org.springframework.core.type.classreading.MetadataReaderFactory)`这个方法，发现进入方法第一步，直接返回true了，所以进入这个matchSelf进行查看![图17](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.15.01%402x.png) *图17*

看到`org.springframework.core.type.filter.AnnotationTypeFilter#matchSelf`源码，我们可以看到这个下图，发现这个代码逻辑非常简单，其实就是判断每个bean对象是否有使用了某个注解，而使用了@WebFIlter的注解，所以这里就会返回true了

![图18](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.19.20%402x.png) *图18*

所以最后在`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents`方法，我们会返回的candidate很少，只有使用了@WebFilter的会被读取出来

![图19](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2016.22.54%402x.png) *图19*

所以这段代码中，我们进入我们想要的那个BeanDifintion继续往下走

![图20](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.42.38%402x.png) *图20*

第一次进入handler.handle的是WebServletHandler的handle方法，由于这几个Handler都是继承于`ServletComponentHandler`这个方法又是一个父类定义的方法，所以就进入了方法`org.springframework.boot.web.servlet.ServletComponentHandler#handle`中，经过判断什么事情都没做，因为我们的是一个Filter，所以可以看出来，应该是只有在`WebFilterHandler`的时候才会有所动作，我们继续往下走，果然当是`WebFilterHandler`的时候，进入了if判断里面，我们继续看下面的流程

![图21](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.45.48%402x.png) *图21*

`org.springframework.boot.web.servlet.WebFilterHandler#doHandle`进入源码，**当我们走到这里的时候就恍然大悟了，原来@WebFilter注解其实就是创建了一个`FilterRegistrationBean`对象作为Bean对象，这也是为什么我们在上篇文档中，在注入的时候@WebFilter会被识别为`org.springframework.boot.web.servlet.ServletContextInitializer`的原因，因为`FilterRegistrationBean`的顶层就是继承与这个接口的，并且可以看到了这里并没有设置order属性，这也是为什么它创建出来的Bean的order属性是默认值的原因了**

![图22](https://raw.githubusercontent.com/Yangushan/images/main/blog/20231212/CleanShot%202023-12-12%20at%2015.46.36%402x.png) *图22*

## 最后的尝试

在前面我们看到由于`BeanFactoryPostProcessor`会直接拒绝@Order注入，但是似乎还是接口继承Ordered接口的模式，所以我们这里再次进行尝试，把代码改为继承`Ordered`接口的的方法看下，依旧还是没有生效，所以并不是因为`BeanFactoryPostProcessor`直接拒绝了@Order的注入这个原因，还是因为上面的一段逻辑，因为@WebFilter在创建`FilterRegistrationBean`对象作为Bean对象的时候，并没有设置order属性，这里才是真正的原因，看起来用@WebFilter注入的Filter是无法使用order排序的



## 总结

1. @WebFilter的注入流程是使用了`ServletComponentRegisteringPostProcessor`它是一个继承了`BeanFactoryPostProcessor`的后置处理器

2. @WebFilter在注入的时候，最终是在`WebFilterHandler`被注入的BeanDifintion，而且是被设置为了`FilterRegistrationBean`，所以这也是为什么它会被识别为`org.springframework.boot.web.servlet.ServletContextInitializer`的原因

3. 经过最后的尝试我们可以看出来@WebFilter无法使用@Order的真正原因，不是因为`BeanFactoryPostProcessor`的后置处理器导致的，而是因为在创建`BeanDefinition`的时候他们使用的`FilterRegistrationBean`并没有给他设置order属性，所以@WebFIlter这种注入Filter的模式是无法排序的，如果想要排序，可以使用下面两种方式：

   1. 第一种，使用@Component的模式来实现一个普通的Filter，这个情况下是可以使用@Order注解的

      ```java
      @Component
      @Order(1)
      public class CorsFilter implements Filter {
      ```

   2. 通过注入@Bean，创建`FilterRegistrationBean`的模式

      ```java
      @Bean
      public FilterRegistrationBean xxFilter() {
          FilterRegistrationBean<Filter> registrationBean = new FilterRegistrationBean<>();
          registrationBean.setFilter(new XxFilter());   //设置过滤器
          registrationBean.setUrlPatterns(asList("/*"));
          registrationBean.setOrder(2);  //设置优先级
          return registrationBean;
      }
      ```




关于这次Filter没有按照希望的顺序执行的问题终于告一段落，不过又有了新的疑问，@Order注解又是如何生效的？为什么`BeanFactoryPostProcessor`会导致@Order注解无法生效呢？还有另外就是关于`BeanFactoryPostProcessor`后置处理器又是如何运作的呢？@Import注解又是怎么注入的、Registrar又是怎么一个流程，又出现了好多新的疑惑啊～

--------

**封面图由bing image creator创建*
