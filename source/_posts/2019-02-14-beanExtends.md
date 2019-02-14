---
title: 读书笔记：增加bean 对IOC容器的感知
date: 2019-02-14 10:12:01
tags: 
- framework
- spring ioc
comments: true
categories: 
- Java架构
- 源码
---
容器管理的bean 一般情况下不需要了解容器的状态和直接使用容器。但是有的情况下，bean要直接操作ioc，那么可以通过让bean实现aware 系列接口来实现操作。

<!--more-->

{% codeblock ApplicationContextAwareProcessor lang:java %}
    //比如这个初始化前置处理中，调用了invokeAwareInterfaces方法
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
    //这个方法根据bean实现的具体接口调用方法。往方法中传入ioc容器的引用，从而使得bean能操作ioc容器
	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
{% endcodeblock %}
ApplicationContextAwareProcessor 作为BeanPostProcessor 的一个实现类，对一系列的aware回调进行了调用。
当容器再初始化bean 的同时使得bean能操作ioc容器。
END