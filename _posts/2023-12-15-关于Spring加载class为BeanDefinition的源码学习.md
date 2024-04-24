---
title: 关于Spring加载class为BeanDefinition的源码学习
date: 2023-12-15 11:20
categories: [Spring源码学习]
tags: [Java, Spring, 源码学习]
pin: false
image: 
---

> 这是一篇关于对于Spring是如何把class加载为BeanDefinition的源码学习

## 前言

在上一篇[《关于Spring BeanFactoryPostProcessor注入源码解析》](https://yangushan.github.io/2023/12/14/%E5%85%B3%E4%BA%8ESpring-BeanFactoryPostProcessor%E6%B3%A8%E5%85%A5%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html) 的学习过程中，我们发现了Spring是通过`ConfigurationClassPostProcessor`这个`PostProcessBeanDefinitionRegistry`注入了我们项目中的所有普通BeanDefinition，因此对这个流程产生了好奇，这篇就是用来解析`PostProcessBeanDefinitionRegistry`的实现流程。









--------

**封面图由bing image creator创建*
