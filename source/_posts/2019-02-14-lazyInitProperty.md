---
title: 读书笔记：lazy-init属性和预实例化
date: 2019-02-14 10:12:01
tags: 
- framework
- spring ioc
comments: true
categories: 
- Java架构
- 源码
---

对于容器的初始化，也有另外一种例外情况，就是用户可以通过设置Bean的lazy-init属性来控制实例的预实例化过程。再容器初始化完成时，完成Bean的依赖注入。

<!--more-->

{% codeblock AbstractApplicationContext.refresh lang:java %}
    public void refresh() throws BeansException, IllegalStateException {
        ...
        // Instantiate all remaining (non-lazy-init) singletons.
        // 初始化所有非延迟加载的单例 ,即初始化所有lazy-init 的单例
        finishBeanFactoryInitialization(beanFactory);
        ...
	}
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	    ...
        // Instantiate all remaining (non-lazy-init) singletons.
        beanFactory.preInstantiateSingletons();
        ...
    }
    @Override
    public void preInstantiateSingletons() throws BeansException {
        if (logger.isDebugEnabled()) {
            logger.debug("Pre-instantiating singletons in " + this);
        }
        // Iterate over a copy to allow for init methods which in turn register new bean definitions.
        // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
        List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
        // Trigger initialization of all non-lazy singleton beans...
        for (String beanName : beanNames) {
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            //lazyinit 懒初始化 默认是false 即随容器启动的时候初始化。 如果设置为true ，则是懒初始化策略，当第一次请求getBean的时候才进行bean初始化。
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                if (isFactoryBean(beanName)) {
                    Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                    //如果是FactoryBean 还要根据FactoryBean 的 isEagerInit 参数，进一步判断是否进行急切初始化（非懒初始化）
                    if (bean instanceof FactoryBean) {
                        final FactoryBean<?> factory = (FactoryBean<?>) bean;
                        boolean isEagerInit;
                        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                            isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                            ((SmartFactoryBean<?>) factory)::isEagerInit,
                                    getAccessControlContext());
                        }
                        else {
                            isEagerInit = (factory instanceof SmartFactoryBean &&
                                    ((SmartFactoryBean<?>) factory).isEagerInit());
                        }
                        if (isEagerInit) {
                            getBean(beanName);
                        }
                    }
                }
                else {
                    getBean(beanName);
                }
            }
        }
        // Trigger post-initialization callback for all applicable beans...
        //如果实现了SmartInitializingSingleton接口，那么再前面初始化bean完成后，执行afterSingletonsInstantiated()初始化后的回调方法
        for (String beanName : beanNames) {
            Object singletonInstance = getSingleton(beanName);
            if (singletonInstance instanceof SmartInitializingSingleton) {
                final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }, getAccessControlContext());
                }
                else {
                    smartSingleton.afterSingletonsInstantiated();
                }
            }
        }
    }
{% endcodeblock %}
以上就是关于lazy-init与初始化后扩展点的分析
END

