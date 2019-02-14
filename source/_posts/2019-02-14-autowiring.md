---
title: 读书笔记：autowiring 的实现
date: 2019-02-14 13:59:01
tags: 
- framework
- spring ioc
comments: true
categories: 
- Java架构
- 源码
---

在自动装配中，不需要对Bean属性做显示的依赖关系声明，只需要配置好autowiring属性，IOC容器会根据这个属性的配置，使用反射自动查找属性的类型或者名字，然后基于属性的类型或名字来自动匹配IOC容器中的Bean,从而自动的完成依赖注入。

<!--more-->

autowiring是在populateBean 填充bean 的方法里面实现的。是populateBean 方法的一部分。
{% codeblock AbstractAutowireCapableBeanFactory.populateBean() lang:java %}
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
        ...
        //读取属性值
        PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
        //看下BeanDefinition中的解析装配模式
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
            //AUTOWIRE_BY_NAME模式
			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
            //AUTOWIRE_BY_TYPE模式
			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}
		...
    }
    protected void autowireByName(
            String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
        
        String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
        //由BeanDefinition中获得propertyName 然后循环getBean
        for (String propertyName : propertyNames) {
            //判断名称是否是FactoryBean的名称，即是否是&开头的名称，如果是，还要验证&开头的名称是否是真的FactoryBean的实现类
            if (containsBean(propertyName)) {
                //向IOC根据beanname获取bean
                Object bean = getBean(propertyName);
                //属性名称得到的bean要缓存到MutablePropertyValues里，后面设置到beanWrapper中,完成自动装配
                pvs.add(propertyName, bean);
                //将依赖的属性名称对应的bean与外部主bean挂钩，添加到由DefaultSingletonBeanRegistry维护的一个ConcurrentHashMap中
                registerDependentBean(propertyName, beanName);
                if (logger.isDebugEnabled()) {
                    logger.debug("Added autowiring by name from bean name '" + beanName +
                            "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
                }
            }
            else {
                if (logger.isTraceEnabled()) {
                    logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                            "' by name: no matching bean found");
                }
            }
        }
    }
{% endcodeblock %}
END